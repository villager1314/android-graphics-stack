# android-graphics-stack

Android 图形软件栈：从 GPU 硬件、底层驱动到 Mesa、API 转换层，以及 Minecraft 启动器和 Switch 模拟器中的图形实现。

> 核查日期：2026-10-07。按**底层 → 上层**介绍；同类项目放在同级标题下。具体硬件兼容范围以对应版本的官方文档、源码设备表和发布说明为准。

## 软件栈树状图

沿树枝由左向右查看分类，五个层级按 ① → ⑤ 从底层到上层排列。连线表示包含关系，实际调用路径见文末表格。

```mermaid
flowchart LR
    ROOT["Android 图形软件栈"]

    ROOT --- HW["① GPU 硬件"]
    ROOT --- KERNEL["② 固件与内核"]
    ROOT --- USER["③ 用户态驱动"]
    ROOT --- MIDDLE["④ 接口与中间层"]
    ROOT --- APP["⑤ 应用实现"]

    HW --- ADRENO["Adreno"]
    HW --- MALI["Mali / Immortalis"]
    HW --- OTHER["PowerVR / Xclipse"]

    KERNEL --- FW["GPU 固件"]
    KERNEL --- AK["Adreno：KGSL / msm DRM"]
    KERNEL --- MK["Mali：kbase / panfrost / panthor"]

    USER --- VENDOR["厂商驱动：Qualcomm / Arm"]
    USER --- MESA["Mesa 硬件驱动"]
    USER --- CPU["CPU：llvmpipe / softpipe
Lavapipe / SwiftShader"]
    MESA --- FD["Adreno：Freedreno / Turnip"]
    MESA --- PAN["Mali：Lima / Panfrost / PanVK"]

    MIDDLE --- API["API：OpenGL / GLES
Vulkan / Direct3D"]
    MIDDLE --- TRANS["API 转换"]
    MIDDLE --- PLATFORM["平台：EGL / GLX / OSMesa / WSI"]
    MIDDLE --- VIRTUAL["虚拟化：VirGL / Venus"]
    TRANS --- GLES["GL → GLES：GL4ES / NG-GL4ES
LTW / MobileGlues"]
    TRANS --- VK["GL → Vulkan：Zink
GLES → 多后端：ANGLE"]
    TRANS --- D3D["D3D → Vulkan：DXVK / VKD3D-Proton"]

    APP --- MC["Minecraft 启动器"]
    APP --- NS["Switch 模拟器"]
    MC --- FCL["FCL 内置项与插件 / 游戏模组"]
    NS --- EMU["GPU 翻译后端 / 驱动加载"]
```

## 目录

1. [GPU 硬件](#1-gpu-硬件)
2. [固件与内核驱动](#2-固件与内核驱动)
3. [用户态驱动与 Mesa](#3-用户态驱动与-mesa)
4. [图形 API 与平台接口](#4-图形-api-与平台接口)
5. [API 转换与兼容层](#5-api-转换与兼容层)
6. [图形虚拟化与运行环境](#6-图形虚拟化与运行环境)
7. [Minecraft 启动器与 FCL 渲染器](#7-minecraft-启动器与-fcl-渲染器)
8. [Android Switch 模拟器与驱动包](#8-android-switch-模拟器与驱动包)
9. [选择与排查](#9-选择与排查)

## 1. GPU 硬件

GPU 执行顶点处理、纹理采样、着色与光栅化。硬件提供哪些能力、驱动实现哪些 API、应用需要哪些功能，共同决定兼容性。

### 1.1 Adreno

Qualcomm 的 GPU 家族，常见于 Snapdragon。Android 通常使用 Qualcomm 闭源用户态驱动；开源路线包括 **Freedreno（OpenGL/GLES）** 和 **Turnip（Vulkan）**。

Adreno 的 Android 自定义驱动生态较完善：支持的应用可借助 libadrenotools 加载另一套用户态 Vulkan 驱动。是否兼容仍需核对真实 GPU 型号、芯片 ID、内核接口和驱动构建。

参考：[Mesa Adreno 文档](https://docs.mesa3d.org/drivers/freedreno.html)。

### 1.2 Mali 与 Immortalis

Arm 提供的 GPU IP/产品系列，由联发科、瑞芯微等厂商集成。不同代际包括 Utgard、Midgard、Bifrost、Valhall 等，不能只看“Mali”名称判断支持情况。

Android 通常使用厂商 Mali 驱动；开源路线包括 **Lima、Panfrost、PanVK**。它们在 Linux 中支持某颗 GPU，并不意味着该 GPU 所在的 Android 手机能直接导入驱动 ZIP：内核接口和平台集成也必须匹配。

参考：[Panfrost/PanVK 支持表](https://docs.mesa3d.org/drivers/panfrost.html)、[Lima](https://docs.mesa3d.org/drivers/lima.html)。

### 1.3 其他 GPU

| 家族 | 常见平台 | 适配要点 |
| --- | --- | --- |
| PowerVR | 部分移动与嵌入式 SoC | 使用其对应驱动，开源支持需逐型号确认 |
| Xclipse | 部分 Samsung Exynos | 使用相应厂商图形栈，不能套用 Turnip |
| NVIDIA Tegra | Switch 等设备 | 是 Switch 模拟器的客机 GPU；需要翻译到安卓宿主 GPU |

**开源驱动不等于硬件、固件和整机系统全部开源。**用户态驱动、内核驱动、固件和硬件文档应分别看待。

## 2. 固件与内核驱动

### 2.1 GPU 固件

固件运行在 GPU 内部控制处理器等单元上，参与命令处理或调度。部分 GPU 即使采用开源驱动，仍需加载厂商固件。

固件、用户态驱动和内核驱动是不同组件；模拟器中导入的普通驱动 ZIP 通常不会替换 GPU 固件。

### 2.2 Adreno 内核驱动

| 项目 | 常见环境 | 职责 |
| --- | --- | --- |
| KGSL | Qualcomm Android 内核 | 为用户态驱动提供设备访问、内存与命令提交等接口 |
| msm DRM | 主线 Linux | 提供 Adreno 图形设备的 Linux DRM 接入 |

同属 Adreno，也可能需要不同构建：**Android/KGSL 驱动包与 Linux/msm DRM 库不能直接视为可互换文件。**

参考：[Mesa Freedreno/Turnip](https://docs.mesa3d.org/drivers/freedreno.html)。

### 2.3 Mali 内核驱动

| 项目 | 常见环境 | 适配范围 |
| --- | --- | --- |
| Mali kbase 等厂商接口 | Android 与厂商 Linux | 依设备所用厂商驱动栈而定 |
| panfrost DRM | 主线 Linux | 支持的 Mali 硬件，依内核版本而定 |
| panthor DRM | 主线 Linux | 采用 CSF 等架构机制的支持型号 |

**Panthor 是内核驱动，PanVK 是用户态 Vulkan 驱动。**名称接近，但层级不同。

参考：[Linux Panfrost](https://docs.kernel.org/gpu/panfrost.html)、[Linux Panthor](https://docs.kernel.org/gpu/panthor.html)。

### 2.4 显示与缓冲区接口

GPU 负责渲染，显示控制器负责输出画面；窗口系统和合成器还要接收、共享并呈现缓冲区。因此，“GPU 可以计算”与“应用能正确显示”需要分别验证。

Android 的缓冲区分配、mapper、同步和呈现接口，也是移植驱动时需要匹配的部分。

## 3. 用户态驱动与 Mesa

用户态驱动接收图形 API 请求，编译着色器、管理资源并生成 GPU 命令，再通过内核接口提交。

### 3.1 Mesa 与 Gallium3D

**Mesa 是项目集合，不是一个单独驱动。**它包含多种 API 实现、硬件驱动、软件渲染器、编译器和平台适配代码。

**Gallium3D 是 Mesa 内部的驱动框架。**Freedreno、Panfrost、Zink、VirGL、llvmpipe 等复用其接口；Turnip 等 Vulkan 驱动有各自的 API 实现路径。

安装 Mesa 后，实际后端可能是硬件驱动，也可能是 llvmpipe。Mesa 的版本号也不等于某颗 GPU 支持的 API 版本。

参考：[Mesa](https://docs.mesa3d.org/index.html)、[Gallium](https://docs.mesa3d.org/gallium/index.html)、[许可证](https://docs.mesa3d.org/license.html)。

### 3.2 厂商硬件驱动

#### 3.2.1 Qualcomm 驱动

Android 系统中的 Adreno GLES/Vulkan 实现通常以闭源二进制随固件提供。模拟器里的“Qualcomm 自定义驱动包”可能是提取并重新包装的用户态库。

GitHub 上可下载的驱动包不一定开源：打包脚本和适配库开放，并不代表其中的厂商驱动源码开放。

#### 3.2.2 Arm/Mali 驱动

Android 的 Mali GLES/Vulkan 通常由设备厂商集成。它与设备内核、缓冲区和显示接口配套；换用 Linux 开源实现需要额外的平台适配。

### 3.3 开源硬件驱动

以下项目处于同一用户态驱动层，按目标 GPU 与 API 区分。

#### 3.3.1 Freedreno

Mesa 的 **Adreno OpenGL/OpenGL ES** 驱动；广义名称也指 Adreno 开源生态。典型 Linux 路径为 OpenGL → Freedreno → msm DRM → Adreno。

它直接面向硬件生成命令，与 Zink 将 GL 转到 Vulkan 的路线不同。

官方：[Freedreno](https://docs.mesa3d.org/drivers/freedreno.html)。

#### 3.3.2 Turnip

Mesa 的 **Adreno Vulkan** 驱动，与 Freedreno 共享部分底层组件，但分别实现不同 API。

Android 模拟器常用针对 KGSL 与应用加载机制编译的 Turnip 包。传统 OpenGL 应用通常需要 Zink 等上层实现与它组合。

支持范围应查看具体源码设备表和发布说明，不能把某代部分型号支持推广到整代 GPU。

官方：[Turnip 文档](https://docs.mesa3d.org/drivers/freedreno.html#turnip)。

#### 3.3.3 Lima

面向旧 **Mali Utgard**，如 Mali-400/450，主要提供 GLES 2.0 与一定桌面 GL 能力。不是现代 Mali-G 系列或现代 Vulkan 的适配方案。

官方：[Lima](https://docs.mesa3d.org/drivers/lima.html)。

#### 3.3.4 Panfrost

面向支持的 **Mali OpenGL/OpenGL ES** 硬件实现。具体型号、API 水平和认证状态见官方表；Linux 运行还需对应内核驱动和显示适配。

官方：[Panfrost](https://docs.mesa3d.org/drivers/panfrost.html)。

#### 3.3.5 PanVK

Panfrost 栈中的 **Mali Vulkan** 实现。与 Turnip 职责相近，但目标硬件不同；不能拿 Turnip 包替代 PanVK。

源码支持、规范认证和 Android 手机上可安装，是不同条件。

官方：[PanVK](https://docs.mesa3d.org/drivers/panfrost.html)。

### 3.4 CPU 软件实现

这些实现用 CPU 完成主要图形计算，是硬件渲染的替代路径。

| 项目 | 所属项目 | 主要 API/特点 |
| --- | --- | --- |
| llvmpipe | Mesa | OpenGL/GLES；LLVM JIT、多线程与向量化软件渲染 |
| softpipe | Mesa | OpenGL/GLES 软件实现，常用于参考与回退 |
| Lavapipe | Mesa | Vulkan 软件实现 |
| SwiftShader | Google 独立项目 | 以 CPU 为主的 Vulkan 实现 |

软件渲染适合测试、故障排查与轻量场景。API 能力较完整不代表游戏流畅；出现 `llvmpipe` 一般说明主要 3D 计算未走预期 GPU 加速。

参考：[llvmpipe](https://docs.mesa3d.org/drivers/llvmpipe.html)、[Mesa 源码](https://gitlab.freedesktop.org/mesa/mesa)、[SwiftShader](https://github.com/google/swiftshader)。

## 4. 图形 API 与平台接口

### 4.1 图形 API

| API | 定位 | 与 Android 适配的关系 |
| --- | --- | --- |
| OpenGL | 桌面图形接口 | Minecraft Java 的传统绘制路线；常需兼容层 |
| OpenGL ES（GLES） | 移动/嵌入式图形接口 | Android 原生图形的重要基础 |
| Vulkan | 显式图形与计算接口 | Turnip、Zink 和许多模拟器使用的宿主接口 |
| Direct3D | Windows 图形接口 | Android Windows 兼容环境常借助转换层 |

**OpenGL 与 GLES 不是仅版本号不同的同一接口。**同样，Vulkan 版本号不能完整表达特性、扩展、格式和限制值。

参考：[Khronos OpenGL](https://www.khronos.org/opengl/)、[OpenGL ES](https://www.khronos.org/opengles/)、[Vulkan](https://www.vulkan.org/)、[Android Vulkan 指南](https://developer.android.com/ndk/guides/graphics)。

### 4.2 上下文、加载与呈现接口

| 名称 | 职责 |
| --- | --- |
| EGL | 建立图形上下文并连接平台表面 |
| GLX | 连接 OpenGL 与 X Window |
| OSMesa | OpenGL 离屏渲染接口；名称本身不能证明采用 CPU |
| Vulkan Loader | 加载图形实现并分发调用 |
| WSI | Vulkan 与窗口、交换链和呈现系统的集成 |

它们属于平台接入，不能与 Adreno、Turnip 等硬件或驱动品牌混为一类。

### 4.3 着色器与中间表示

- **GLSL：**OpenGL/GLES 常用的着色器语言。
- **SPIR-V：**Vulkan 常见着色器输入，是中间表示，不是所有 GPU 直接执行的机器码。
- **NIR：**Mesa 内部用于优化和转换的中间表示。
- **GPU 机器码：**硬件驱动最终为特定 GPU 生成的指令。

首次进入场景卡顿可能来自着色器或管线编译；这与持续绘制性能不足应分开判断。

## 5. API 转换与兼容层

转换层处理应用 API 与平台 API 的差异，再交给下层驱动执行。**转换方向**是区分这些项目的关键。

### 5.1 桌面 OpenGL → OpenGL ES

#### 5.1.1 GL4ES / Holy GL4ES

GL4ES 上游侧重 OpenGL 2.1/1.5 到 GLES 2.0/1.1 的转换，包括固定管线模拟。Holy GL4ES 是 Minecraft 启动器生态中的定向变体。

通常仍由 GPU 绘制；复杂桌面 GL 扩展和着色器不一定能完整转换。

项目：[GL4ES](https://github.com/ptitSeb/gl4es)、[Holy GL4ES](https://github.com/FCL-Team/Holy-GL4ES)。

#### 5.1.2 NG-GL4ES / Krypton Wrapper

从 GL4ES 相关实现发展而来，扩展着色器处理与 OpenGL 能力。核查的 FCL 项目将 NG-GL4ES 称为 Krypton Wrapper。

它属于 GL→GLES 路线，兼容范围由具体构建决定。

项目：[NG-GL4ES](https://github.com/FCL-Team/NG-GL4ES)。

#### 5.1.3 LTW

Large Thin Wrapper，主要面向 Minecraft，将桌面 **OpenGL Core** 适配到 GLES。Core 路线能运行，不代表依赖旧固定管线的全部程序兼容。

项目：[LTW](https://github.com/MojoLauncher/LTW)。

#### 5.1.4 MobileGlues（MG）

面向 Minecraft Java 的 GL→GLES 实现，项目推荐 GLES 3.2，最低要求 GLES 3.0。现代模组与光影的支持需按版本测试。

通常使用系统 GLES 驱动，不必因选择 MG 就额外配 Turnip。

项目：[MobileGlues](https://github.com/MobileGL-Dev/MobileGlues)、[发布与插件](https://github.com/MobileGL-Dev/MobileGlues-release)。

### 5.2 桌面 OpenGL → Vulkan

#### 5.2.1 Zink

Mesa Gallium 驱动，将 OpenGL 实现到 Vulkan 上，下层可接 Turnip 或其他兼容 Vulkan 驱动。

**Zink 与 Turnip 经常同时使用：**前者提供 GL，后者执行 Vulkan。Zink 的能力受到自身实现与底层 Vulkan 特性共同限制。

官方：[Zink](https://docs.mesa3d.org/drivers/zink.html)。

### 5.3 OpenGL ES → 多种宿主 API

#### 5.3.1 ANGLE

对外提供 GLES/EGL，使用 Vulkan、Direct3D、Metal、OpenGL 等后端。它通常不能独自替代 Minecraft 所需的桌面 OpenGL 层。

可能组合为 GL 包装器 → GLES → ANGLE → Vulkan → 驱动；是否提供这条路径要看具体应用。

项目：[ANGLE](https://github.com/google/angle)。

### 5.4 Direct3D → 宿主图形 API

| 项目 | 主要转换方向 | 常见用途 |
| --- | --- | --- |
| DXVK | D3D8/9/10/11 → Vulkan | Wine/Windows 游戏兼容 |
| VKD3D-Proton | D3D12 → Vulkan | Proton 等 Windows 游戏兼容 |
| WineD3D | Direct3D → OpenGL 等后端 | Wine 图形实现 |

这些项目可以与 Turnip 组合，但不是传统 Minecraft Java 或 Switch 模拟器必需的图形层。

项目：[DXVK](https://github.com/doitsujin/dxvk)、[VKD3D-Proton](https://github.com/HansKristian-Work/vkd3d-proton)、[Wine](https://www.winehq.org/)。

## 6. 图形虚拟化与运行环境

图形虚拟化解决客体/客户端如何使用宿主图形能力，不能与 CPU 软件渲染混为一类。

### 6.1 VirGL

虚拟 3D GPU 路线：客户端/客体的图形工作通过 VirGL 交给宿主 virglrenderer，再由宿主后端执行。

Android 适配可使用本地 vtest/socket 转发，不一定启动完整虚拟机。它通常可利用宿主 GPU，但也可能接到软件后端。

官方：[VirGL](https://docs.mesa3d.org/drivers/virgl.html)。

### 6.2 Venus

用于 Vulkan 命令序列化与 Virtio-GPU 环境。它与 VirGL 的经典 GL 路线不同，需要客体设备、宿主渲染器及驱动满足相应条件。

官方：[Venus](https://docs.mesa3d.org/drivers/venus.html)。

### 6.3 Android Linux 环境的差异

| 环境 | 图形接入要点 |
| --- | --- |
| Android APK | 可调用系统 GLES/Vulkan；自定义驱动需应用主动支持 |
| Termux 原生 | 使用 Android/Bionic；还需适配图形库、设备权限与窗口系统 |
| proot/chroot 发行版 | 共用 Android 内核；glibc、内核接口与厂商库可能不匹配 |
| 完整虚拟机 | 独立客体内核；需要明确的虚拟 GPU、转发或直通机制 |
| 主线 Linux 开发板 | 需 GPU 内核驱动、固件、Mesa 与显示控制器共同适配 |

安装 Mesa、增加虚拟 CPU 核心或内存，都不会自动补上缺失的 GPU 接入通道。

## 7. Minecraft 启动器与 FCL 渲染器

### 7.1 应用层的职责

Minecraft Java 的传统路径通过 LWJGL 使用桌面 OpenGL。Android 启动器负责 JVM、原生库、窗口与输入接入，并提供合适的 GL 实现。

Sodium、Embeddium、Iris 等模组可能改变扩展和着色器需求。Minecraft 版本兼容，不等于所有整合包和光影兼容。

### 7.2 FCL 内置菜单

核查官方主分支提交 `25fb237d48dfa19ba27d7d74d43ba380ec94fa71`，内置以下六项；其后会追加插件。正式版与旧版菜单可能不同。

| 菜单项 | 实际技术路线 | 下层依赖 |
| --- | --- | --- |
| Krypton Wrapper / NG-GL4ES | GL→GLES 包装器 | 系统 GLES |
| Holy-GL4ES | GL→GLES 包装器变体 | 系统 GLES |
| VirGLRenderer | Mesa/VirGL 转发 | virglrenderer 与宿主后端 |
| VGPU | FCL 特定 GLES 兼容方案 | `libvgpu.so` 与系统 GLES |
| Zink | Mesa GL→Vulkan | 合适的 Vulkan 驱动 |
| Freedreno | Adreno 原生 GL 路线 | 支持硬件与匹配内核接入 |

相关技术原理见前文。**VGPU 的完整独立上游、内部实现与许可证，本次未充分确认**；不能仅凭名称认定为 VirGL、llvmpipe 或通用虚拟显卡。

官方：[FCL](https://github.com/FCL-Team/FoldCraftLauncher)、[核查版本的 RendererManager](https://github.com/FCL-Team/FoldCraftLauncher/blob/25fb237d48dfa19ba27d7d74d43ba380ec94fa71/FCL/src/main/java/com/mio/manager/RendererManager.kt)。

### 7.3 插件与相近方案

- **MobileGlues、LTW：**常见 GL→GLES 扩展路线。
- **Mesa/Zink 组合：**可能使用不同 Mesa、Vulkan 驱动和窗口桥接版本。
- **VirGL、软件渲染或实验组合：**是否可选由安装版、插件与第三方分支决定。

插件能不断增加，因此“全部渲染器”应以指定启动器版本和已安装插件为范围。

参考：[FCLRendererPlugin](https://github.com/ShirosakiMio/FCLRendererPlugin)。

### 7.4 三种扩展不要混用

| 扩展 | 改变的层级 | 例子 |
| --- | --- | --- |
| 渲染器插件 | 提供 GL 实现、包装器及环境配置 | MG、LTW、Zink 组合 |
| 驱动包/驱动插件 | 更底层的用户态 API 实现 | Turnip、Qualcomm Vulkan 包 |
| 游戏模组 | 改变 Minecraft 的绘制方式和需求 | Sodium、Iris、Vulkan 类模组 |

选了 Zink 以后仍要考虑底层 Vulkan 驱动；游戏改用 Vulkan 则是另一条应用路线。

## 8. Android Switch 模拟器与驱动包

### 8.1 渲染路径

模拟器把 Switch GPU 命令、着色器和资源行为转换成宿主图形操作，常使用 Vulkan 后端，再交给 Android GPU 驱动。

| 层级 | 工作 |
| --- | --- |
| 模拟器 GPU 后端 | 翻译客机图形行为，处理资源与着色器 |
| 宿主 Vulkan 驱动 | 编译和提交宿主 GPU 命令 |
| 内核驱动与硬件 | 管理执行、同步和故障恢复 |

CPU 模拟、GPU 翻译和宿主驱动是不同工作。换驱动主要改变宿主执行层。

项目参考：[Skyline](https://github.com/skyline-emu/skyline)、[Strato](https://github.com/strato-emu/strato)；仓库存在不代表当前仍持续维护。

### 8.2 驱动类别

#### 8.2.1 系统驱动

随固件提供，与设备接口配套。通常是所有测试的基线；Mali Android 设备常主要依赖这一路线。

#### 8.2.2 Turnip 自定义包

Android 可加载的 Mesa Turnip 构建，用于支持的 Adreno。可能修复系统驱动问题，也可能出现回归；应核对 GPU、Android API、KGSL 与模拟器加载支持。

#### 8.2.3 Qualcomm 自定义包

提取、修补依赖或包装的 Qualcomm 闭源用户态驱动。厂商版本编号不能与 Mesa 版本直接比较，也不能据此推断跨全部 Adreno 兼容。

发布参考：[AdrenoToolsDrivers](https://github.com/K11MCH1/AdrenoToolsDrivers)、[版本说明](https://github.com/K11MCH1/AdrenoToolsDrivers/releases)。这是分发项目，不是 Mesa 官方发布渠道。

### 8.3 libadrenotools 与无 root 加载

libadrenotools 是驱动加载/修改工具库，**不是 GPU 驱动本身**。支持它的应用可以在进程内加载另一套 Adreno 用户态实现。

这种替换通常只影响该应用，不会替换整机内核驱动或让其他应用自动使用新驱动。

项目：[libadrenotools](https://github.com/bylaws/libadrenotools)。

### 8.4 驱动 ZIP 与版本标签

典型包包含 `meta.json`、主驱动 `.so` 和依赖库。

| 字段/标签 | 含义 |
| --- | --- |
| `vendor`、`driverVersion` | 实现类别及驱动版本 |
| `minApi`、`libraryName` | 最低 Android API 与主库入口 |
| `packageVersion`、Revision/R | 发布者的打包修订；不同作者间不能直接比较 |
| Mesa 版本 | 上游基础版本，不是完整兼容保证 |
| Qualcomm vxxx | 厂商驱动版本标识 |
| beta/experimental | 实验支持，需逐设备验证 |
| GMEM/SYSMEM/autotuner | 分块/系统内存渲染策略及调优标记 |

格式参考：[ADPKG](https://github.com/bylaws/libadrenotools/blob/master/tools/ADPKG.md)。

### 8.5 纹理与着色器兼容

ASTC、BCn 等压缩格式，以及着色器编译、资源布局和同步行为，都会影响模拟器画面与性能。格式不匹配可能需要解码或转换，增加内存和计算开销。

花屏不一定由单一纹理格式造成；新驱动也不一定更快。应同时比较画面、崩溃、帧时间和持续运行表现。

## 9. 选择与排查

### 9.1 典型路径

| 场景 | 主要路径 |
| --- | --- |
| Android 原生 GLES 游戏 | 应用 → GLES → 系统驱动 → 内核 → GPU |
| FCL GL→GLES | Minecraft → 包装器 → GLES → 系统驱动 → GPU |
| FCL Zink + Turnip | Minecraft → OpenGL/Zink → Vulkan/Turnip → KGSL → Adreno |
| Linux Adreno GL | 应用 → OpenGL/Freedreno → msm DRM → Adreno |
| Switch 模拟器 | 客机 GPU 工作 → 模拟器 Vulkan 后端 → 宿主驱动 → GPU |
| 软件 GL | 应用 → llvmpipe → CPU → 显示呈现 |

调用方向为上层 → 底层，文档阅读顺序则是底层 → 上层。

### 9.2 测试起点

| 条件 | 可优先比较 |
| --- | --- |
| 旧 Minecraft、传统 GL 模组 | Holy GL4ES 与启动器默认方案 |
| 现代 Minecraft/渲染模组 | NG-GL4ES、MG、适配的 LTW |
| Adreno 上复杂 GL 光影 | 合适的 Zink + Turnip 与其他兼容路线 |
| Mali/Immortalis | GLES 包装器；满足条件时再测 Zink + 系统 Vulkan |
| Switch 模拟器 | 系统驱动基线，再测明确支持设备的自定义包 |

这是按技术定位给出的对照起点，不是设备实测排行榜。

### 9.3 检查项目

1. 记录 **GPU 型号、系统固件、应用、渲染器、底层驱动、游戏/模组版本**。
2. 固定场景、分辨率和散热条件，一次只改变一个变量。
3. 区分首次编译卡顿、持续低帧率和运行后降频。
4. 更换驱动后按应用说明处理相关着色器/管线缓存。
5. 从应用日志确认实际 renderer/driver，不只看菜单名称。

Linux 图形环境中可用 `glxinfo -B`、`vulkaninfo --summary`、`eglinfo` 辅助检查；结果只代表该工具自己的图形上下文。

### 9.4 关键判断

- **出现 llvmpipe：**主要 3D 工作通常在 CPU 上。
- **Zink 能加载：**还要检查底层 Vulkan 驱动与所需特性。
- **ZIP 能导入：**不等于驱动能初始化目标 GPU。
- **GL 版本覆写：**不会补齐实际缺失的功能。
- **调试变量：**如 `flushall`、`syncdraw` 可能用于定位问题，不宜当作通用性能优化。
- **开源或上游支持：**不等于每台 Android 同型号设备都可直接使用。

本文保留软件栈主干，不维护逐设备、逐游戏的兼容榜；具体支持范围请继续查对应条目后的官方文档与发布说明。
