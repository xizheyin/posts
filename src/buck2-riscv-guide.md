# 在RISC-V虚拟机中安装和运行Buck2构建系统

## 概述

Buck2是Meta开发的新一代构建系统，专为大型代码库设计，具有出色的性能和可扩展性。本文将详细介绍如何在RISC-V架构的Debian虚拟机中安装Buck2，并运行第一个示例项目。

## 环境准备

### 系统要求
- RISC-V 64位架构
- Debian 12或更高版本
- 至少2GB可用内存
- 至少5GB可用磁盘空间

### 检查系统环境

检查系统架构
```bash
root@debian:~/.buck2/bin# uname -m
riscv64
```

检查系统版本
```bash
root@debian:~/.buck2/bin# cat /etc/os-release 
PRETTY_NAME="Debian GNU/Linux 13 (trixie)"
NAME="Debian GNU/Linux"
VERSION_ID="13"
VERSION="13 (trixie)"
VERSION_CODENAME=trixie
DEBIAN_VERSION_FULL=13.0
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
```

检查可用内存
```bash
free -h
```


检查磁盘空间
```
df -h
```

### 安装必要的编译工具

在安装Buck2之前，需要确保系统中有必要的编译工具：

```bash
# 更新包管理器
apt update

# 安装基础构建工具
apt install -y build-essential

# 安装clang编译器（Buck2推荐使用）
apt install -y clang lld


# 安装RISC-V工具链（可选，用于交叉编译）
apt install -y gcc-riscv64-linux-gnu g++-riscv64-linux-gnu

# 安装Rust工具链（如果使用Rust项目）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env
rustup target add riscv64gc-unknown-linux-gnu

# 验证工具安装
gcc --version
clang --version
rustc --version
```

## 安装Buck2

### 方法1：从GitHub Releases下载安装（推荐）

#### 1. 下载Buck2 RISC-V版本
```bash
# 创建安装目录
mkdir -p ~/.buck2/bin
cd ~/.buck2/bin

# 下载最新版本的Buck2 RISC-V版本
# 注意：URL中的日期需要根据实际发布版本调整
wget https://github.com/facebook/buck2/releases/download/latest/buck2-riscv64gc-unknown-linux-gnu.zst
```

#### 2. 解压和安装
```bash
# 解压文件
zstd -d buck2-riscv64gc-unknown-linux-gnu.zst

# 重命名为buck2并添加执行权限
mv buck2-riscv64gc-unknown-linux-gnu buck2
chmod +x buck2

# 验证文件
file buck2
ls -la buck2
```


#### 4. 配置环境变量
```bash
# 将Buck2添加到PATH
echo 'export PATH="$HOME/.buck2/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# 验证安装
buck2 --version
```


### 方法2：从源码编译安装

```bash
# 安装Rust工具链（如果尚未安装）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env

# 克隆Buck2源码
git clone https://github.com/facebook/buck2.git
cd buck2

# 编译Buck2
cargo install --path app/buck2 --release

# 验证安装
buck2 --version
```

### 验证安装

安装完成后，运行以下命令验证：

```bash
# 检查Buck2版本
buck2 --version

# 检查Buck2帮助信息
buck2 --help

# 检查Buck2是否在PATH中
which buck2

# 验证prelude安装
ls -la ~/.buck2/prelude/

# 测试Buck2基本功能
buck2 init --help
```

## 创建第一个Buck2项目

官方给出了一个例子，用于测试你是否正确安装了Buck2：<https://buck2.build/docs/getting_started/tutorial_first_build/>
