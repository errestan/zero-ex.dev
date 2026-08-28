---
title: "Ditching GRUB for a Single-Partition systemd-boot + UKI Setup on Fedora"
date: 2026-08-28
draft: true
tags: ["fedora", "systemd-boot", "secure-boot", "uki", "framework", "linux"]
categories: ["linux"]
summary: "How (and why) I moved my Framework Desktop from GRUB2 + a split ESP/boot layout to a single FAT32 /efi partition, Unified Kernel Images, and firmware-enrolled Secure Boot keys — including every wrong turn along the way."
---

I recently rebuilt the boot setup on my Fedora Workstation install (running on a Framework Desktop) from the ground up: GRUB2 out, [systemd-boot](https://www.freedesktop.org/software/systemd/man/latest/systemd-boot.html) in, [Unified Kernel Images](https://uapi-group.org/specifications/specs/unified_kernel_image/) (UKIs) replacing the old separate kernel/initrd pairs, Secure Boot enforced with my own signing key, and — the part that actually caused most of the pain — a single FAT32 `/efi` partition instead of Fedora's default split `/boot` + `/boot/efi` layout.

None of this is strictly necessary. Fedora's default GRUB2 + BLS setup works fine. But I wanted the security properties a UKI gives you (a Secure-Boot-covered initrd, a locked-down kernel command line, no separate unsigned files an attacker could tamper with) without the layout inherited from an era when kernels and initrds were separate files scattered across two partitions. If you're doing the same thing, here's the actual path I took, including the mistakes, so you can skip them.

## Why the partition layout matters more than it looks like it should

Fedora's default install puts `/boot` on its own ext4 partition and the EFI System Partition (ESP) at `/boot/efi`. That's fine for GRUB, which reads both with its own drivers. It becomes a real problem once `kernel-install` and `systemd-boot` are doing the work, because they don't use GRUB's drivers — they rely on the firmware's own ability to read the filesystem, and on strict adherence to the [Boot Loader Specification](https://uapi-group.org/specifications/specs/boot_loader_specification/)'s partition-type detection.

Recent Fedora installers actually set `/boot`'s GPT partition type to the XBOOTLDR GUID (`bc13c2ff-59e6-4262-a352-b275fd6f7172`), specifically for forward-compatibility with this exact migration — even though the partition itself stays ext4. The Boot Loader Specification says that when both an ESP and an XBOOTLDR partition exist, kernel images should go on XBOOTLDR, not the ESP. `kernel-install` follows that rule to the letter, which meant every UKI I built kept landing in `/boot/EFI/Linux/` — on an ext4 filesystem UEFI firmware has no driver for and can never actually read. It wasn't a bug; it was correct behavior for a partition table that no longer matched what I was trying to do with it.

The fix wasn't to fight the detection logic. It was to remove the ambiguity: one partition, FAT32, mounted at `/efi`, nothing else.

## Step 1: Layout and dracut config

First, tell `kernel-install` to build UKIs instead of separate kernel/initrd files:

```bash
# /etc/kernel/install.conf
layout=uki
```

Then tell dracut how to sign them — this is a separate setting from the layout, and it's easy to assume `layout=uki` alone handles signing. It doesn't:

```bash
# /etc/dracut.conf.d/20-uki.conf
uefi_secureboot_cert="/etc/kernel/secure-boot-certificate.pem"
uefi_secureboot_key="/etc/kernel/secure-boot-private-key.pem"
kernel_cmdline="root=UUID=xxxx-xxxx ro rhgb quiet"
```

That `kernel_cmdline` line matters more than it looks. There's a widely repeated assumption that `/etc/kernel/cmdline` supplies the command line baked into a UKI — it doesn't, at least not for Dracut-built images. That file is only consulted by `ukify` and by systemd-boot's own BLS entry generation. Dracut reads its cmdline from `dracut.conf`, full stop. Skip this and you'll get a UKI that boots with no `root=` parameter at all.

Generate a signing key pair if you don't have one:

```bash
openssl req -quiet -newkey rsa:4096 -nodes \
  -keyout /etc/kernel/secure-boot-private-key.pem \
  -new -x509 -sha256 -days 3650 \
  -subj "/CN=My UKI Signing Key/" \
  -out /etc/kernel/secure-boot-certificate.pem
```

And make sure `sbsigntools` is installed — dracut's signing step silently does nothing without `sbsign` present.

## Step 2: Two kernel-install plugins you'll want to mask

Fedora ships kernel-install with a few hook scripts under `/usr/lib/kernel/install.d/`. Two of them actively fight a UKI-only setup:

**`51-dracut-rescue.install`** tries to build a separate rescue kernel/initrd pair — a concept that doesn't map cleanly onto UKI layout and reliably failed with exit status 2 in my testing. Since your fallback in a UKI + systemd-boot setup comes from keeping older UKI generations around (systemd-boot's boot counting), not a dedicated rescue image, it's safe to mask:

```bash
sudo touch /etc/kernel/install.d/51-dracut-rescue.install
```

**`90-uki-copy.install`** is the *other* Fedora mechanism for building UKIs — it expects dracut to hand it a plain initrd, then calls `ukify` separately to assemble and sign the final image. If dracut is already configured to build and sign the complete UKI itself (as above), this plugin gets a finished image instead of the plain initrd it expects, and the result is a UKI missing its initrd section entirely — silently, with a zero exit code from `kernel-install`. Pick one mechanism. I masked this one since dracut's own path was already working:

```bash
sudo touch /etc/kernel/install.d/90-uki-copy.install
```

(An empty, non-executable file with the same name in `/etc/kernel/install.d/` is the standard way to mask a kernel-install plugin shipped in `/usr/lib/`.)

## Step 3: Rebuild the ESP as a single `/efi` partition

This is the part that needs a live USB, since you can't repartition the disk your root filesystem lives on while it's mounted.

From the live environment:

```bash
# format the new/resized ESP partition
sudo mkfs.vfat -F32 -n EFI /dev/<esp-partition>

# mount your system and chroot in
sudo mount /dev/<root-partition> /mnt
sudo mount /dev/<esp-partition> /mnt/efi
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
sudo chroot /mnt
```

Inside the chroot, `/etc/fstab` loses its `/boot` and `/boot/efi` lines entirely and gains one for `/efi`:

```
UUID=xxxx-xxxx  /efi   vfat   umask=0077,shortname=winnt  0  2
```

With no separate `/boot` partition at all, there's nothing left for `kernel-install` to mistake for an XBOOTLDR candidate — it just finds `/efi`, correctly, every time. `/boot` itself becomes an ordinary empty directory inside the root filesystem, which is exactly what you want once a UKI holds everything that used to live there.

## Step 4: Secure Boot — shim, or your own key in firmware?

There are two ways to get a custom-signed UKI trusted under Secure Boot:

1. **Shim + MOK** — the standard Fedora mechanism. Firmware trusts Microsoft's key, which trusts shim, which lets you enroll your own certificate into a separate "Machine Owner Key" list that shim checks for everything it subsequently loads. Removing shim (which I did briefly, by accident, while cleaning out GRUB) breaks this entirely — MOK enrollment lives in a shim-specific NVRAM variable that only shim ever reads, so a bootloader chain that skips shim ignores your enrolled key completely, even if the UKI's signature itself is perfectly valid.
2. **Direct firmware enrollment** — replace the firmware's own Secure Boot key database with your own certificate. No shim required, but it means Windows dual-boot and any other Microsoft-signed binaries stop being trusted unless you re-add Microsoft's certs too, and it requires the firmware to be in **Setup Mode** (Platform Key cleared) to write new keys at all.

I went with the second option, since this machine doesn't dual-boot and it meant one less binary in the chain to keep signed and in sync.

### Getting into Setup Mode on a Framework Desktop

Framework's Insyde BIOS has a specific quirk here: hitting Enter from the "boot failed" prompt only gets you a limited menu without Secure Boot options. You need the **full** setup menu, which means mashing **F2** repeatedly right at power-on. From there: **Security → Secure Boot → Erase all Secure Boot Settings**. That's the option that actually drops the platform into Setup Mode — just disabling Secure Boot enforcement isn't the same thing.

### Installing and enrolling

```bash
sudo dnf install -y libdnf5-plugin-actions sbsigntools
sudo bootctl install --esp-path=/efi --secure-boot-auto-enroll=yes \
  --certificate=/etc/kernel/secure-boot-certificate.pem \
  --private-key=/etc/kernel/secure-boot-private-key.pem
```

Two things worth knowing about this step that weren't obvious to me going in:

- `--secure-boot-auto-enroll=yes` doesn't actually enroll anything by itself on bare metal. It's gated by systemd-boot's `if-safe` policy, which only auto-enrolls unattended inside a hypervisor. On real hardware, it generates the key material and drops it under `/efi/loader/keys/`, but you have to manually select **"Enroll Secure Boot keys: auto"** from the systemd-boot menu on a subsequent boot for it to take effect. If your loader's timeout is 0 (Fedora's default), you'll never see that menu — set a timeout first:
  ```
  # /efi/loader/loader.conf
  timeout 5
  console-mode max
  ```
- `bootctl install` **copies** the systemd-boot binary into place — it does not necessarily sign it with your key, especially if a prior run of the command failed partway through (mine did, on the `Operation not permitted` error from attempting to enroll before Setup Mode was active). A partially-completed run can leave a completely unsigned `systemd-bootx64.efi` sitting in `/efi/EFI/BOOT/BOOTX64.EFI`, which will pass every check right up until you actually flip Secure Boot on in firmware and the machine refuses to boot at all. Always verify explicitly rather than trusting the install output:
  ```bash
  sbverify --cert /etc/kernel/secure-boot-certificate.pem /efi/EFI/BOOT/BOOTX64.EFI
  sbverify --cert /etc/kernel/secure-boot-certificate.pem /efi/EFI/systemd/systemd-bootx64.efi
  ```
  If either comes back `No signature table present`, sign them by hand:
  ```bash
  sudo sbsign --key /etc/kernel/secure-boot-private-key.pem \
              --cert /etc/kernel/secure-boot-certificate.pem \
              --output /efi/EFI/BOOT/BOOTX64.EFI \
              /usr/lib/systemd/boot/efi/systemd-bootx64.efi

  sudo sbsign --key /etc/kernel/secure-boot-private-key.pem \
              --cert /etc/kernel/secure-boot-certificate.pem \
              --output /efi/EFI/systemd/systemd-bootx64.efi \
              /usr/lib/systemd/boot/efi/systemd-bootx64.efi
  ```

## Step 5: Keeping the boot loader signed across updates

The signature on `systemd-bootx64.efi` doesn't survive a package update — the next `systemd-udev` update drops a fresh, unsigned copy straight back into place, and Secure Boot silently breaks again until you notice. Fedora's `dnf5` has a hook mechanism built for exactly this via the `libdnf5-plugin-actions` package:

```bash
sudo tee /usr/local/sbin/sign-systemd-boot.sh << 'EOF'
#!/bin/bash
set -euo pipefail
CERT=/etc/kernel/secure-boot-certificate.pem
KEY=/etc/kernel/secure-boot-private-key.pem
SRC=/usr/lib/systemd/boot/efi/systemd-bootx64.efi

sbsign --key "$KEY" --cert "$CERT" --output /efi/EFI/BOOT/BOOTX64.EFI "$SRC"
sbsign --key "$KEY" --cert "$CERT" --output /efi/EFI/systemd/systemd-bootx64.efi "$SRC"
logger -t sign-systemd-boot "Re-signed systemd-boot binaries after package update"
EOF
sudo chmod +x /usr/local/sbin/sign-systemd-boot.sh

sudo mkdir -p /etc/dnf/libdnf5-plugins/actions.d
sudo tee /etc/dnf/libdnf5-plugins/actions.d/sign-systemd-boot.actions << 'EOF'
post_transaction:systemd-udev*:in::/usr/local/sbin/sign-systemd-boot.sh
EOF
```

Confirm which package actually owns the binary on your system before relying on the filter (`rpm -qf /usr/lib/systemd/boot/efi/systemd-bootx64.efi`) and adjust the glob if it's not `systemd-udev`. Test it fires correctly with a harmless reinstall:

```bash
sudo dnf reinstall -y systemd-udev
sbverify --cert /etc/kernel/secure-boot-certificate.pem /efi/EFI/BOOT/BOOTX64.EFI
```

## Where things ended up

- One FAT32 partition, mounted at `/efi`, holding systemd-boot and every UKI generation Fedora keeps around
- No `/boot` partition, no GRUB, no BLS entry files — UKIs in `EFI/Linux/` are auto-discovered
- Kernel + initrd + cmdline bundled and signed into a single file per kernel version, rebuilt automatically by `kernel-install` on every kernel update
- Secure Boot enforced against a key I control, enrolled directly into firmware
- A dnf hook keeping the boot loader's signature intact across `systemd-udev` updates

If you're doing this on a Framework Desktop specifically, the two Framework-flavored gotchas are worth remembering on their own: F2 (not Enter) for the full setup menu, and "Erase all Secure Boot Settings" (not "Disable Secure Boot") to actually reach Setup Mode. Everything else here should translate to any UEFI machine running Fedora.
