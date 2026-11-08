# OpenXzaOS

> 一个从零开始编写的图形化 **x86-64** 操作系统 —— 内核、驱动、图形界面、网络、
> 声音、内置 C 编译器与原生应用全部自主实现，不依赖任何第三方内核。
>
> A from-scratch, graphical **x86-64** operating system — kernel, drivers, GUI,
> networking, sound, a built-in C compiler and native apps, all implemented from
> the ground up. No third-party kernel is used.

---

## OpenXzaOS 是什么？

OpenXzaOS 是一个用于学习 / 爱好的操作系统，可以在真实的 x86-64 机器和虚拟机上启动。
它通过一个极小的 Multiboot 引导垫片（shim）从 32 位模式启动，将 CPU 切换到**长模式（long mode）**，
然后运行一个完全 freestanding（不依赖宿主环境）的 64 位内核。合成式窗口管理器驱动着一个
1024×768 的桌面，包含任务栏、开始菜单和鼠标指针，使用自带的汉字（CJK）字体和抗锯齿 ASCII
字体进行渲染。

核心设计要点：

- **Freestanding 的 x86-64 内核** —— 无 libc、无宿主操作系统。由一个 32 位 Multiboot
  垫片（`openxzaos.elf`）引导进入长模式，并跳转到 64 位内核镜像（`kernel64.bin`）。
- **混合启动介质** —— `build/OpenXzaOS.iso` 可以同时支持 **传统 BIOS（MBR）** 与
  **UEFI** 启动。
- **自研协议栈** —— DHCP、DNS、TCP、HTTP 以及 **HTTPS**（基于 mbedTLS）均在 kernel 内实现，
  因此自带浏览器可以直接加载真实网页。
- **原生应用运行时** —— 内核内置 **C 编译器**，并带有 ELF 与 PE 加载器，使系统能够编译并
  运行原生应用（例如 `apps/calc` 中的计算器）。

---

## 功能特性

- **图形桌面**：合成式窗口管理器、任务栏、开始菜单、鼠标指针、壁纸。
- **中文显示**：HZK16 点阵汉字字体 + 抗锯齿 ASCII 字体，由内核自带的 2D 图形层渲染
  （1024×768、32 位帧缓冲）。
- **内置应用（开始菜单）**：
  - 文件管理器
  - 文本编辑器
  - 浏览器（支持 HTTP / HTTPS）
  - 扫雷 · 乒乓 · 纸牌 · 弹球
  - 终端 · 画图 · 图片查看器
  - 设置 · 显示分辨率 · 关于
  - 注销 · 重启 · 关机
- **网络**：RTL8139 / Intel e1000 / AMD PCnet 网卡驱动，支持 DHCP、DNS、TCP、HTTP、HTTPS。
- **声音**：Sound Blaster 16 与 Intel HD Audio（HDA）输出，附带开机提示音。
- **存储**：ATA（IDE）驱动与磁盘镜像上的 **XzaFS** 文件系统。
- **内核内置 C 编译器** 以及 ELF / PE 应用加载器。
- **磁盘安装器**，可将系统写入硬盘 / 磁盘镜像。

---

## 硬件兼容性

OpenXzaOS 的目标是同时兼容 **Intel** 与 **AMD** 平台。

| 子系统 | 支持的硬件 | 说明 |
|-----------|-------------------|-------|
| **CPU** | Intel & AMD x86-64 | 通过 `CPUID` + `EFER` 进入长模式（与厂商无关）。系统调用使用 `int 0x80` 及标准 x86-64 MSR（`EFER`、`FS_BASE`）—— 不含任何 Intel 专属指令。 |
| **网卡** | 瑞昱 **RTL8139**（QEMU / VirtualBox 默认） · Intel **8254x e1000**（VMware 默认） · AMD **PCnet / Am79C973**（较旧的 VMware / QEMU） | 内核会自动探测三者，并按 RTL8139 → e1000 → PCnet 的顺序回退。 |
| **声卡** | **Sound Blaster 16**（默认） · **Intel HD Audio** | HDA 驱动按 PCI class `0x0403` 探测，因此**一套代码通吃 Intel、AMD 与 NVIDIA 的 HDA 控制器**。优先尝试 HDA，其次 SB16，最后 PC 喇叭。 |
| **启动** | 传统 **BIOS（MBR）** 与 **UEFI** | 同一个 ISO 即可启动两者；内核以原始镜像方式加载，不受 640 KiB 垫片空间限制。 |

---

## 构建

### 前置条件

- 一个 **`x86_64-elf-gcc`** 交叉编译工具链（构建脚本基于 GCC 15.x）。工具链路径在 `build.sh`
  与 `tools/build_iso.sh` 顶部硬编码为 `TC=/d/x86_64-elf-gcc/bin` —— 请**自行修改**为你的安装路径。
- **QEMU**（仅运行 / 测试系统时需要）。
- **Python 3** —— 构建脚本与字体生成脚本依赖它。
- **MinGW-w64** `x86_64-w64-mingw32-gcc` —— *可选*；仅在需要重新构建 UEFI 引导加载器时需要。
  若缺失，构建会回退到预置的引导加载器。
- **mbedTLS 2.28.8** —— 已内置在 `build/dl/mbedtls-2.28.8` 目录。若该目录不存在，请下载
  mbedTLS 2.28.8 并解压到此处（构建使用 `kernel/mbedtls_config.h` 中的精简配置）。

### 构建步骤

```bash
# 1. 构建内核（+ 32 位垫片） -> build/openxzaos.elf
bash build.sh

# 2. 构建 BIOS/UEFI 混合 ISO -> build/OpenXzaOS.iso
bash tools/build_iso.sh
```

在 Windows 上，`tools/build.ps1` 提供了等价的构建步骤。

---

## 运行

**快速开始（直接在 QEMU 下运行内核）：**

```bash
# Linux / Git Bash
./run.sh

# Windows
run.bat
```

**启动 ISO**（在 QEMU、VirtualBox、VMware 或真实硬件上）：

```bash
qemu-system-x86_64 -m 128 -vga std \
  -drive file=build/OpenXzaOS.iso,format=raw,if=ide,media=cdrom \
  -boot order=d -netdev user,id=n0 -device rtl8139,netdev=n0
```

- **登录密码：** `admin123`
- **默认网卡：** RTL8139 · **默认声卡：** Sound Blaster 16
  （若检测到 HDA 控制器，则自动改用 Intel HDA）。

---

## 自动化测试

`tools/qtest.py` 通过一组 UI 指令脚本驱动无头 QEMU 实例，并将调试日志输出到 `build/dbg.log`。

```bash
# 启动 ISO 并运行 UI / 网络脚本
python3 tools/qtest.py tools/t_net_browser.txt --iso

# 切换使用不同的网卡或声卡控制器
QTEST_E1000=1 python3 tools/qtest.py tools/t_net_browser.txt --iso   # Intel e1000
QTEST_PCNET=1 python3 tools/qtest.py tools/t_net_browser.txt --iso   # AMD PCnet
QTEST_HDA=1   python3 tools/qtest.py tools/t_hda.txt --iso           # Intel HD Audio
```

`build/dbg.log` 会记录网卡初始化（`[net]NIC ok`、`[net]OFFER`、`[net]ACK`）以及驱动标记
（`A` = AMD PCnet 已就绪、`H` = HDA 已就绪、`h` = HDA 回退）。

---

## 项目结构

```
kernel/      内核源码：引导垫片、长模式入口、驱动、图形界面、
             网络、声音、文件系统、内核内置 C 编译器、应用。
apps/        用户态 ELF 应用（例如 apps/calc）。
tools/       构建、ISO、字体生成与 QEMU 测试脚本。
build/       构建产物（openxzaos.elf、kernel64.bin、OpenXzaOS.iso …）。
docs/        手册（例如内核内置 C 编译器手册）。
vmware/      示例 VMware 配置。
```

---

## 说明与第三方组件

- 本内核为 **freestanding** 实现，不含任何外部操作系统代码。
- **mbedTLS 2.28.8**（© Arm Limited）内置在 `build/dl` 目录下，为 HTTPS 提供 TLS 层。
  其许可证为 **Apache-2.0 或 GPL-2.0-or-later**，是唯一的链接进内核的第三方组件。
- 图形、字体（HZK16 为国家标准的公有领域汉字字形集）、窗口管理器、协议栈与驱动均为原创实现。

---

## 许可证

本项目**目前尚未提供 LICENSE 文件** —— 发布前请自行添加（爱好型操作系统项目常用 MIT 或
GPLv2）。无论你为 OpenXzaOS 选择何种许可证，内置的 mbedTLS 仍保留其自身的
Apache-2.0 / GPL-2.0-or-later 许可证。

---

<p align="center">OpenXzaOS · 自主实现的 x86-64 图形操作系统</p>
