# 第04章 VPSS（Video Process SubSystem）模块详解——AI Camera 的图像处理中心

> **如果说 VI 是整个 RKMPI 数据流的入口，那么 VPSS 就是整个 AI Camera Pipeline 的"图像处理中心"。**
>
> 几乎所有 AI 应用，在送入 RKNN 推理之前，都会经过 VPSS。因此，理解 VPSS，是学习 RKMPI 最重要的一步。

------

# 一、本章目标

完成本章学习后，你将能够理解：

- 什么是 VPSS？
- 为什么 AI 项目几乎都会使用 VPSS？
- VPSS 在整个 Pipeline 中处于什么位置？
- VPSS 能完成哪些图像处理工作？
- Group 和 Channel 分别是什么？
- 为什么 VPSS 是 RKNN 最好的前处理模块？

------

# 二、回顾整体 Pipeline

上一章，我们学习了 VI 模块。

目前的数据流如下：

```text
Camera Sensor
      │
MIPI CSI
      │
ISP
      │
VI
```

但是，这里的图像还不能直接送给 AI。

原因很简单。

假设摄像头输出：

```text
1920 × 1080
NV12
```

而 YOLO 模型要求输入：

```text
640 × 640
RGB888
```

两者完全不同。

因此，中间必须有一个图像处理模块。

它就是：

```text
VPSS
```

------

# 三、什么是 VPSS？

VPSS 全称：

```text
Video Process SubSystem
```

中文通常翻译为：

> **视频处理子系统**

它位于：

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
```

可以把 VPSS 理解成：

> **整个 AI Camera 的图像加工厂。**

所有进入 AI 的图像，

几乎都会先经过 VPSS。

------

# 四、为什么需要 VPSS？

举一个最典型的例子。

摄像头输出：

```text
1920 × 1080
NV12
```

YOLO 输入：

```text
640 × 640
RGB888
```

如果没有 VPSS。

程序只能：

```text
Camera

↓

CPU Resize

↓

CPU Color Convert

↓

CPU Crop

↓

RKNN
```

CPU 不断：

- memcpy
- Resize
- Convert

不仅速度慢，

还会增加大量内存拷贝。

而使用 VPSS：

```text
Camera

↓

VI

↓

VPSS

↓

RKNN
```

这些工作全部交给硬件完成。

CPU 基本不用参与。

------

# 五、VPSS 可以完成哪些工作？

VPSS 的主要作用就是：

**图像预处理。**

它支持：

| 功能          | 说明         |
| ------------- | ------------ |
| Resize        | 图像缩放     |
| Crop          | 图像裁剪     |
| Rotate        | 图像旋转     |
| Mirror        | 镜像         |
| Flip          | 翻转         |
| Color Convert | 颜色格式转换 |
| Frame Rate    | 帧率控制     |
| Multi Output  | 多路输出     |

可以看到：

几乎所有 AI 前处理都会使用 VPSS。

------

# 六、为什么 VPSS 是 AI 的最佳搭档？

AI 模型通常都有固定输入尺寸。

例如：

| 模型      | 输入尺寸 |
| --------- | -------- |
| YOLOv5    | 640×640  |
| YOLOv8    | 640×640  |
| MobileNet | 224×224  |
| ResNet    | 224×224  |

而 Camera：

可能输出：

```text
1920×1080

2560×1440

3840×2160
```

如果全部交给 CPU：

每一帧都要：

```text
Resize

↓

Color Convert

↓

Copy
```

CPU 会非常忙。

而 VPSS：

专门负责：

```text
1920×1080

↓

640×640
```

速度远远快于 CPU。

------

# 七、VPSS 在 Pipeline 中的位置

整个数据流如下：

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
```

很多同学会问：

为什么不能：

```text
Camera

↓

VI

↓

RKNN
```

答案很简单。

RKNN 并不负责：

- Resize
- Crop
- Rotate

RKNN 负责的是：

> **神经网络推理。**

因此：

图像必须先处理好，

才能送入 RKNN。

------

# 八、VPSS 的核心设计——Group 与 Channel

VPSS 和 VI 一样，

采用：

```text
Group

↓

Channel
```

架构。

很多初学者第一次看到都会迷惑。

其实很好理解。

------

## Group

Group 可以理解为：

**一个图像处理流水线。**

例如：

```text
VI

↓

VPSS Group0
```

所有进入 Group 的图像，

都会按照同样方式处理。

------

## Channel

一个 Group 可以拥有多个 Channel。

例如：

```text
VPSS Group0

├────────────┐
│            │
▼            ▼
Ch0        Ch1
```

为什么？

因为：

同一张图，

可能有不同用途。

例如：

```text
1920×1080

↓

VPSS
```

输出：

```text
Channel0

↓

640×640

↓

RKNN
```

同时：

```text
Channel1

↓

1920×1080

↓

VENC
```

这样：

AI 使用：

640×640。

录像：

仍然保持：

1080P。

互不影响。

------

# 九、VPSS 多路输出

这是 VPSS 最大的优势。

例如：

```text
Camera

↓

VI

↓

VPSS Group0

├────────────┬──────────────┐
│            │              │
▼            ▼              ▼
640×640   1280×720     1920×1080
│            │              │
RKNN       LCD         H264
```

同一份数据。

一次输入。

多个输出。

全部硬件完成。

CPU 不需要重复处理。

------

# 十、VPSS 常用 API

后续 Demo 会大量使用以下接口。

------

创建 Group

```cpp
RK_MPI_VPSS_CreateGrp()
```

作用：

创建 VPSS 图像处理组。

------

启动 Group

```cpp
RK_MPI_VPSS_StartGrp()
```

开始处理图像。

------

创建 Channel

```cpp
RK_MPI_VPSS_EnableChn()
```

创建输出通道。

------

获取图像

```cpp
RK_MPI_VPSS_GetChnFrame()
```

获取处理后的图像。

------

释放图像

```cpp
RK_MPI_VPSS_ReleaseChnFrame()
```

释放 Frame。

------

停止 Group

```cpp
RK_MPI_VPSS_StopGrp()
```

停止处理。

------

销毁 Group

```cpp
RK_MPI_VPSS_DestroyGrp()
```

释放资源。

------

# 十一、VPSS 与 OpenCV 的区别

很多人都会问：

为什么不用：

```cpp
cv::resize()
```

答案如下。

OpenCV：

```text
Camera

↓

CPU Resize

↓

CPU Convert

↓

RKNN
```

全部：

CPU 运算。

而 VPSS：

```text
Camera

↓

VI

↓

VPSS

↓

RKNN
```

全部：

硬件处理。

因此：

- 更快
- 功耗更低
- 几乎没有 CPU 占用

对于嵌入式 SoC 来说，

这是最大的优势。

------

# 十二、VPSS 与 RKNN 的关系

很多同学认为：

RKNN 自己会完成前处理。

实际上：

推荐的数据流是：

```text
Camera

↓

VI

↓

VPSS

↓

RKNN
```

原因：

VPSS：

负责：

- Resize
- Crop
- Format

RKNN：

负责：

- AI 推理

职责非常清晰。

这样：

RKNN 可以专心做神经网络计算。

------

# 十三、VPSS 在整个 AI Camera 中的位置

整个系统可以理解为：

```text
                 Camera
                    │
                   ISP
                    │
                    VI
                    │
             VPSS(图像处理中心)
      ┌─────────┼──────────┐
      │         │          │
      ▼         ▼          ▼
    RKNN      VENC        VO
      │
     YOLO
```

整个 Pipeline 中：

只有 VPSS：

真正负责图像处理。

因此：

它也是整个媒体框架中最重要的模块之一。

------

# 十四、本章总结

本章主要介绍了 VPSS 的整体设计思想。

需要牢记以下几点：

**① VPSS 是 RKMPI 的图像处理中心，负责 AI 前处理和视频预处理。**

**② VPSS 可以完成 Resize、Crop、Rotate、Color Convert 等常见图像处理任务，全部由硬件完成。**

**③ VPSS 采用 Group + Channel 架构，一个 Group 可以输出多个不同规格的视频流。**

**④ AI 推理通常不直接处理摄像头原始输出，而是先经过 VPSS，再送入 RKNN。**

**⑤ VPSS 的意义不仅是缩放图像，更重要的是利用 SoC 专用硬件完成高性能、低功耗的数据处理，为后续 RKNN、VENC、VO 等模块提供符合要求的数据流。**

------

# 下一章预告

下一章我们将学习整个 RKMPI 中最容易被忽略、但也是理解零拷贝（Zero Copy）的关键内容：

> **第05章 MB Buffer 与 Zero Copy——为什么 RKMPI 可以做到高性能？**

届时将深入分析：

- 什么是 MB（Media Buffer）？
- 什么是 MB_BLK？
- DMA Buffer 与普通内存有什么区别？
- 为什么 `RK_MPI_SYS_Bind()` 可以实现零拷贝？
- CPU、Cache、DMA 三者之间是如何协同工作的？

理解这一章后，你就会真正明白 **RKMPI 为什么比传统 V4L2 + OpenCV 的方式效率更高**，也为后续学习 `SYS Bind` 和 `RKNN` 打下最重要的基础。