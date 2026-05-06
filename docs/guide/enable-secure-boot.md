---
id: enable-secure-boot
title: Enable Secure Boot
description: This guide explains how to enable UEFI Secure Boot for openRuyi on the QEMU platform.
slug: /guide/enable-secure-boot
---

# Enable Secure Boot on openRuyi

This guide explains how to enable UEFI Secure Boot for openRuyi on the QEMU platform.

## Before You Begin

You will first need to obtain the openRuyi UEFI firmware and the openRuyi system image.

To obtain the UEFI firmware, download our latest `ovmf` package and manually extract the necessary files. You can do this by following the example command below:

```bash
wget https://boat.openruyi.cn/stable/rva23/riscv64/ovmf-202602-11.3.or.riscv64.rpm
rpm2cpio ovmf-202602-11.3.or.riscv64.rpm | cpio -idmv
cp ./usr/share/ovmf/*.fd ./
```

If the latest version is different from the one shown in the commands, please check [our repository](https://boat.openruyi.cn/stable/rva23/riscv64) for the latest version.

For the system image, please visit our news site and refer to the latest release article for the download link. Here, we will use the "openRuyi Server edition (Qcow2)" image as an example.

## Create Quick Start Script

First, create a script named `start_or.sh` and make it executable:

```bash
touch start_or.sh
chmod +x start_or.sh
```

Then insert the code below:

```bash
#!/usr/bin/env bash

# The script is created for starting a riscv64 qemu virtual machine with specific parameters.

RESTORE=$(echo -en '\001\033[0m\002')
YELLOW=$(echo -en '\001\033[00;33m\002')

## Configuration
vcpu=2
memory=8
drive="openRuyi-2026.04-Server-cloud.qcow2"
fw1="virt_code.fd"
fw2="virt_vars.fd"
ssh_port=12055

cmd="qemu-system-riscv64 \
  -nographic -machine virt,pflash0=pflash0,pflash1=pflash1,acpi=off \
  -cpu rva23s64,sv39=on \
  -smp "$vcpu" -m "$memory"G \
  -blockdev node-name=pflash0,driver=file,read-only=on,filename="$fw1" \
  -blockdev node-name=pflash1,driver=file,filename="$fw2" \
  -drive file="$drive",format=qcow2,id=hd0,if=none \
  -object rng-random,filename=/dev/urandom,id=rng0 \
  -device virtio-vga \
  -device virtio-rng-device,rng=rng0 \
  -device virtio-blk-device,drive=hd0 \
  -device virtio-net-device,netdev=usernet \
  -netdev user,id=usernet,hostfwd=tcp::"$ssh_port"-:22 \
  -device qemu-xhci -usb -device usb-kbd -device usb-tablet"

echo ${YELLOW}:: Starting VM...${RESTORE}
echo ${YELLOW}:: Using following configuration${RESTORE}
echo ""
echo ${YELLOW}vCPU Cores: "$vcpu"${RESTORE}
echo ${YELLOW}Memory: "$memory"G${RESTORE}
echo ${YELLOW}Disk: "$drive"${RESTORE}
echo ${YELLOW}SSH Port: "$ssh_port"${RESTORE}
echo ""
echo ${YELLOW}:: NOTE: Make sure ONLY ONE .qcow2 file is${RESTORE}
echo ${YELLOW}in the current directory${RESTORE}
echo ""
echo ${YELLOW}:: Tip: Try setting DNS manually if QEMU user network doesn\'t work well. ${RESTORE}
echo ${YELLOW}:: HOWTO -\> https://serverfault.com/a/810639 ${RESTORE}
echo ""
echo ${YELLOW}:: Tip: If \'ping\' reports permission error, try reinstalling \'iputils\'. ${RESTORE}
echo ${YELLOW}:: HOWTO -\> \'sudo dnf reinstall iputils\' ${RESTORE}
echo ""

sleep 2

eval $cmd
```

## Start the Virtual Machine

Start your openRuyi environment by executing the script:

```bash
./start_or.sh
```

After the system has booted, you can log in using one of the following accounts:

 - `root` / `openruyi`
 - `openruyi` / `openruyi`

## Install the Required Tools

Before we enable Secure Boot on openRuyi, we need to install the necessary tools:

```bash
dnf install -y efibootmgr efitools grub-unsigned mokutil sbsigntools shim wget
```

After installation, copy both the `grub.cfg` and the `grubriscv64.efi` to the `/boot/efi/EFI/openruyi` folder:

```bash
cp /boot/efi/EFI/BOOT/grub.cfg /boot/efi/EFI/openruyi/
cp /lib/grub/riscv64-efi/monolithic/grubriscv64.efi /boot/efi/EFI/openruyi/
```

## Enable Secure Boot

Run all commands in this section as `root`. If you log in as the `openruyi` user, prefix each command with `sudo`.

### Download Secure Boot Helper Scripts

Download the helper scripts from the `secureboot-setup` repository, extract the archive, and enter the source directory:

```bash
wget https://github.com/wxjstz/secureboot-setup/archive/refs/heads/main.tar.gz
tar xvf main.tar.gz
cd secureboot-setup-main
```

The helper scripts perform three tasks:

* `1.gen_keys` creates PK, KEK, db, and MOK keys.
* `2.import_mok` starts MOK enrollment through `mokutil`.
* `3.import_uefi_cert` imports PK, KEK, and db into the UEFI variable store.

:::warning Note

Keep the same `virt_vars.fd` file after you enroll Secure Boot keys. QEMU stores UEFI variables in that file, so a fresh copy loses the enrolled keys.

:::

### Verify Setup Mode

Before you create or enroll keys, confirm that the firmware runs in Setup Mode:

```bash
mokutil --sb-state
```

You should see the following output:

```text
SecureBoot disabled
Platform is in Setup Mode
```

If it fails to enter Setup Mode, either start QEMU with a clean `virt_vars.fd` and try again, or enter the BIOS/UEFI and clear the Secure Boot keys.

### Generate Signing Keys

Generate the Secure Boot keys and certificates:

```bash
./1.gen_keys
```

Enter a certificate owner name at the prompt. The script uses the name as the Common Name field in the generated certificates. Press Enter to use the default value.

After the script finishes, it creates the following files and directories:

```text
secureboot/
├── cert/       # Private keys, certificates, and MOK.der
├── esl/        # EFI Signature Lists
└── keystore/   # Signed .auth files for UEFI enrollment
```

Back up the entire `secureboot/` directory, especially the private key files under `secureboot/cert`.

### Sign the Boot Files

Secure Boot requires signatures on the files that participate in the boot chain. For this setup, sign shim with the db key, then sign MokManager, GRUB, and the Linux kernel with the MOK key.

Create a backup directory first:

```bash
mkdir -p orig

cp /boot/linux orig/
cp /boot/efi/EFI/openruyi/shimriscv64.efi orig/
cp /boot/efi/EFI/openruyi/mmriscv64.efi orig/
cp /boot/efi/EFI/openruyi/grubriscv64.efi orig/
```

Sign shim with the db key:

```bash
sbsign \
    --key secureboot/cert/db.key \
    --cert secureboot/cert/db.crt \
    --output /boot/efi/EFI/openruyi/shimriscv64.efi \
    orig/shimriscv64.efi
```

Sign MokManager, GRUB, and the Linux kernel with the MOK key:

```bash
sbsign \
    --key secureboot/cert/MOK.key \
    --cert secureboot/cert/MOK.crt \
    --output /boot/efi/EFI/openruyi/mmriscv64.efi \
    orig/mmriscv64.efi

sbsign \
    --key secureboot/cert/MOK.key \
    --cert secureboot/cert/MOK.crt \
    --output /boot/efi/EFI/openruyi/grubriscv64.efi \
    orig/grubriscv64.efi

sbsign \
    --key secureboot/cert/MOK.key \
    --cert secureboot/cert/MOK.crt \
    --output /boot/linux \
    orig/linux
```

### Enroll the MOK Certificate

Enroll the MOK certificate before you import the UEFI certificates. Shim uses MOK to trust the signed MokManager, GRUB, and kernel.

Start the MOK enrollment request:

```bash
./2.import_mok
```

Set an enrollment password when prompted. Remember this password because MokManager will ask for it after the reboot.

Then, reboot the virtual machine:

```bash
reboot
```

After openRuyi reboots, MokManager opens the blue MOK Management screen. Complete the enrollment:

1. Select **Enroll MOK** → **Continue**.
2. Enter the password you created earlier.
3. Select **Yes** to confirm enrollment.
4. Select **Reboot**.

After the second reboot, confirm that shim enrolled the MOK certificate:

```bash
mokutil --list-enrolled
```

### Import the UEFI Certificates

Import PK, KEK, and db into the UEFI variable store:

```bash
./3.import_uefi_cert
```

Check each UEFI key database:

```bash
mokutil --pk
mokutil --kek
mokutil --db
```

Look for your generated certificates in the command output. Reboot again:

```bash
reboot
```

### Verify Secure Boot

After the system boots, check the Secure Boot state:

```bash
mokutil --sb-state
```

You should see:

```text
SecureBoot enabled
```

Done! openRuyi now boots with Secure Boot enabled. The firmware trusts the signed shim via db, and the shim trusts the signed MokManager, GRUB, and Linux kernel via MOK.
