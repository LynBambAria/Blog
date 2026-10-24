# BIOS常见概念和知识点

## ACPI

### ACPI 是什么

**ACPI = Advanced Configuration and Power Interface**（高级配置与电源接口）

一套由 **Intel / Microsoft / Toshiba** 于 1996 年牵头制定的开放规范，目的是把原本散落在各厂商 BIOS 里、各搞各的以下能力，统一成 **OS 可以直接掌握的一套标准接口**：

- 电源管理（睡眠、唤醒、调频）
- 设备配置与枚举
- 热插拔
- 系统事件处理

**核心思想**：把主导权交给 **OSPM（Operating System-directed configuration and Power Management）**——**OS 当家**，firmware/BIOS 只负责**提供硬件描述信息**。

> 一句话：BIOS 告诉 OS "这台机器长什么样、能怎么省电"，然后 OS 按自己的策略去操作硬件。

**它由什么运转**：一套 **ACPI 表**（描述硬件）+ 里面的 **AML 字节码**（可执行的固件逻辑）+ OS 里的 **AML 解释器**。

---

### 为什么有 ACPI：从 APM 到 OSPM

#### 前辈 APM（Advanced Power Management）

- 套路：**BIOS 说了算**。OS 通过 `INT 15h` 喊一声"我要睡了"，具体怎么睡由 BIOS 自己决定。
- 问题：到了 Win98 / 2000 时代，OS 越来越想主导硬件，与 BIOS 打架；能力也受限，扩展困难。

#### ACPI 的解法：OSPM

- 把**主动权交给 OS**，BIOS 退化为"描述者 + 有限逻辑提供者"。
- OS 能**直接操作电源寄存器**，绕开传统 BIOS 调用（INT 15h）。
- 想扩展？**加个 SSDT 表就行**，不用改 BIOS 二进制。

---

### 整体架构

```
┌───────────────────────────────────────────────┐
│                    OS / OSPM                  │
│   ┌────────────────┐   ┌───────────────────┐  │
│   │ AML Interpreter│──▶│  电源/配置/热插拔   │  │
│   │  (ACPICA等)    │   │     策略与驱动      │  │
│   └───────┬────────┘   └───────────────────┘  │
└───────────┼───────────────────────────────────┘
            │ 读表 + 执行 AML
┌───────────┼───────────────────────────────────┐
│           ▼            Firmware / BIOS        │
│   ┌──────────────┐   ┌──────────────────────┐ │
│   │  ACPI Tables │   │  寄存器接口(PM1/GPE等) │ │
│   │ DSDT/SSDT/.. │   │  + SMM 内部逻辑       │ │
│   └──────────────┘   └──────────────────────┘ │
└───────────────────────────────────────────────┘
            │
┌───────────┼───────────────────────────────────┐
│           ▼               Hardware            │
│   CPU / Memory / PCIe / EC / 电源控制器 ...     │
└───────────────────────────────────────────────┘
```

**三大件**：

1. **ACPI 表**：描述硬件（结构 + 逻辑）
2. **AML**：表里装载的可执行字节码
3. **寄存器接口**：OS 通过 FADT 里给出的地址直接操作

---

### ACPI 表体系

BIOS 开机时把一堆表填好放到内存，OS 启动后解析它们来了解硬件。

#### 表的入口与索引

| 表                                            | 作用                                                                                                                       |
| --------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **RSDP**（Root System Description Pointer）   | 一切表的入口。位于内存固定区域（前 1MB 的 `0xE0000~0xFFFFF`，或 Extended BIOS Data Area）。存有指向 RSDT/XSDT 的地址及校验 |
| **RSDT**（Root System Description Table）     | 表的索引表，列出其他所有表的地址（32 位）                                                                                  |
| **XSDT**（Extended System Description Table） | RSDT 的 64 位版本（ACPI 2.0+）                                                                                             |

> 查找顺序：先找 **RSDP** → 由它找到 **XSDT/RSDT** → 再枚举所有表。

#### 核心表

| 表                            | 全称                                    | 作用                                                                           |
| ----------------------------- | --------------------------------------- | ------------------------------------------------------------------------------ |
| **DSDT**                      | Differentiated System Description Table | **最重要**，描述系统有哪些设备、如何组织（含 AML）                             |
| **SSDT**                      | Secondary System Description Table      | 补充表，可有多个，用于拆分/动态补充描述                                        |
| **FADT/FACP**                 | Fixed ACPI Description Table            | **固定那块**的描述：寄存器物理地址（PM1、PM Timer 等）、支持的睡眠状态、能力位 |
| **MADT/APIC**                 | Multiple APIC Description Table         | 多核 CPU、中断控制器拓扑                                                       |
| **MCFG**                      | Memory Mapped Configuration             | PCIe ECAM 配置空间基址                                                         |
| **SRAT / SLIT**               | NUMA 相关                               | CPU/内存亲和性与距离                                                           |
| **HPET**                      | High Precision Event Timer              | 高精度定时器描述                                                               |
| **BGRT**                      | Boot Graphics Resource Table            | 开机 Logo                                                                      |
| **SPCR**                      | Serial Port Console Redirection         | 串口控制台                                                                     |
| **BERT / EINJ / ERST / HEST** | 错误相关                                | 硬件错误上报与注入                                                             |

> 规范里有几十种表，写代码时以 **ACPI Spec** 为准，配合 **ACPICA** 源码看行为最直观。

---

### AML / ASL 与编译流程

#### 概念

- **ASL**（ACPI Source Language）：你写的**源码**（`.asl`）
- **AML**（ACPI Machine Language）：编译后的**字节码**，装在 DSDT/SSDT 里
- **编译器**：Intel 的 **`iasl`**（ACPI Component Architecture 里的工具）

#### 编译流程

```
xxx.asl  ──[ iasl ]→  xxx.aml  →  打包进 FV / 运行时发布到内存
                                              ↓                                          
                                        OS 里的 AML 解释器执行
```

#### 为什么这样设计

- OS 里带一个 **AML Interpreter**（Windows / Linux 都用 **ACPICA**）。
- 只要解释器在，**同一份 DSDT 能被 Windows、Linux 都认**——这是 ACPI 跨 OS 的关键。

#### ASL 小样（感受一下语法）

```asl
DefinitionBlock ("", "DSDT", 2, "OEMID ", "TABLEID", 0x00000001)
{
    Device (\_SB.PCI0)
    {
        Name (_HID, EisaId ("PNP0A08"))   // PCI 主机桥

        Device (LPCB)
        {
            Name (_ADR, 0x001F0000)       // LPC 桥
        }
    }
}
```

---

### 命名空间

AML 被解析后形成一棵**树状命名空间（Namespace）**，每个节点是一个**对象**。

**根节点 `\`** 下面挂几棵主要的子树：

| 路径    | 含义                             |
| ------- | -------------------------------- |
| `\_SB`  | System Bus，设备基本都挂在这     |
| `\_PR`  | Processor，处理器                |
| `\_TZ`  | Thermal Zone，热区               |
| `\_GPE` | General Purpose Events，通用事件 |
| `\_SI`  | System Indicator                 |
| `\_OS`  | OS 相关信息                      |
| `\_GL`  | Global Lock                      |

**常见路径示例**：

```
\_SB.PCI0                    # PCI 主机桥
\_SB.PCI0.LPCB.EC0           # 嵌入式控制器
\_PR.CPU0                    # 0 号处理器
```

> BIOS 里天天打交道的下划线名字，都是这棵树上的节点。

---

### 对象与控制方法

#### 对象类别

| 类别                                        | 说明                           |
|:-------------------------------------------:|:------------------------------ |
| **DefinitionBlock / Device / Scope**        | 结构界定                       |
| **Name**                                    | 存常量                         |
| **Method**                                  | 可执行方法，OS 可调用          |
| **OperationRegion / Field**                 | 把一段物理地址映射成可读写字段 |
| **Processor / PowerResource / ThermalZone** | 特定设备对象                   |

#### 下划线控制方法（Control Methods）

BIOS 暴露给 OS 的"接口函数"，OS 在特定时机**主动调用**：

| 方法                     | 何时被调   | 干什么                         |
|:------------------------:|:----------:|:------------------------------ |
| `_STA`                   | 查设备状态 | 返回 present / enable 等状态位 |
| `_INI`                   | 初始化设备 | 首次配置                       |
| `_HID` / `_CID`          | 枚举       | 硬件 ID，OS 靠它匹配驱动       |
| `_ADR`                   | 枚举       | 父总线上的地址                 |
| `_CRS`                   | 要资源     | 返回当前资源（IO / MEM / IRQ） |
| `_PRS` / `_SRS`          | 设资源     | 可选项 / 设置资源              |
| `_DSM`                   | 设备特定   | 厂商自定义功能（很多特性靠它） |
| `_PS0` / `_PS3`          | 设备电源   | 打开 / 关闭设备电源（D0/D3）   |
| `_TMP` / `_SCP`          | 温度       | 读温度 / 设被动散热            |
| `_PTS`                   | 睡前       | Prepare To Sleep，睡前一串配置 |
| `_WAK`                   | 醒后       | Wake，恢复现场                 |
| `_S3_` / `_S4_` / `_S5_` | 睡眠       | 描述各睡眠态下要保留的唤醒信息 |

> `_PTS` 和 `_WAK` 是调睡眠问题时**最常动的两个**。

---

### 电源状态体系（重点）

> **核心认知**：Gx / Sx / Cx / Px / Dx **不是并列，而是分层嵌套**。
> 一台机器现在的状态 = **每层各处于某个状态，叠加在一起**。

**举例：按下睡眠键**

- **全局层**：G0 → G1
- **系统睡眠层**：S0 → S3
- **CPU 层**：C0 → 深 C 态（甚至整颗掉电）
- **设备层**：各设备 D0 → D3

四层**同时**在动。调 bug 时要**按层切开看**，别一锅乱炖。

#### 第一层：Gx（全局状态 Global State）

描述**整机电源域**状态，共四个：

| 状态   | 名称           | 掉电情况 | 对应               |
| ------ | -------------- | -------- | ------------------ |
| **G0** | Working        | 全供电   | 正常工作 / S0      |
| **G1** | Sleeping       | 部分供电 | 睡眠（内含 S1~S4） |
| **G2** | Soft Off       | 仅待机电 | 软关机（= S5）     |
| **G3** | Mechanical Off | 彻底没电 | 拔电源 / 关插座    |

**要点**：

- **G1 不是单一状态，是一族**，里面装 S1~S4。
- **G2 与 S5 是同一个东西的两个名字**，规范常写 `G2/S5`。
- **G3 才真正掉光**；G2/S5 下主板仍有 **5V standby**，RTC、电源按钮、网卡唤醒逻辑都活着。
- **G3 只能靠断物理电进入**，软件进不去。
- **G3 → 冷启动（完整 POST）**；**G1 → 唤醒（走 `_WAK`）**，两条路完全不同。

#### 第二层：Sx（睡眠状态 Sleep State）

日常最常打交道的一层。**S1/S2 基本绝迹**，重点是 S3/S4/S5 + 新势力 S0ix。

| 状态   | 名字             | CPU        | 内存     | 上下文     | 唤醒速度   |
| ------ | ---------------- | ---------- | -------- | ---------- | ---------- |
| **S1** | POS / Stop Grant | 停（供电） | 供电     | 保持       | 快         |
| **S2** | 更深停           | 掉电       | 供电     | 内存保留   | 快         |
| **S3** | Suspend to RAM   | **掉电**   | **供电** | 内存       | 秒级       |
| **S4** | Suspend to Disk  | 掉电       | 掉电     | **写硬盘** | 慢         |
| **S5** | Soft Off         | 掉电       | 掉电     | 不保存     | 完整冷启动 |

**S1 vs S2**：差别在 CPU 供电。S1 时钟停、上下文没丢；S2 CPU 电源也断，cache/寄存器上下文没了（内存仍在）。现代平台基本**直接砍掉 S1/S2**。

**S3 要点**：

- CPU 整体掉电，**内存靠 standby 电压维持**（掉电就丢）。
- 唤醒路径：`_PTS`（睡前配置）→ 进 S3 → 硬件唤醒事件 → 复位 → 固件恢复 → `_WAK`（恢复）→ 交回 OS。
- 是 BIOS 活最重的一环：唤醒源配置、内存训练记忆、设备状态恢复。

**S4 要点**：

- 内存内容整个 dump 到硬盘（Windows：`hiberfil.sys`），然后**彻底断电**。
- 醒来 = **开机 + 从硬盘读回镜像**两步：先正常 POST，再灌回内存恢复现场。
- 两种实现：**S4 BIOS**（BIOS 负责存恢复）/ **S4 OS**（OS 自己存，BIOS 只当普通开机）——现在基本后者。
- **关键区别**：S3 怕掉电，S4 不怕。笔记本合盖久放要进 S4。

**S5 要点**：

- 就是"关机"，内存全丢，只剩待机电。
- 但**不是死的**：RTC 定时、电源按钮、WoL、USB 唤醒都能拉起它。
- 从 S5 起来 = 冷启动，无上下文恢复。

**S0ix（Modern Standby / Connected Standby）**：

- 思路完全不同：**机器一直停在 S0**（全局不睡），只让 CPU 进很深 idle、设备进低功耗，但**随时能响应网络/后台收邮件**，像手机。
- 好处：唤醒"瞬时"（本就没真睡）；坏处：**比 S3 费电**。
- 现代轻薄本很多**禁掉 S3、只留 S0ix**。BIOS 菜单里看到 "Modern Standby" / "Connected Standby" 就是它。
- 规范层面靠 `Low Power S0 Idle` capability bit（FADT flag）标识。

#### 第三层：Cx（CPU 空闲状态 C-State）

管 **CPU 在工作态下闲着时怎么省电**，与睡眠是两码事。

| 状态           | 说明                                            | 唤醒延迟 |
| -------------- | ----------------------------------------------- | -------- |
| **C0**         | 运行，正常执行                                  | —        |
| **C1**         | Halt，停时钟，随时被中断唤醒                    | 极低     |
| **C2/C3...Cn** | 越深关得越多（时钟、cache、核心电压），省电越多 | 越大     |

**要点**：

- Cx 是 **idle 省电**，进得去出得来，**不影响系统运行**——CPU 一闲就自动往下钻。
- 影响的是**唤醒延迟**。BIOS 里常见 "C-State 开关 / latency 配置"：**关了省电、开了响应快**（取舍相反）。
- 深度 C 态可能 flush cache、甚至核心断电（C6/CC6，核心/package 级）。
- **别混**：Sx 是**整机**睡眠，Cx 是 **CPU 单独**空闲。

#### 第四层：Px（性能状态 P-State）

| 概念        | 管什么                        |
| ----------- | ----------------------------- |
| **C-State** | **闲时省电**（idle）          |
| **P-State** | **忙时快慢**（调频调压 DVFS） |

- **P0**：最高性能（最高频/压）；**P1...Pn**：逐级降压降频，越省电越慢。
- P-State 是 **"跑多快"**的权衡，**不是睡眠**——系统照样干活，只是动态调频换功耗。
- 现代平台主要由 **HWP（Hardware P-States，Intel Speed Shift）** 接管：**硬件自己根据负载调**，比早期 OS 驱动方式快。BIOS 里可见 "HWP"、"Speed Shift"、"EPP"（Energy Performance Preference）。
- **T-State（Throttling）**：老概念，通过降时钟占空比强制降速，多用于**过热保护**；现在基本被 P/C-State + thermal 框架吸收。

> **一句话理清 C 和 P**：**闲了降 C（省 idle 功耗），忙了调 P（平衡性能功耗）**。两个维度。

#### 第五层：Dx（设备状态 Device Power State）

**每颗设备自己的电源状态**，由各自驱动管理。

| 状态       | 含义                      | 上下文 | 功耗 |
| ---------- | ------------------------- | ------ | ---- |
| **D0**     | 全开                      | 保存   | 最高 |
| **D1/D2**  | 中间态（很多设备不实现）  | 部分   | 中   |
| **D3hot**  | 软关（仍在 standby 电下） | 丢     | 低   |
| **D3cold** | 冷关（主电断）            | 丢     | 近零 |

**要点**：

- D1/D2 可选，实际多为 **D0 / D3hot / D3cold** 三档。
- **D3hot vs D3cold 差在供电**：hot 还在 standby 电下、只是功能关了；cold 主电都断，靠外部事件重新上电。
- **驱动 + 固件配合**：驱动决定何时放 D3；固件（ACPI 表里 `_PS0/_PS3` + PowerResource）负责实际断电/上电。
- 进 S3/S4 前，OS 先把各设备放到合适的 Dx，唤醒后再拉回 D0。
- **"设备睡不醒/醒不来"的 bug 基本在这层**——多半 `_PS3`/`_PS0` 没配好或 PowerResource 时序不对。

#### 各层如何嵌套（两个例子）

**例 1：正常工作，啥也没干**

```
G0 (Working)
  └─ S0 (工作)
       ├─ CPU: 钻进 C6/C7（闲着省电）
       ├─ P-State: 降到低频（负载低）
       └─ 大部分设备: 驱动按需放到 D3hot
```

**例 2：按下睡眠键（S3）**

```
G0 → G1          # 全局进睡眠
S0 → S3          # 系统睡眠
CPU: C0 → ... → 整颗 CPU 掉电（内存保电）
设备: 各自 D0 → D3cold（按能力）
固件: _PTS 先跑，睡前配置唤醒源
```

唤醒时反向走：硬件事件 → `_WAK` → 设备回 D0 → CPU 回来 → 交 OS。

---

### 事件机制：GPE / SCI / SMI

设备要"叫醒" OS 靠这三个东西（**别混淆**）：

| 机制    | 全称                        | 谁处理        | 说明                                                                             |
| ------- | --------------------------- | ------------- | -------------------------------------------------------------------------------- |
| **GPE** | General Purpose Event       | OS            | 通用事件引脚，分 GPE0/GPE1 block；USB 插入、温度超限、网卡收包都触发某个 GPE bit |
| **SCI** | System Control Interrupt    | OS            | GPE 触发后产生 SCI 中断，告知 OS"有事"，OS 去读 GPE 寄存器看是哪个               |
| **SMI** | System Management Interrupt | **BIOS/固件** | 固件内部维护用，**OS 看不见**                                                    |

**事件流**：

```
设备事件 → GPE bit 置位 → SCI 中断 → OS 读 GPE 寄存器 → 调对应 _Lxx/_Exx 方法处理
```

---

### 睡眠与唤醒流程

#### 睡眠（以 S3 为例）

```
1. OS 决定进 S3
2. OS 调 _PTS（Prepare To Sleep）   ← 固件睡前配置：唤醒源、设备状态保存
3. OS 把设备放到合适的 Dx
4. OS 写 PM1 控制寄存器，触发睡眠
5. 硬件进入 S3：CPU 掉电，内存保电
```

#### 唤醒

```
1. 硬件唤醒事件（电源键/网卡/RTC/USB...）
2. 退出 S3，CPU 复位
3. 固件自检 / 恢复记忆（内存训练等）
4. OS 调 _WAK（Wake）              ← 固件恢复现场
5. 设备回 D0，交回 OS
```

> **FADT** 里的 `PM1a_CNT_BLK`、`PM1_EVT_LEN`、`SMI_CMD`、`ACPI_ENABLE` 等字段，就是告诉 OS "睡眠往哪个 IO 端口写什么值"。

---

### BIOS 在 ACPI 中的实际工作

实际开发中你会碰到的点：

1. **生成 ACPI 表**

   - EDK2 常见做法：ASL（`.asl`）→ `iasl` 编译成 AML → 打包进 FV → 运行时由 `AcpiTableDxe` 等发布到内存。
   - 也可**运行时动态 patch 表**（如按配置改 DSDT）。

2. **填 FADT / 发布寄存器地址**

   - 把 PM1、PM Timer、SMI_CMD、ACPI_ENABLE 等地址与能力位报告给 OS。

3. **处理 `_PTS` / `_WAK`**

   - 固件睡眠/唤醒逻辑的落点，**S3 问题一半以上从这查**。

4. **处理 GPE / 唤醒源**

   - 哪些设备能唤醒、唤醒后如何恢复。

5. **OSPM 越过 SMM 直接操作硬件**

   - OS 直接操作电源寄存器，绕开传统 BIOS 调用——ACPI 相对 APM 最大的变化。

---

### ACPI vs 传统 BIOS 调用

| 维度     | 传统（APM / INT 15h） | ACPI                    |
|:--------:| --------------------- | ----------------------- |
| 谁做主   | BIOS                  | **OS（OSPM）**          |
| 硬件描述 | 靠 BIOS 调用返回      | **靠表描述**            |
| 睡眠     | BIOS 自己决定流程     | OS 按表操作寄存器       |
| 事件     | BIOS 拦截             | **GPE → SCI → OS 处理** |
| 扩展性   | 难                    | **加个 SSDT 就行**      |

---

### 规范版本

ACPI 现由 **UEFI Forum** 维护，从 1.0 至今 **6.x**。

| 版本          | 里程碑                        |
| ------------- | ----------------------------- |
| **1.0**       | 基础框架                      |
| **2.0**       | 64 位、XSDT                   |
| **4.0+**      | 新增多表（BGRT、SLIT 等）     |
| **5.0 / 6.0** | 低功耗 idle、更多设备相关能力 |

> 规范文档（ACPI Spec）是写代码时的圣经；**ACPICA** 源码是理解命名空间与解释器行为的最佳补充。

---

### 调试要点与常见坑

- **先分清来路**：机器是 **G3 冷启动** 还是 **G1 唤醒**？流程完全不同。
- **按层排查**：全局进对了吗？CPU 进了吗？那块设备进 D3 了吗？——别只看一层。
- **`_PTS` / `_WAK` 是重灾区**：唤醒源 enable、设备状态保存/恢复。
- **FADT 能力位**：想让机器支持某睡眠态，得先把对应能力位配出来。
- **Dx + PowerResource**：设备电源域开关时序错 = 睡眠/唤醒故障。
- **S0ix 是另一套 debug 方法**：调的不是"能不能进 S3"，而是"**能不能进最深的低功耗 S0 idle**"——看 SOC 各 IP 是否让出时钟/电源、有无 IP 阻止进最低功耗，靠 **residency 计数器**排查。
- **别混层**：Sx 整机、Cx CPU 空闲、Px CPU 性能、Dx 单设备。

---

### 术语速查

| 缩写      | 全称                                           | 含义                     |
| --------- | ---------------------------------------------- | ------------------------ |
| ACPI      | Advanced Configuration and Power Interface     | 高级配置与电源接口       |
| OSPM      | OS-directed configuration and Power Management | OS 主导的配置与电源管理  |
| APM       | Advanced Power Management                      | ACPI 的前辈              |
| ASL       | ACPI Source Language                           | 你写的源码               |
| AML       | ACPI Machine Language                          | 编译后的字节码           |
| RSDP      | Root System Description Pointer                | 表入口                   |
| RSDT/XSDT | Root/Extended System Description Table         | 表索引表                 |
| DSDT      | Differentiated System Description Table        | 主描述表                 |
| SSDT      | Secondary System Description Table             | 补充表                   |
| FADT/FACP | Fixed ACPI Description Table                   | 固定描述表               |
| MADT      | Multiple APIC Description Table                | 中断控制器描述           |
| MCFG      | Memory Mapped Configuration                    | PCIe ECAM 基址           |
| GPE       | General Purpose Event                          | 通用事件                 |
| SCI       | System Control Interrupt                       | 系统控制中断             |
| SMI       | System Management Interrupt                    | 系统管理中断（固件内部） |
| HWP       | Hardware P-States                              | Intel Speed Shift        |

---

### 学习资源

- **ACPI Specification**（UEFI Forum 官网）—— 权威规范，写代码必查
- **ACPICA**（ACPI Component Architecture）—— Intel 开源，含 `iasl` 编译器 + OS 无关的解释器
- **EDK2** —— 看 `AcpiTableDxe`、各种 `*AcpiTable` 模块、`.asl` 文件
- **Linux 内核** `drivers/acpi/` —— 看 OS 侧如何用表、跑方法

---

> 笔记持续补充。建议结合真实平台（具体 DSDT/SSDT 内容、`_PTS`/`_WAK` 实现、GPE 处理）对照学习，效果最好。

 
