# 第03章 VI（Video Input）模块详解——RKMPI 数据流的第一站

> **从本章开始，我们正式进入 RKMPI 的代码世界。**
>
> 在上一章中，我们已经了解了整个 AI Camera Pipeline 的整体架构。本章将重点学习 **VI（Video Input）模块**，理解它在整个数据流中的作用，以及为什么所有图像都会首先进入 VI。

------

# 一、本章目标

完成本章学习后，你将能够理解：

- 什么是 VI（Video Input）？
- VI 在整个 RKMPI 中处于什么位置？
- 为什么所有图像都会先进入 VI？
- VI 与 V4L2 有什么关系？
- VI 的主要 API 有哪些？
- 如何创建一个 VI 通道（Channel）？

> **注意：本章重点是理解 VI 的设计思想，不急于编写完整 Demo。下一章开始再逐步构建完整的取图程序。**

------

# 二、回顾整体架构

上一章，我们介绍了整个 Rockchip AI Camera Pipeline。

```text
                 Camera Sensor
                       │
                 MIPI CSI-2
                       │
                      ISP
                       │
                 Video Input(VI)
                       │
                     VPSS
              ┌────────┴────────┐
              │                 │
            RKNN              VENC
              │                 │
            YOLO              RTSP
```

可以看到：

**VI 是整个 RKMPI 的入口。**

无论后续是：

- AI 推理
- 视频编码
- 视频显示
- 图片抓拍

图像都会首先进入 VI。

因此：

> **VI 可以理解为整个媒体流水线的数据入口。**

------

# 三、什么是 VI？

VI，全称：

```text
Video Input
```

中文通常翻译为：

> **视频输入模块**

很多初学者看到这个名字，很容易理解成：

> "VI 就是摄像头。"

实际上，这是错误的。

VI 并不是 Camera。

它只是：

**负责接收 Camera 输出的视频流。**

真正的数据来源仍然是：

```text
Camera Sensor
      │
MIPI CSI
      │
ISP
      │
VI
```

因此：

VI 更准确的职责应该描述为：

> **负责接收 ISP 输出的数据，并将视频帧送入整个媒体流水线。**

------

# 四、为什么需要 VI？

有人可能会问：

> ISP 已经输出图像了，为什么不能直接送给 VPSS？

答案是：

因为整个 RKMPI 是模块化架构。

所有图像必须有统一的数据入口。

因此：

Rockchip 在 ISP 后增加了：

```text
Video Input（VI）
```

以后：

无论来自：

- Camera0
- Camera1
- USB Camera
- 虚拟视频源

最终都会统一进入：

```text
VI
```

这样：

后面的 VPSS、VENC、RKNN 根本不用关心：

图像来自哪里。

它们只需要：

> 从 VI 获取视频流。

这就是模块化设计最大的优势。

------

# 五、VI 在整个 Pipeline 中的位置

如果按照数据流来看：

```text
RAW

↓

Sensor

↓

CSI

↓

ISP

↓

VI

↓

VPSS

↓

RKNN
```

VI 已经位于：

ISP 后面。

因此：

VI 接收到的数据：

通常已经不是 RAW Bayer。

而是：

```text
NV12

YUV420

RGB（部分平台）
```

所以：

以后我们通过：

```cpp
RK_MPI_VI_GetChnFrame()
```

获得的数据，

通常已经可以直接用于后续处理。

------

# 六、VI 的基本概念

学习 VI，需要先理解几个概念。

------

## 1、Device（设备）

一个 Camera，

通常对应一个 VI Device。

例如：

```text
Camera0

↓

VI Device0
```

如果有双摄：

```text
Camera0

↓

Device0


Camera1

↓

Device1
```

Device 代表：

物理输入设备。

------

## 2、Channel（通道）

Device 可以拥有多个 Channel。

例如：

```text
Camera

↓

VI Device

├─────────────┐
│             │
▼             ▼
Ch0         Ch1
```

为什么？

因为：

同一个 Camera，

可能需要：

一路：

```text
1920×1080
```

用于录像。

另一路：

```text
640×640
```

用于 AI。

因此：

VI 支持多个输出通道。

------

# 七、为什么设计成 Device + Channel？

这是很多媒体框架共同采用的设计。

例如：

```text
Camera

↓

Device

↓

多个Channel
```

这样可以：

- 不同分辨率
- 不同帧率
- 不同用途

共享同一个 Camera。

而不是：

每一路都重新打开摄像头。

这种设计：

不仅减少资源占用，

还能方便后续扩展。

------

# 八、VI 常用 API

后续我们会频繁接触以下几个接口。

## 创建 VI 通道

```cpp
RK_MPI_VI_CreateChn()
```

作用：

创建一个 VI Channel。

------

## 启动 VI

```cpp
RK_MPI_VI_EnableChn()
```

作用：

开始接收图像。

------

## 获取视频帧

```cpp
RK_MPI_VI_GetChnFrame()
```

作用：

获取一帧图像。

这一点非常类似：

V4L2：

```cpp
VIDIOC_DQBUF
```

但是：

RKMPI 已经帮我们封装好了。

------

## 释放图像

```cpp
RK_MPI_VI_ReleaseChnFrame()
```

作用：

通知系统：

这一帧已经使用完成。

可以继续循环使用。

------

## 停止 VI

```cpp
RK_MPI_VI_DisableChn()
```

停止视频输入。

------

## 删除 Channel

```cpp
RK_MPI_VI_DestroyChn()
```

释放资源。

------

# 九、VI 与 V4L2 有什么关系？

很多同学都会问：

> **学习 VI 后，是不是以后就不用 V4L2 了？**

答案是否定的。

关系如下：

```text
Linux Kernel

V4L2 Driver

───────────────

Rockchip SDK

VI

───────────────

Application
```

可以理解为：

V4L2：

负责：

驱动 Camera。

VI：

负责：

管理视频流。

因此：

VI 建立在 Linux Camera Framework 之上。

而不是替代它。

------

# 十、VI 模块完成了哪些工作？

很多人认为：

VI 就是：

"取图。"

实际上：

VI 做了很多事情。

包括：

- 建立视频输入通道
- 接收 ISP 输出
- 管理视频 Buffer
- 向后级模块提供统一接口
- 与 VPSS 建立连接

所以：

VI 更像：

整个 Media Pipeline 的入口。

而不仅仅是：

一个 GetFrame()。

------

# 十一、后续的数据流

当 VI 创建完成以后。

数据流如下：

```text
Camera

↓

ISP

↓

VI

↓

Frame Buffer
```

下一章。

我们将加入：

```text
VPSS
```

整个流程变成：

```text
Camera

↓

ISP

↓

VI

↓

VPSS

↓

CPU
```

开始真正完成：

- Resize
- Crop
- Color Convert

并为 RKNN 输入做好准备。

------

# 十二、本章总结

本章重点理解了 VI 在整个媒体框架中的定位。

需要牢记以下几点：

**① VI（Video Input）是 RKMPI 的数据入口，而不是 Camera 本身。**

**② VI 位于 ISP 后面，负责接收经过 ISP 处理后的图像数据。**

**③ VI 采用 Device + Channel 的设计，使一个摄像头可以同时输出多个视频流。**

**④ VI 提供统一的视频输入接口，为 VPSS、RKNN、VENC 等后续模块提供数据来源。**

**⑤ 学习 VI，不只是学习几个 API，更重要的是理解它在整个 AI Camera Pipeline 中承担的职责。**

------

# 下一章预告

下一章我们将学习：

> **第04章 VPSS（Video Process SubSystem）模块详解**

届时将重点分析：

- 为什么 AI 推理几乎都离不开 VPSS？
- VPSS 如何完成 Resize、Crop、Rotate、Color Convert？
- 为什么 VPSS 是整个 AI Camera Pipeline 中最重要的图像处理模块？
- 第一个完整的 **VI → VPSS** 数据流将正式搭建完成，为后续 RKNN 与 YOLO 推理做好准备。