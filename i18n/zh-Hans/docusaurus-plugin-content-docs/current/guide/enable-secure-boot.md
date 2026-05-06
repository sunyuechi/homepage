---
id: enable-secure-boot
title: 开启安全启动
description: 本文档介绍如何在 QEMU 平台上为 openRuyi 开启 UEFI Secure Boot。
slug: /guide/enable-secure-boot
---

# 在 openRuyi 上开启安全启动

## 开始前准备

首先，获取 openRuyi UEFI 固件和 openRuyi 系统镜像。

如需获取 UEFI 固件，请下载最新的 `ovmf` 软件包，并手动解压所需文件。可以参考以下命令：

```bash
wget https://boat.openruyi.cn/stable/rva23/riscv64/ovmf-202602-11.3.or.riscv64.rpm
rpm2cpio ovmf-202602-11.3.or.riscv64.rpm | cpio -idmv
cp ./usr/share/ovmf/*.fd ./
```

如果最新版本号与上述命令中的版本不同，请访问 [openRuyi 软件源](https://boat.openruyi.cn/stable/rva23)获取最新版本。

系统镜像请访问我们的新闻页，并参考最新发布文章中的下载链接。本文档以 "openRuyi Server edition (QCOW2)" 镜像为例。

## 创建快速启动脚本

创建名为 `start_or.sh` 的脚本，并赋予可执行权限:

```bash
touch start_or.sh
chmod +x start_or.sh
```

然后写入以下内容:

```bash
#!/usr/bin/env bash

# Start a riscv64 QEMU virtual machine with specific parameters.

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

## 启动虚拟机

执行启动脚本，启动 openRuyi 环境:

```bash
./start_or.sh
```

系统启动后，可以登录以下的两个用户:

 - `root` / `openruyi`
 - `openruyi` / `openruyi`

## 安装所需工具

开启 Secure Boot 前，先安装所需工具:

```bash
dnf install -y efibootmgr efitools grub-unsigned mokutil sbsigntools shim wget
```

安装完成后，将 `grub.cfg` 和 `grubriscv64.efi` 复制到 `/boot/efi/EFI/openruyi` 目录:

```bash
cp /boot/efi/EFI/BOOT/grub.cfg /boot/efi/EFI/openruyi/
cp /lib/grub/riscv64-efi/monolithic/grubriscv64.efi /boot/efi/EFI/openruyi/
```

## 开启 Secure Boot

本节中的所有命令都需要以 `root` 用户执行。如果使用 `openruyi` 用户登录，请在命令前添加 `sudo`。

### 下载 Secure Boot 辅助脚本

下载 `secureboot-setup` 仓库中的辅助脚本，解压归档文件，并进入源码目录:

```bash
wget https://github.com/wxjstz/secureboot-setup/archive/refs/heads/main.tar.gz
tar xvf main.tar.gz
cd secureboot-setup-main
```

这些辅助脚本分别完成以下任务：

* `1.gen_keys`: 创建 PK、KEK、db 和 MOK 密钥。
* `2.import_mok`: 通过 mokutil 发起 MOK enrollment。
* `3.import_uefi_cert`: 将 PK、KEK 和 db 导入 UEFI 变量存储区。

:::warning 注意

完成 Secure Boot 密钥 enrollment 后，请继续使用同一个 `virt_vars.fd` 文件。QEMU 会将 UEFI 变量保存在该文件中；如果替换为新的 `virt_vars.fd`，系统会丢失已导入的密钥。

:::

### 确认 Setup Mode 状态

创建或导入密钥前，先确认固件处于 Setup Mode:

```bash
mokutil --sb-state
```

预期输出如下:

```text
SecureBoot disabled
Platform is in Setup Mode
```

如果没有进入 Setup Mode，请使用干净的 `virt_vars.fd` 重新启动 QEMU 并重试，或者进入 BIOS/UEFI 并清除 Secure Boot 密钥。

### 生成签名密钥

生成 Secure Boot 所需的密钥和证书:

```bash
./1.gen_keys
```

脚本会提示输入证书所有者名称。脚本会将该名称写入生成证书的 Common Name 字段。直接按 Enter 可以使用默认值。

脚本执行完成后，会生成以下文件和目录:

```text
secureboot/
├── cert/       # Private keys, certificates, and MOK.der
├── esl/        # EFI Signature Lists
└── keystore/   # Signed .auth files for UEFI enrollment
```

请备份整个 `secureboot/` 目录，尤其是 `secureboot/cert` 下的私钥文件。

### 签名启动文件

Secure Boot 要求启动链中的关键文件具备有效签名。这里，我们使用 db 密钥签名 shim，再使用 MOK 密钥签名 MokManager、GRUB 和 Linux kernel。

先创建备份目录:

```bash
mkdir -p orig

cp /boot/linux orig/
cp /boot/efi/EFI/openruyi/shimriscv64.efi orig/
cp /boot/efi/EFI/openruyi/mmriscv64.efi orig/
cp /boot/efi/EFI/openruyi/grubriscv64.efi orig/
```

然后使用 db 密钥签名 shim:

```bash
sbsign \
    --key secureboot/cert/db.key \
    --cert secureboot/cert/db.crt \
    --output /boot/efi/EFI/openruyi/shimriscv64.efi \
    orig/shimriscv64.efi
```

再使用 MOK 密钥签名 MokManager、GRUB 和 Linux kernel:

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

### 导入 MOK 证书

导入 UEFI 证书前，先导入 MOK 证书。shim 会通过 MOK 信任已签名的 MokManager、GRUB 和 kernel。

发起 MOK enrollment 请求:

```bash
./2.import_mok
```

请根据提示设置 enrollment password。记住该密码，因为重启后 MokManager 会要求输入该密码。

然后重启虚拟机:

```bash
reboot
```

在 openRuyi 重启后，MokManager 会进入蓝色的 Mok Management 界面。按以下步骤完成 enrollment:

1. 选择 **Enroll MOK** → **Continue**。
2. 输入刚刚设置的密码。
3. 选择 **Yes** 确认 enrollment。
4. 选择 **Reboot**。

第二次重启完成之后，确认 shim 已经导入 MOK 证书:

```bash
mokutil --list-enrolled
```

### 导入 UEFI 证书

将 PK、KEK 和 db 导入 UEFI 变量存储区:

```bash
./3.import_uefi_cert
```

分别检查各个 UEFI key database:

```bash
mokutil --pk
mokutil --kek
mokutil --db
```

在命令输出中确认已导入前面生成的证书。最后再次重启:

```bash
reboot
```

### 验证 Secure Boot 状态

系统启动完成后，检查 Secure Boot 状态:

```bash
mokutil --sb-state
```

预期输出如下:

```text
SecureBoot enabled
```

大功告成！openRuyi 现已开启 Secure Boot。固件通过 db 信任已签名的 shim，shim 则通过 MOK 信任已签名的 MokManager、GRUB 和 Linux kernel。
