# EDK2 Learning Notes

[EDK2官方教程](https://www.tianocore.org/tianocore-wiki.github.io/)

在经历了三个月的客制化BIOS后，决定推进到下一进程的学习，开始尝试使用EDK2。其实是有些不理解EDK2和UEFI的关系，可以现在好好梳理一番。

第一点最需要纠正的就是：我们常被成为BIOS工程师，但是BIOS已经是过去式了，BIOS指的是**传统 Legacy BIOS**，目前已经不再使用。现在用的是UEFI，所以本质上我们做的是**UEFI固件开发**。但是由于历史习惯还被称为BIOS工程师，就像很多人还把手机充电口叫"USB 口"，其实早就是 Type-C 了。所以阐述一下BIOS到UEFI的演变还是有必要的，详情如下：

## 从 BIOS 演变到 UEFI 的原因

本质原因是：**传统 BIOS 受困于 1980 年代的设计假设（16 位实模式、1MB 寻址、MBR 分区），已经跟不上现代硬件的规模，且缺乏标准化扩展与安全机制**。UEFI 不是简单的"升级版 BIOS"，而是一次**固件架构的重构**。

### 一、传统 BIOS 的核心局限（推动演变的痛点）

| 维度           | BIOS 的限制                                | 带来的实际后果                                          |
|:--------------:|:------------------------------------------:|:------------------------------------------------------- |
| **执行环境**   | 16 位实模式                                | 只有 1MB 寻址空间、实际可用约 640KB，无法直接访问大内存 |
| **分区表**     | MBR（主引导记录）                          | 单盘最大 **2TB**，最多 4 个主分区                       |
| **单次读取**   | 一次只能读 512 字节                        | 加载内核慢，大镜像引导效率低                            |
| **ROM 容量**   | 单芯片约 16MB 上限                         | 塞不下复杂驱动和功能                                    |
| **初始化方式** | 串行、固定顺序                             | 开机自检（POST）慢，无法并行                            |
| **扩展性**     | 无标准驱动模型，各家私有中断（INT 13h 等） | 硬件厂商无法统一扩展                                    |
| **安全**       | 无验证机制                                 | 易被 **Bootkit/Rootkit** 在引导前植入                   |
| **界面/网络**  | 纯文本、无网络栈                           | 难以图形化配置、难以网络引导与远程管理                  |

其中最致命的是**16 位实模式 + MBR**这两条硬约束，它们直接卡住了硬件发展。

### 二、硬件与需求的发展（外部驱动力）

1. **磁盘容量爆炸**：机械盘、SSD 很快突破 2TB，MBR 无法描述 >2TB 的空间。
2. **内存与多核**：服务器动辄上百 GB 内存、几十上百核，16 位环境完全无法驾驭。
3. **安全威胁升级**：操作系统级杀毒管不到固件层，需要**在引导链最前端就做签名验证**。
4. **启动速度诉求**：用户无法忍受几十秒的 POST，需要快速启动。
5. **图形化 / 网络管理**：企业需要远程 KVM、网络引导（PXE）、图形化 Setup。
6. **可扩展生态**：新总线（PCIe、NVMe）、新设备层出不穷，需要一个**标准驱动模型**。

### 三、UEFI 针对性解决了什么

- **执行环境**：运行在 32/64 位保护模式，突破 1MB 限制，可访问全部物理内存。
- **分区方案**：**GPT** 分区表，单盘支持远超 2TB，默认最多 128 个分区。
- **引导方式**：从"读扇区跳转"改为**加载 `.efi` 可执行映像**，一次可加载大块数据。
- **模块化架构**：引入 **Protocol / Handle / Driver** 模型和 **EFI Byte Code (EBC)**，驱动可跨平台复用，可动态加载。
- **并行初始化**：支持驱动并行执行，配合 **Fast Boot** 大幅缩短开机时间。
- **内建能力**：自带网络栈、图形界面、文件系统（FAT）、Shell，可作小型 OS 运行。
- **安全启动 Secure Boot**：逐级验证引导链签名，阻断 Bootkit。
- **标准化**：由 UEFI Forum 统一规范，取代了各家私有 BIOS 实现。

BIOS 用 **16 位实模式 + MBR + 私有中断**这套 1980 年代的老架构，卡住了 2TB 磁盘、大内存、多核、安全启动和图形化管理；UEFI 用 **32/64 位 + GPT + 标准 Protocol 驱动模型 + Secure Boot**重新设计了固件层，既是**硬件规模倒逼**，也是**安全与可扩展性需求驱动**的必然结果。

了解完了演变历程，想要更加深入了解UEFI，必然绕不过以下几个词，最开始我也是云里雾里，雾里悟理。

- **UEFI Spec**：管"固件对**外**（OS loader、Option ROM、驱动）暴露什么接口"——Boot/Runtime Services、HII、协议。
- **PI Spec**：管"固件对**内**（SEC→PEI→DXE→BDS 各阶段之间）怎么交接"——阶段划分、PPI/Protocol 交接、FV/FD。
- **EDK2**：把上面两份规范翻译成能编译的 C 代码 + 构建系统。**它是"实现 UEFI 和 PI 的通用样板房"。**
- **CRB/FSP**：芯片厂在样板房上做的硅专用装修，负责这颗 SoC 的 GPIO、时钟、内存、上电流程。
- **UEFI 固件**：`EDK2 + CRB` 一起编译出来的成品，`UEFI` 是它的**技术身份/名字**，不是它的某一层。

## EDK2的安装与环境配置（基于Windows）

### 一、安装Visual Studio 2019

在最初的客制化教程中就已有此需要安装，就不多额外赘述。需要注意的是如下组件是否被包括：

![](assets\cdcf9d1ced562a5dfa7f50eed1f42636362985e7.png)

### 二、安装Python

虽然网上很多教程说需要下载 **Python 3.10 或 3.11**，但是在我的实际测试中，**3.14**也是可以使用的，所以我单纯觉得更高的版本应该也可以。网上说不能使用的原因应该是某些三方库还没跟上，也许是他们写的教程较早，当时还未同步，现在应该是可以了。

> 记得把Python加入在环境变量内，安装Pyhton的时候可以顺手勾选**“Add Python to PATH”**选项，否则还需要手动配置

记得验证是否安装成功：

```powershell
python --version
# 应该输出 Python 3.10.x 或 3.11.x

pip --version
# 确认 pip 可用
```

### 三、安装NASM

NASM（Netwide Assembler）是 EDK2 中某些底层模块（比如 SEC 阶段的汇编代码）需要的汇编器。前往 [NASM 官网](https://www.nasm.us/) 下载最新的 Windows 安装包。

1. 安装到 `C:\nasm`（路径简单，后面配置方便）
2. 把 `C:\nasm` 添加到系统 **PATH** 环境变量

记得验证是否安装成功：

```powershell
nasm -v
# 应该输出类似 NASM version 2.16.xx
```

### 四、安装ASL编译器（iasl）

ACPI 相关的模块需要 ASL 编译器。EDK2 构建过程中会自动调用 `iasl`。前往 [ACPICA 下载页面](https://acpica.org/downloads) 下载 **Windows Binary Tools**。

1. 解压到 `C:\ASL`
2. 把 `C:\ASL` 添加到系统 PATH

记得验证是否安装成功：

```powershell
iasl -v
# 应该输出 Intel ACPI Component Architecture ASL+ Optimizing Compiler/Disassembler version xxxxxxxx
```

### 五、安装Git

如果你还没装 Git：前往 [Git 官网](https://git-scm.com/) 下载 Windows 安装包，一路 Next 即可。

找一个没有中文和空格的目录（比如 `F:\EDK2`），打开 Git Bash 或 PowerShell：

```git
# 克隆 EDK2 主仓库
git clone https://github.com/tianocore/edk2.git
# 克隆 EDK2 主仓库(网络不好时用ssh可能会快一些，比较玄学)
git clone git@github.com:tianocore/edk2.git

# 进入目录
cd edk2

# 初始化子模块（很重要！EDK2 有一些依赖是 git submodule）
git submodule update --init --recursive
```

- `git submodule update --init --recursive` 这一步**不能跳过**！否则后面编译会报一堆找不到文件的错
- 如果 GitHub 下载很慢，可以用 `https://ghproxy.com/` 等镜像加速，或者配置 Git 代理
- 完整克隆（含子模块）大约需要 2~3GB 磁盘空间

### 六、配置 EDK2 构建环境

这是最关键的一步。打开 VS 命令行环境，不要直接用普通的 cmd，要用 Visual Studio 提供的开发者命令行工具。在开始菜单搜索：**“x64 Native Tools Command Prompt for VS 2019”**，打开它。

> 这个命令行自动配置了 MSVC 编译器的环境变量，直接在普通 cmd 里编译会找不到 `cl.exe`。

**运行 edksetup.bat：**

```powershell
cd /d F:\EDK2\edk2

edksetup.bat
```

这个脚本会：

1. 设置 EDK2 相关的环境变量（`WORKSPACE`、`EDK_TOOLS_PATH` 等）
2. 编译 BaseTools 的 C 工具（如果是第一次运行）

**编译 BaseTools：**

首次使用需要编译 EDK2 的构建工具：

```powershell
edksetup.bat Rebuild
```

如果一切顺利，你会看到类似这样的输出：

```powershell
Building ... C Tools ...
Finished building BaseTools C Tools.
```

> ⚠️ 如果报错 `nmake not found`，请尝试用AI分析问题。

**配置 target.txt：**

编辑 `F:\UEFI\edk2\Conf\target.txt`（第一次运行 edksetup.bat 后会自动生成）：

```text
ACTIVE_PLATFORM       = OvmfPkg/OvmfPkgX64.dsc
TARGET                = DEBUG
TARGET_ARCH           = X64
TOOL_CHAIN_TAG        = VS2019
```

> 💡 **关键字段说明**：  

- `ACTIVE_PLATFORM`：指定编译哪个平台，OVMF 是 [QEMU](https://zhida.zhihu.com/search?content_id=273955041&content_type=Article&match_order=1&q=QEMU&zhida_source=entity) 虚拟机的固件
- `TARGET`：DEBUG 版本会输出调试日志，RELEASE 版本会做优化
- `TARGET_ARCH`：X64 表示编译 64 位固件
- `TOOL_CHAIN_TAG`：编译器版本，和你安装的 VS 对应

### 七、编译

万事俱备，一行命令开始编译：

```powershell
build
```

第一次编译 OVMF 大约需要 5~15 分钟（取决于你的 CPU 和硬盘速度）。

如果看到类似这样的输出，恭喜你，编译成功了：

```text
- Done -
Build end time: ...
Build total time: 00:08:23

Build successfully!
```

编译产物在这里：

```text
F:\UEFI\edk2\Build\OvmfX64\DEBUG_VS2022\FV\OVMF.fd
```

这个 `OVMF.fd` 就是你编译出来的 UEFI 固件镜像！虽然它是虚拟机用的，但它是一个货真价实的 UEFI 固件。

### 八、常见问题

| 错误信息                     | 原因                      | 解决方法                                |
|:----------------------------:|:-------------------------:| --------------------------------------- |
| 'nmake' is not recognized    | 没有用 VS 开发者命令行    | 用 “x64 Native Tools Command Prompt”    |
| 'python' is not recognized   | Python 没加入 PATH        | 重装 Python，勾选 “Add to PATH”         |
| 'nasm' is not recognized     | NASM 没安装或没加 PATH    | 安装 NASM 并加到 PATH                   |
| No module named 'xxx'        | Python 缺少依赖包         | pip install xxx                         |
| submodule xxx not found      | 没有初始化子模块          | git submodule update --init --recursive |
| 路径中有中文导致乱码         | EDK2 工具链不支持中文路径 | 把 edk2 放到纯英文路径下                |
| Build Failed + IA32 相关错误 | target.txt 架构配置不对   | 确认 TARGET_ARCH = X64yy                |
