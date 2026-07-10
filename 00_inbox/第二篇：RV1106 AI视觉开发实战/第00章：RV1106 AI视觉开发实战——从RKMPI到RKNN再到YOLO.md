# 《RV1106 AI视觉开发实战》第一章：为什么还要学习 RKMPI？—— 从 V4L2 到 AI Camera Pipeline

> 本系列基于 Luckfox RV1106 平台，逐步实现 **RKMPI + RKNN + YOLO** 的完整 AI 视觉开发流程。

------

# 一、本章目标

完成本章学习后，你将能够回答以下几个问题：

- 什么是 RKMPI？
- 为什么已经学习了 V4L2，还需要学习 RKMPI？
- V4L2 和 RKMPI 是什么关系？
- RKMPI 是否替代了 V4L2？
- 为什么 Rockchip 官方 AI Demo 几乎全部使用 RKMPI？
- 本系列课程最终要实现什么？

------

# 二、课程回顾

在上一套 **Linux Camera Framework** 系列课程中，我们已经完成了 V4L2 的学习。

主要内容包括：

- Camera 基本工作流程
- V4L2 Framework
- Video Device
- Buffer Queue
- mmap 零拷贝
- StreamOn / StreamOff
- RAW Bayer 图像
- ISP 基础知识
- 编写自己的 V4L2 采图程序

最终，我们已经能够使用 V4L2 从摄像头采集 RAW 图像，并理解 Linux Camera Framework 的基本工作原理。

数据流如下：

```text
Camera
    │
MIPI CSI
    │
V4L2
    │
Application
```

对于 Linux Camera 的学习来说，这已经是一个非常重要的里程碑。

------

# 三、为什么还要学习 RKMPI？

很多同学都会产生一个疑问：

> **既然已经能够使用 V4L2 获取图像，为什么官方 Demo 却全部使用 RKMPI？**

这是因为：

**V4L2 与 RKMPI 面向的是两个不同层次的问题。**

V4L2 解决的是：

> **如何从 Linux 获取摄像头数据。**

而 RKMPI 解决的是：

> **如何构建完整的多媒体处理流水线（Media Pipeline）。**

举一个最简单的例子。

如果仅使用 V4L2，一个 AI 程序通常需要自己完成：

```text
Camera
    │
V4L2
    │
CPU
    │
Resize
    │
Color Convert
    │
RKNN
```

CPU 需要负责大量图像处理工作，例如：

- 图像缩放
- 格式转换
- 数据拷贝
- Buffer 管理

随着分辨率越来越高，这种方式 CPU 开销会越来越大。

而使用 RKMPI 后，数据流变成：

```text
Camera
    │
VI
    │
VPSS
    │
RKNN
```

图像缩放、颜色转换等工作由硬件模块完成，大量减少 CPU 参与，提高整体性能。

------

# 四、什么是 RKMPI？

RKMPI 的全称是：

**Rockchip Media Process Interface**

它是 Rockchip 提供的一套媒体开发框架，用于统一管理 SoC 内部各种多媒体硬件。

需要特别说明的是：

**RKMPI 并不是 Linux 标准接口。**

它属于 Rockchip SDK 的组成部分，仅适用于 Rockchip 平台。

但是：

它背后的设计思想——**Media Pipeline（媒体处理流水线）**，却广泛存在于各种视觉 SoC 中。

例如：

- Rockchip RV1106
- Horizon RDK X5
- NVIDIA Jetson
- 海思 HiSilicon
- TI Vision SoC

虽然 API 不同，但整体数据流基本一致。

------

# 五、V4L2 与 RKMPI 的关系

很多初学者误认为：

```text
RKMPI
    ↓
替代
    ↓
V4L2
```

事实上，这种理解并不准确。

更合理的理解应该是：

```text
Linux Kernel

        │

    V4L2 Framework

        │

────────────────────────

Rockchip SDK

        │

      RKMPI

        │

Application
```

V4L2 仍然是 Linux Camera Framework 的基础。

而 RKMPI 则是在此基础上，对 Rockchip 各种硬件模块进行了统一封装。

因此：

**RKMPI 不是替代 V4L2，而是站在 V4L2 之上的媒体开发框架。**

------

# 六、为什么 AI 项目几乎都使用 RKMPI？

现代 AI Camera 并不仅仅需要"取图"。

完整的数据流通常如下：

```text
Camera
    │
ISP
    │
VI
    │
VPSS
    │
RKNN
    │
YOLO
    │
OSD
    │
VENC
    │
RTSP
```

整个过程中涉及：

- Camera
- ISP
- 图像缩放
- 图像裁剪
- 格式转换
- AI 推理
- 图像叠加
- H.264 编码
- 网络推流

如果全部由 CPU 完成，性能会非常差。

因此 Rockchip 提供了：

- VI（Video Input）
- VPSS（Video Process SubSystem）
- VENC（Video Encoder）
- VO（Video Output）

这些硬件模块，并通过 RKMPI 统一管理。

这也是官方 AI Demo 全部采用 RKMPI 的原因。

------

# 七、本系列课程最终实现什么？

本系列不会停留在 API 的调用。

我们的目标是完成一套真正的 AI Camera Pipeline。

最终效果如下：

```text
Camera
    │
MIPI CSI
    │
ISP
    │
VI
    │
VPSS
    ├────────────┐
    │            │
    ▼            ▼
 RKNN        VENC(H.264)
    │            │
 YOLO         RTSP
    │
 OSD
    │
最终视频输出
```

整个课程将逐步完成：

- RKMPI 图像采集
- RKNN 模型加载
- YOLO 推理
- 目标框绘制
- H.264 硬件编码
- RTSP 网络推流

最终搭建一套完整的 AI Camera 系统。

------

# 八、本系列课程安排

整个系列计划如下：

| 章节   | 内容                   |
| ------ | ---------------------- |
| 第01章 | 为什么学习 RKMPI       |
| 第02章 | RKMPI 整体架构         |
| 第03章 | VI 模块详解            |
| 第04章 | VPSS 模块详解          |
| 第05章 | MB Buffer 与 Zero Copy |
| 第06章 | SYS Bind 机制          |
| 第07章 | 第一个 RKMPI 取图程序  |
| 第08章 | RKMPI 替代 V4L2        |
| 第09章 | RKNN 模型加载          |
| 第10章 | YOLO 推理              |
| 第11章 | YOLO 前处理            |
| 第12章 | YOLO 后处理            |
| 第13章 | OSD 绘制               |
| 第14章 | VENC 硬件编码          |
| 第15章 | RTSP 推流              |
| 第16章 | 完整 AI Camera 项目    |

课程将坚持以下原则：

- 从零开始，不直接照搬官方 Demo。
- 每一章都包含完整 Demo。
- 每一章都分析底层设计思想。
- 每一章都能够独立运行验证。

------

# 九、本章总结

本章主要建立了整个系列的学习目标。

需要牢记以下几点：

1. **V4L2 与 RKMPI 并不是竞争关系。**
2. **V4L2 负责 Linux Camera Framework，而 RKMPI 负责 Rockchip Media Pipeline。**
3. **RKMPI 将多个硬件模块统一组织起来，实现高性能、多媒体处理。**
4. **AI Camera 的核心不仅是获取图像，而是建立完整的数据处理流水线。**

从下一章开始，我们将正式进入 **RKMPI 的整体架构**，分析 VI、VPSS、MB、VENC 等模块之间的关系，为后续 RKNN 与 YOLO 推理打下基础。