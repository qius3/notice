UEFI 固件并不是把所有模块按顺序执行一遍，而是由 **固件阶段、模块类型、依赖表达式和协议安装/消费关系**共同驱动。你看到的“模块彼此独立”，实际上是 UEFI 的一种设计目标：模块通过标准接口协作，而不是直接互相调用。

## 一、整体启动链路

典型的启动流程如下：

```text
复位
  ↓
SEC
  ↓
PEI
  ↓
DXE
  ↓
BDS
  ↓
操作系统 Loader
  ↓
Runtime Services
```

在某些平台上还会有独立的 SMM/Management Mode 代码。

---

## 二、SEC：最早期启动

SEC，即 Security/Pre-EFI Initialization，主要由 CPU 复位入口开始执行。

它负责：

- 建立最初的执行环境；
- 设置临时栈；
- 初始化最基础的 CPU 状态；
- 建立临时内存，通常使用 Cache-as-RAM；
- 找到并调用 PEI 阶段入口；
- 传递早期启动信息，例如 `SEC HOB`。

SEC 通常不是一个复杂的模块调度阶段，代码规模较小，很多实现是平台相关的汇编或 C 代码。

大致可以理解为：

```text
CPU Reset Vector
    ↓
SEC Core
    ↓
PEI Core
```

---

## 三、PEI：初始化内存和平台基础环境

PEI，即 Pre-EFI Initialization，负责把平台从“刚上电状态”初始化到可以运行大部分 C 代码的状态。

PEI 中的主要组件是：

### 1. PEI Core

PEI Core 是 PEI 阶段的调度核心，负责：

- 管理 PEI Services；
- 调度 PEIM；
- 解析 PEIM 的依赖关系；
- 管理 PPI；
- 创建 HOB；
- 加载后续 DXE 阶段所需的固件文件。

### 2. PEIM

PEIM 是 PEI Module，例如：

- 内存初始化 PEIM；
- CPU 初始化 PEIM；
- 芯片组初始化 PEIM；
- SPI Flash PEIM；
- 固件卷发现 PEIM；
- 安全验证 PEIM；
- DXE IPL PEIM。

PEIM 之间主要通过 **PPI** 通信。

PPI 可以看作 PEI 阶段的接口：

```text
内存初始化 PEIM
    └── 安装 Memory Discovered PPI

其他 PEIM
    └── 查找并使用 Memory Discovered PPI
```

PEI 阶段还会生成大量 HOB，即 Hand-Off Block。例如：

- 内存资源 HOB；
- 固件卷 HOB；
- CPU 信息 HOB；
- ACPI 地址信息；
- Boot Mode 信息。

这些 HOB 会传给 DXE 阶段。

---

## 四、DXE：大部分 UEFI 驱动在这里运行

DXE，即 Driver Execution Environment，是你在 EDK2 中最常见的阶段。

DXE 的核心是：

```text
DXE Core
    ├── DXE Dispatcher
    ├── Boot Services
    ├── Runtime Services
    ├── Protocol Database
    └── 各类 DXE Driver
```

### 1. DXE Core

DXE Core 初始化：

- 内存管理；
- 事件系统；
- 定时器；
- Boot Services；
- Runtime Services；
- Protocol Database；
- DXE Dispatcher。

### 2. DXE Driver

DXE Driver 负责实现各种硬件和固件功能，例如：

- PCI 总线驱动；
- USB 总线驱动；
- SATA/NVMe 驱动；
- 网络驱动；
- 文件系统驱动；
- 图形输出驱动；
- 键盘、鼠标驱动；
- ACPI 驱动；
- TPM 驱动；
- 调试和诊断驱动。

但是，**DXE Driver 并不一定一启动就执行**。它是否被调度，取决于：

1. 它是否位于当前加载的 Firmware Volume 中；
2. 它的依赖表达式 `Depex` 是否满足；
3. 当前平台是否需要它；
4. 是否有其他模块触发它；
5. 它是否是某种特殊类型的驱动。

---

## 五、模块之间如何“关联”？

主要有四种机制。

# 1. Depex：声明调度依赖

EDK2 模块通常可以在 INF 文件中指定：

```ini
[Depex]
  gEfiPciRootBridgeIoProtocolGuid
```

含义是：

> 只有当 `EFI_PCI_ROOT_BRIDGE_IO_PROTOCOL` 已经被安装之后，才可以调度此驱动。

例如：

```text
PciHostBridgeDxe
    └── 安装 PCI Root Bridge I/O Protocol

PciBusDxe
    └── Depex: 依赖 PCI Root Bridge I/O Protocol
```

于是 DXE Dispatcher 的执行关系大致是：

```text
PciHostBridgeDxe
    ↓ 安装协议
PciBusDxe 被调度
```

Depex 常见操作包括：

```text
TRUE
FALSE
AND
OR
NOT
```

例如：

```ini
[Depex]
  gEfiPciIoProtocolGuid AND
  gEfiDevicePathProtocolGuid
```

这就构成了一个动态依赖图。

---

# 2. Protocol：运行时接口

DXE 阶段模块之间最重要的通信机制是 Protocol。

一个驱动可以：

```c
gBS->InstallProtocolInterface(...)
```

安装一个协议；另一个驱动可以：

```c
gBS->LocateProtocol(...)
```

或者：

```c
gBS->OpenProtocol(...)
```

查找和使用这个协议。

例如：

```text
网络控制器驱动
    └── 安装 EFI_SIMPLE_NETWORK_PROTOCOL

网络协议栈
    └── 查找 EFI_SIMPLE_NETWORK_PROTOCOL

PXE Boot
    └── 使用网络协议栈
```

协议本质上类似于：

```text
GUID + 结构体函数表
```

可以把它理解为一种动态的、带 GUID 标识的接口。

---

# 3. Handle：协议的载体

UEFI 中的 Protocol 一般安装在 Handle 上。

一个 Handle 可以代表：

- 一个 PCI 设备；
- 一个 USB 设备；
- 一个硬盘；
- 一个文件系统；
- 一个控制器；
- 一个驱动创建的抽象对象。

例如：

```text
Handle A
 ├── EFI_PCI_IO_PROTOCOL
 └── EFI_DEVICE_PATH_PROTOCOL

Handle B
 ├── EFI_BLOCK_IO_PROTOCOL
 └── EFI_DEVICE_PATH_PROTOCOL

Handle C
 └── EFI_SIMPLE_FILE_SYSTEM_PROTOCOL
```

设备驱动通常按照如下方式协作：

```text
PCI Bus Driver
    └── 产生 PCI Device Handle

NVMe Driver
    └── 在 PCI Device Handle 上安装 Block I/O Protocol

Partition Driver
    └── 消费 Block I/O，产生分区 Handle

File System Driver
    └── 消费分区 Block I/O，产生 Simple File System Protocol
```

所以很多模块虽然没有直接调用关系，但通过 Handle 和 Protocol 形成了间接关系。

---

# 4. HOB：PEI 到 DXE 的交接数据

PEI 和 DXE 之间通常不是直接调用关系，而是通过 HOB 交接信息。

```text
PEI 内存初始化
    └── 创建内存资源 HOB

DXE Core
    └── 读取内存资源 HOB

DXE 内存管理器
    └── 根据 HOB 管理内存
```

HOB 相当于启动阶段之间的结构化消息或启动上下文。

---

## 六、Firmware Volume 中的模块不是都一定会运行

EDK2 编译后，模块通常被打包进 Firmware Volume，结构大致是：

```text
Firmware Volume
 ├── PEI Core
 ├── PEIM A
 ├── PEIM B
 ├── DXE Core
 ├── DXE Driver A
 ├── DXE Driver B
 ├── Application
 ├── Firmware Volume Image
 └── Raw data
```

但“被打包进去”不等于“肯定执行”。

可能出现以下情况：

### 1. 依赖不满足

例如驱动依赖某协议，但平台没有提供这个协议：

```text
Driver A
  Depex: Protocol X

Protocol X 未安装
  ↓
Driver A 永远不会被调度
```

### 2. 平台 DSC 没有包含它

INF 只是描述模块，真正决定模块是否进入某个平台固件的通常是 DSC：

```ini
[FV.MAIN]
  INF MdeModulePkg/Universal/Network/DhcpDxe/DhcpDxe.inf
```

没有被 DSC/FDF 包含的模块，不会进入最终固件。

### 3. 固件卷没有被加载

某些 FVs 是按需加载的，比如：

- 网络 FV；
- 恢复 FV；
- 厂商扩展 FV；
- 诊断 FV；
- Option ROM 中的 FV。

### 4. 模块是库，不是独立运行实体

很多 EDK2 目录中的模块实际上是 Library：

```text
BaseLib
BaseMemoryLib
DebugLib
UefiLib
DevicePathLib
PciLib
```

库代码会被链接到其他模块中，本身没有独立入口，也不会由 DXE Dispatcher 调度。

例如：

```text
MyDriver
 ├── UefiDriverEntryPoint
 ├── UefiLib
 ├── DebugLib
 └── BaseMemoryLib
```

最终运行的是 `MyDriver`，而不是单独运行 `UefiLib`。

---

## 七、DXE Dispatcher 是如何决定执行顺序的？

可以把 DXE Dispatcher 简化为以下算法：

```text
1. 扫描 Firmware Volume
2. 找到所有 DXE_DRIVER 文件
3. 解析每个驱动的 Depex
4. 找出当前依赖已经满足的驱动
5. 加载并执行其入口函数
6. 驱动安装新的 Protocol
7. 重新检查其他驱动的 Depex
8. 循环直到没有新的可调度驱动
```

伪代码：

```text
while true:
    progress = false

    for driver in discovered_drivers:
        if driver.not_dispatched and depex_is_satisfied(driver):
            load(driver)
            call_entry_point(driver)
            mark_dispatched(driver)
            progress = true

    if not progress:
        break
```

注意：这不是简单的源码顺序，也不完全是固定拓扑排序。因为驱动入口函数执行后可能动态安装协议，从而改变后续调度条件。

---

## 八、一个实际的设备启动例子

以从 NVMe 硬盘启动为例：

```text
SEC
  ↓
PEI Core
  ↓
内存初始化 PEIM
  ↓
DXE Core
  ↓
PCI Host Bridge Driver
  └── 安装 PCI Root Bridge I/O
        ↓
PCI Bus Driver
  └── 枚举 NVMe PCI 设备
        ↓
NVMe Driver
  └── 安装 Block I/O
        ↓
Partition Driver
  └── 识别 GPT 分区
        ↓
File System Driver
  └── 安装 Simple File System
        ↓
BDS
  └── 查找 EFI/BOOT/BOOTX64.EFI
        ↓
加载操作系统 Boot Manager
```

这里每个模块看起来都是独立的，但实际关系是：

```text
协议安装 → 依赖满足 → 驱动调度 → 新协议产生
```

也可以表示为：

```text
PCI Root Bridge
    ↓
PCI Bus
    ↓
NVMe Controller
    ↓
Block I/O
    ↓
Partition
    ↓
File System
    ↓
Boot Loader
```

---

## 九、BDS 阶段负责“从固件功能转向启动目标”

BDS，即 Boot Device Selection，不是简单的设备驱动调度阶段，而是负责选择启动设备。

它会：

- 处理平台启动策略；
- 读取 `Boot####`、`BootOrder` 等变量；
- 连接设备；
- 查找文件系统；
- 加载 EFI 应用程序；
- 启动 Windows Boot Manager、GRUB 或其他 EFI Loader；
- 处理启动菜单和启动失败策略。

典型入口关系：

```text
DxeCore
    ↓
BdsDxe
    ↓
Platform BDS Policy
    ↓
Boot Manager
    ↓
EFI Loader
```

---

## 十、Runtime Services 和 Boot Services

操作系统启动后，绝大多数 DXE 驱动会退出使用，或者被操作系统接管。

### Boot Services

只在 `ExitBootServices()` 之前可用，例如：

- 分配内存；
- 查找协议；
- 加载镜像；
- 创建事件；
- 访问设备；
- 操作 Handle。

### Runtime Services

操作系统启动后仍然可用，例如：

- `GetVariable()`；
- `SetVariable()`；
- `GetTime()`；
- `ResetSystem()`；
- 某些平台的固件运行时服务。

执行过程：

```text
UEFI Boot Manager
    ↓
操作系统 Loader
    ↓
ExitBootServices()
    ↓
操作系统接管硬件
```

调用 `ExitBootServices()` 后，普通 Boot Service 驱动不应再被使用。

---

## 十一、SMM 是另一条相对独立的执行路径

SMM，即 System Management Mode，通常用于：

- 电源管理；
- 风扇控制；
- 安全处理；
- 固件更新；
- ACPI 事件；
- 平台厂商管理功能。

它并不完全按照普通 DXE 驱动的方式运行，而是由 SMI 触发：

```text
硬件事件 / 软件触发 SMI
    ↓
CPU 进入 SMM
    ↓
SMM Handler 执行
    ↓
恢复原执行环境
```

EDK2 中常见的 SMM 模块包括：

- SMM Driver；
- SMM Core；
- Standalone MM Driver；
- SMI Handler。

---

## 十二、如何理解“模块独立”这个现象？

你的观察是正确的，但 EDK2 的独立性是分层次的：

```text
源代码层面：模块相互独立
        ↓
构建层面：DSC/FDF 决定哪些模块进入固件
        ↓
装载层面：FV/FFS 决定模块如何存放
        ↓
调度层面：Depex 决定何时执行
        ↓
运行层面：Protocol/Handle 决定如何协作
        ↓
启动策略：BDS 决定最终启动什么
```

因此，EDK2 不是一个传统的：

```text
main()
 ├── initA()
 ├── initB()
 ├── initC()
 └── boot()
```

而更像是：

```text
事件驱动 + 依赖驱动 + 协议驱动的插件系统
```

---

## 十三、如果要建立 EDK2 知识图谱，应该抓哪些关系？

可以重点抽取以下实体和关系：

### 实体

- `INF` 模块；
- `DSC` 平台描述；
- `FDF` 固件布局；
- `FV`；
- `FFS`；
- PEIM；
- DXE Driver；
- UEFI Application；
- Library；
- Protocol；
- PPI；
- HOB；
- GUID；
- Handle；
- 硬件设备；
- BDS 启动项。

### 关系

```text
模块 --包含于--> Firmware Volume
模块 --由 DSC 引入--> Platform
模块 --依赖--> Protocol
模块 --安装--> Protocol
模块 --消费--> Protocol
模块 --依赖--> PPI
模块 --创建--> HOB
模块 --链接--> Library
驱动 --绑定--> 设备
BDS --加载--> EFI Application
驱动 --实现--> Protocol
```

其中最重要的图不是单纯的函数调用图，而是：

```text
模块
  ├── Depex 依赖
  ├── Protocol 安装/消费
  ├── PPI 安装/消费
  ├── HOB 生产/消费
  ├── DSC/FDF 打包关系
  └── 硬件设备绑定关系
```

固件为什么会启动、哪些模块会被调度，以及它们如何协作。
