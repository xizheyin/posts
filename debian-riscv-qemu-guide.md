# 在WSL2（Debian）中运行RISC-V QEMU并执行程序指南

## 目录
1. [环境准备](#环境准备)
2. [安装QEMU](#安装qemu)
3. [下载RISC-V系统镜像](#下载risc-v系统镜像)
4. [运行RISC-V虚拟机](#运行risc-v虚拟机)
5. [在虚拟机中编译和运行程序](#在虚拟机中编译和运行程序)
6. [常见问题解决](#常见问题解决)

## 环境准备

### 检查WSL2环境
首先确认您的WSL2 Debian环境是否正常：

```bash
# 检查WSL版本
wsl --version

# 检查当前运行的发行版
wsl --list --verbose

# 更新包管理器
sudo apt update && sudo apt upgrade -y
```

### 安装必要的依赖
```bash
# 安装编译工具和依赖
sudo apt install -y build-essential git wget curl

# 安装QEMU相关依赖
sudo apt install -y libglib2.0-dev libfdt-dev libpixman-1-dev zlib1g-dev
sudo apt install -y libnfs-dev libiscsi-dev libssh-dev libvde-dev
sudo apt install -y libvdeplug-dev libvte-dev libxen-dev libxenstore3.0
```

## 安装QEMU

### 方法1：通过包管理器安装（推荐）
```bash
# 安装QEMU
sudo apt install -y qemu-system-misc qemu-utils

# 验证安装
qemu-system-riscv64 --version
```

### 方法2：从源码编译安装
如果需要最新版本或特定功能，可以从源码编译：

```bash
# 下载QEMU源码
cd ~
wget https://download.qemu.org/qemu-8.2.0.tar.xz
tar xf qemu-8.2.0.tar.xz
cd qemu-8.2.0

# 配置和编译
./configure --target-list=riscv64-softmmu,riscv64-linux-user
make -j$(nproc)
sudo make install

# 验证安装
qemu-system-riscv64 --version
```

## 下载RISC-V系统镜像

### 下载Debian RISC-V镜像
```bash
# 创建目录
mkdir -p ~/riscv-vm
cd ~/riscv-vm

# 下载Debian RISC-V镜像（约1.5GB）
wget https://cdimage.debian.org/cdimage/ports/12.0-0/riscv64/iso-cd/debian-12.0.0-riscv64-netinst.iso

# 验证镜像
ls -lh debian-12.0.0-riscv64-netinst.iso
```

### 创建虚拟磁盘
```bash
# 创建虚拟磁盘镜像（10GB）
qemu-img create -f qcow2 debian-riscv.qcow2 10G

# 验证磁盘镜像
ls -lh debian-riscv.qcow2
```

### 创建启动脚本
创建一个便捷的启动脚本：

```bash
cat > start-debian-riscv.sh << 'EOF'
#!/bin/bash

# Debian RISC-V虚拟机启动脚本
VM_IMAGE="debian-riscv.qcow2"
ISO_IMAGE="debian-12.0.0-riscv64-netinst.iso"
VM_MEMORY="2G"
VM_CORES="2"

# 检查镜像文件是否存在
if [ ! -f "$VM_IMAGE" ]; then
    echo "错误：找不到磁盘镜像文件 $VM_IMAGE"
    echo "请先创建虚拟磁盘镜像"
    exit 1
fi

if [ ! -f "$ISO_IMAGE" ]; then
    echo "错误：找不到ISO镜像文件 $ISO_IMAGE"
    echo "请先下载Debian RISC-V ISO镜像"
    exit 1
fi

echo "启动Debian RISC-V虚拟机..."
echo "磁盘镜像: $VM_IMAGE"
echo "ISO镜像: $ISO_IMAGE"
echo "内存: $VM_MEMORY"
echo "CPU核心: $VM_CORES"
echo ""

# 启动虚拟机（首次安装）
qemu-system-riscv64 \
    -machine virt \
    -cpu rv64 \
    -m $VM_MEMORY \
    -smp $VM_CORES \
    -device virtio-blk-device,drive=hd0 \
    -drive file=$VM_IMAGE,if=none,id=hd0 \
    -device virtio-blk-device,drive=cd0 \
    -drive file=$ISO_IMAGE,if=none,id=cd0,media=cdrom \
    -device virtio-net-device,netdev=net0 \
    -netdev user,id=net0,hostfwd=tcp::2222-:22 \
    -device virtio-rng-device \
    -device virtio-balloon-device \
    -nographic \
    -serial mon:stdio \
    -bios /usr/share/qemu-efi-aarch64/QEMU_EFI.fd
EOF

# 给脚本添加执行权限
chmod +x start-debian-riscv.sh
```

### 创建运行脚本（安装完成后使用）
```bash
cat > run-debian-riscv.sh << 'EOF'
#!/bin/bash

# Debian RISC-V虚拟机运行脚本（安装完成后使用）
VM_IMAGE="debian-riscv.qcow2"
VM_MEMORY="2G"
VM_CORES="2"

# 检查镜像文件是否存在
if [ ! -f "$VM_IMAGE" ]; then
    echo "错误：找不到磁盘镜像文件 $VM_IMAGE"
    echo "请先创建并安装系统"
    exit 1
fi

echo "启动Debian RISC-V虚拟机..."
echo "磁盘镜像: $VM_IMAGE"
echo "内存: $VM_MEMORY"
echo "CPU核心: $VM_CORES"
echo ""

# 启动虚拟机（正常启动）
qemu-system-riscv64 \
    -machine virt \
    -cpu rv64 \
    -m $VM_MEMORY \
    -smp $VM_CORES \
    -device virtio-blk-device,drive=hd0 \
    -drive file=$VM_IMAGE,if=none,id=hd0 \
    -device virtio-net-device,netdev=net0 \
    -netdev user,id=net0,hostfwd=tcp::2222-:22 \
    -device virtio-rng-device \
    -device virtio-balloon-device \
    -nographic \
    -serial mon:stdio \
    -bios /usr/share/qemu-efi-aarch64/QEMU_EFI.fd \
    -append "root=/dev/vda1 rw console=ttyS0"
EOF

# 给脚本添加执行权限
chmod +x run-debian-riscv.sh
```

## 运行RISC-V虚拟机

### 首次安装Debian
```bash
# 进入虚拟机目录
cd ~/riscv-vm

# 启动虚拟机进行安装
./start-debian-riscv.sh
```

### 安装过程指导
1. **启动安装程序**：等待Debian安装程序启动
2. **选择语言**：选择English
3. **选择国家**：选择您的国家
4. **键盘布局**：选择适合的键盘布局
5. **网络配置**：选择自动配置DHCP
6. **主机名**：输入主机名（如：debian-riscv）
7. **域名**：可以留空
8. **root密码**：设置root密码
9. **创建用户**：创建普通用户账户
10. **分区**：选择"Guided - use entire disk"
11. **软件选择**：选择"Debian desktop environment"和"standard system utilities"
12. **安装GRUB**：选择安装到/dev/vda
13. **完成安装**：重启虚拟机

### 正常启动虚拟机
安装完成后，使用运行脚本启动：

```bash
# 启动已安装的虚拟机
./run-debian-riscv.sh
```

### 网络配置
虚拟机启动后，网络应该自动配置。您可以通过SSH连接到虚拟机：

```bash
# 在WSL2主机上连接虚拟机
ssh -p 2222 username@localhost
```

## 在虚拟机中编译和运行程序

### 安装开发工具
在Debian RISC-V虚拟机中安装必要的开发工具：

```bash
# 更新包管理器
sudo apt update

# 安装编译工具
sudo apt install -y build-essential gcc g++ make cmake

# 安装调试工具
sudo apt install -y gdb valgrind

# 安装常用库
sudo apt install -y libc6-dev libstdc++6-dev

# 安装其他有用的工具
sudo apt install -y vim nano git curl wget
```

### 编译和运行C程序
创建一个简单的C程序进行测试：

```bash
# 创建测试目录
mkdir ~/test-programs
cd ~/test-programs

# 创建Hello World程序
cat > hello.c << 'EOF'
#include <stdio.h>

int main() {
    printf("Hello from Debian RISC-V!\n");
    printf("Architecture: RISC-V 64-bit\n");
    printf("Distribution: Debian\n");
    return 0;
}
EOF

# 编译程序
gcc -o hello hello.c

# 运行程序
./hello
```

### 编译和运行C++程序
```bash
# 创建C++程序
cat > hello.cpp << 'EOF'
#include <iostream>

int main() {
    std::cout << "Hello from Debian RISC-V C++!" << std::endl;
    std::cout << "C++ version: " << __cplusplus << std::endl;
    std::cout << "Distribution: Debian" << std::endl;
    return 0;
}
EOF

# 编译C++程序
g++ -o hello_cpp hello.cpp

# 运行程序
./hello_cpp
```

### 编译和运行Python程序
```bash
# 安装Python
sudo apt install -y python3 python3-pip

# 创建Python程序
cat > hello.py << 'EOF'
#!/usr/bin/env python3
import platform
import os

print("Hello from Debian RISC-V Python!")
print(f"Python version: {platform.python_version()}")
print(f"Architecture: {platform.machine()}")
print(f"Distribution: {platform.linux_distribution()}")
print(f"Kernel: {platform.release()}")
EOF

# 运行Python程序
python3 hello.py
```

### 交叉编译测试
在WSL2主机上交叉编译RISC-V程序：

```bash
# 安装RISC-V交叉编译工具链
sudo apt install -y gcc-riscv64-linux-gnu g++-riscv64-linux-gnu

# 创建测试程序
cat > cross_compile_test.c << 'EOF'
#include <stdio.h>

int main() {
    printf("Cross-compiled for RISC-V on Debian!\n");
    return 0;
}
EOF

# 交叉编译
riscv64-linux-gnu-gcc -o cross_test cross_compile_test.c

# 将编译好的程序传输到虚拟机
scp -P 2222 cross_test username@localhost:~/

# 在虚拟机中运行
ssh -p 2222 username@localhost "chmod +x ~/cross_test && ~/cross_test"
```

## 性能优化和配置

### 优化虚拟机性能
修改启动脚本以提升性能：

```bash
# 编辑启动脚本，添加性能优化选项
cat > run-debian-riscv-optimized.sh << 'EOF'
#!/bin/bash

VM_IMAGE="debian-riscv.qcow2"
VM_MEMORY="4G"
VM_CORES="4"

qemu-system-riscv64 \
    -machine virt,accel=kvm \
    -cpu rv64 \
    -m $VM_MEMORY \
    -smp $VM_CORES \
    -device virtio-blk-device,drive=hd0 \
    -drive file=$VM_IMAGE,if=none,id=hd0,cache=writeback \
    -device virtio-net-device,netdev=net0 \
    -netdev user,id=net0,hostfwd=tcp::2222-:22 \
    -device virtio-rng-device \
    -device virtio-balloon-device \
    -nographic \
    -serial mon:stdio \
    -bios /usr/share/qemu-efi-aarch64/QEMU_EFI.fd \
    -append "root=/dev/vda1 rw console=ttyS0"
EOF

chmod +x run-debian-riscv-optimized.sh
```

### 配置快照和备份
```bash
# 创建虚拟机快照
qemu-img snapshot -c snapshot1 debian-riscv.qcow2

# 列出快照
qemu-img snapshot -l debian-riscv.qcow2

# 恢复到快照
qemu-img snapshot -a snapshot1 debian-riscv.qcow2

# 删除快照
qemu-img snapshot -d snapshot1 debian-riscv.qcow2
```

## 常见问题解决

### 1. QEMU启动失败
```bash
# 检查QEMU是否正确安装
which qemu-system-riscv64

# 检查依赖库
ldd $(which qemu-system-riscv64)

# 重新安装QEMU
sudo apt remove --purge qemu-system-misc
sudo apt install qemu-system-misc
```

### 2. 虚拟机无法启动
```bash
# 检查镜像文件完整性
file debian-12.0.0-riscv64-netinst.iso
file debian-riscv.qcow2

# 检查磁盘镜像信息
qemu-img info debian-riscv.qcow2
```

### 3. 网络连接问题
```bash
# 检查端口转发
netstat -tlnp | grep 2222

# 测试SSH连接
ssh -v -p 2222 username@localhost

# 在虚拟机内检查网络
ip addr show
ping -c 3 8.8.8.8
```

### 4. 性能问题
```bash
# 检查KVM支持
ls /dev/kvm

# 检查CPU虚拟化支持
egrep -c '(vmx|svm)' /proc/cpuinfo

# 检查虚拟机内CPU信息
ssh -p 2222 username@localhost "lscpu"
```

### 5. 磁盘空间不足
```bash
# 扩展磁盘镜像
qemu-img resize debian-riscv.qcow2 +5G

# 在虚拟机内扩展分区
ssh -p 2222 username@localhost "sudo fdisk /dev/vda"
ssh -p 2222 username@localhost "sudo resize2fs /dev/vda1"
```

## 高级用法

### 使用图形界面
如果需要图形界面，可以修改启动参数：

```bash
qemu-system-riscv64 \
    -machine virt \
    -cpu rv64 \
    -m 2G \
    -smp 2 \
    -device virtio-blk-device,drive=hd0 \
    -drive file=debian-riscv.qcow2,if=none,id=hd0 \
    -device virtio-net-device,netdev=net0 \
    -netdev user,id=net0,hostfwd=tcp::2222-:22 \
    -device virtio-gpu-device \
    -device virtio-tablet-device \
    -device virtio-keyboard-device \
    -bios /usr/share/qemu-efi-aarch64/QEMU_EFI.fd \
    -append "root=/dev/vda1 rw"
```

### 调试程序
在虚拟机中调试程序：

```bash
# 编译带调试信息的程序
gcc -g -o debug_test debug_test.c

# 使用GDB调试
gdb debug_test

# 使用Valgrind检查内存
valgrind --leak-check=full ./debug_test
```

### 安装桌面环境
```bash
# 安装XFCE桌面环境
sudo apt install -y task-xfce-desktop

# 安装GNOME桌面环境
sudo apt install -y task-gnome-desktop

# 安装KDE桌面环境
sudo apt install -y task-kde-desktop
```

## 总结

通过本指南，您已经成功在WSL2（Debian）环境中：

1. ✅ 安装了QEMU模拟器
2. ✅ 下载了Debian RISC-V镜像
3. ✅ 配置并安装了Debian RISC-V虚拟机
4. ✅ 在虚拟机中编译和运行了程序
5. ✅ 了解了性能优化和故障排除方法

这个环境非常适合：
- RISC-V架构学习和开发
- Debian系统学习和使用
- 交叉编译测试
- 嵌入式系统开发
- 操作系统学习

建议定期备份重要的程序和配置文件，并保持系统和工具的更新。 