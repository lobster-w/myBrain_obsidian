# 第02章 RKMPI整体架构——一张图看懂Rockchip AI Camera Pipeline

> **本章是整个 RKMPI 系列最重要的一章。**
>
> 从这一章开始，我们不再关注某一个 API，而是先建立整个 AI Camera 的系统架构。后续学习的 VI、VPSS、RKNN、VENC 等模块，都将在这张架构图中找到自己的位置。

------

# 一、本章目标

完成本章学习后，你将能够理解：

- RKMPI 是如何组织整个 AI Camera Pipeline 的？
- Camera 图像是如何一步一步流向 AI 模型的？
- VI、VPSS、RKNN、VENC 分别负责什么？
- 什么是 Media Pipeline？
- 为什么官方 Demo 都采用模块化设计？

------

# 二、从一张图开始

现代 AI 摄像头并不仅仅是"获取图像"。

完整的数据流通常如下：

```text
                    Camera Sensor
                          │
                    MIPI CSI-2
                          │
                    ISP(Image Signal Processor)
                          │
                ┌─────────┴─────────┐
                │                   │
             Video Input（VI）      Metadata
                │
         Video Process SubSystem（VPSS）
                │
        ┌───────┼────────┬──────────┐
        │       │        │          │
        ▼       ▼        ▼          ▼
      RKNN     VENC      VO        RGA
      │         │        │          │
      ▼         ▼        ▼          ▼
    YOLO     H264/H265   LCD      图像处理
      │
      ▼
  检测结果（Boxes）
      │
      ▼
     OSD
      │
      ▼
 最终视频输出/RTSP
```

可以看到：

摄像头获取图像只是整个系统的第一步。

真正的 AI Camera 是由多个硬件模块共同完成的。

------

# 三、Media Pipeline（媒体处理流水线）

RKMPI 的核心思想就是：

> **Media Pipeline（媒体处理流水线）**

所谓 Pipeline，就是：

上一模块的输出，直接作为下一模块的输入。

例如：

```text
Camera
   │
   ▼
VI
   │
   ▼
VPSS
   │
   ▼
RKNN
   │
   ▼
YOLO
```

整个过程中：

图像一直沿着流水线流动。

开发者只需要负责：

- 创建模块
- 配置参数
- 建立连接

而不需要自己管理每一次图像拷贝。

------

# 四、为什么叫"模块化"？

观察官方 Demo 可以发现：

每一个功能几乎都是一个独立模块。

例如：

```text
VI

VPSS

VENC

VO

RGA

RKNN
```

为什么这样设计？

原因非常简单。

不同产品，对流水线的需求完全不同。

例如：

## 场景一：AI 检测

```text
Camera
   │
VI
   │
VPSS
   │
RKNN
```

不需要编码。

------

## 场景二：RTSP 推流

```text
Camera
   │
VI
   │
VPSS
   │
VENC
   │
RTSP
```

不需要 AI。

------

## 场景三：AI + 推流

```text
Camera
   │
VI
   │
VPSS
   ├──────────────┐
   │              │
   ▼              ▼
 RKNN           VENC
   │              │
 YOLO           RTSP
```

一份图像同时提供给 AI 和编码器。

这就是模块化设计最大的优势。

------

# 五、各个模块负责什么？

下面介绍整个 Pipeline 中最重要的几个模块。

------

## 1、ISP（Image Signal Processor）

ISP 位于 Camera 后面。

主要负责：

- 自动曝光（AE）
- 自动白平衡（AWB）
- 去噪（NR）
- Demosaic
- Gamma
- CCM

输入：

```text
RAW Bayer
```

输出：

```text
YUV
RGB
```

如果没有 ISP，图像通常会出现：

- 偏绿
- 偏暗
- Bayer 格式

这也是我们在 V4L2 系列中看到的现象。

------

## 2、VI（Video Input）

VI：

Video Input。

它负责：

**接收 ISP 输出的视频数据。**

需要注意：

VI 并不是 Camera。

Camera 前面还有：

```text
Sensor
    │
CSI
    │
ISP
```

真正进入 RKMPI 的第一站，就是 VI。

------

## 3、VPSS（Video Process SubSystem）

VPSS 可以理解为：

图像处理中心。

主要负责：

- Resize
- Crop
- Rotate
- Mirror
- Color Convert

例如：

摄像头输出：

```text
1920×1080
```

YOLO 输入：

```text
640×640
```

VPSS 可以直接完成：

```text
1920×1080
      │
      ▼
640×640
```

整个过程无需 CPU 参与。

------

## 4、RKNN

RKNN：

Rockchip NPU 推理模块。

负责：

- 加载模型
- 神经网络推理
- 输出检测结果

例如：

```text
YOLOv5

YOLOv8

MobileNet

ResNet
```

最终输出：

- Box
- Score
- Class

------

## 5、VENC（Video Encoder）

VENC：

硬件视频编码器。

负责：

```text
YUV
   │
   ▼
H264

H265

JPEG
```

它最大的优势就是：

无需 CPU 软件编码。

因此：

RTSP 推流几乎都会使用 VENC。

------

## 6、VO（Video Output）

VO：

Video Output。

负责：

- HDMI
- LCD
- MIPI Display

把图像显示到屏幕。

------

## 7、RGA（Raster Graphic Accelerator）

RGA：

图像加速模块。

主要负责：

- 图像缩放
- 图像旋转
- 图像拼接
- Alpha 混合

很多 OpenCV 操作都可以交给 RGA 完成。

------

# 六、整个数据流是怎样工作的？

下面是一个典型的 AI Camera 数据流。

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
    ├──────────────┐
    │              │
    ▼              ▼
 RKNN           VENC
    │              │
 YOLO         H264编码
    │              │
    ▼              ▼
  OSD         RTSP推流
```

这里有两个非常重要的特点。

第一：

图像可以同时流向多个模块。

第二：

整个过程几乎都是硬件模块完成。

CPU 主要负责：

- 初始化
- 参数配置
- 结果处理

而不是图像搬运。

------

# 七、为什么 RKMPI 性能更高？

如果使用普通方式：

```text
Camera
    │
V4L2
    │
CPU
    │
Resize
    │
CPU
    │
RKNN
```

CPU 会不断：

- memcpy
- Resize
- Convert

效率较低。

而 RKMPI：

```text
Camera
    │
VI
    │
VPSS
    │
RKNN
```

整个过程：

- DMA Buffer
- Zero Copy
- Hardware Pipeline

CPU 基本不参与图像处理。

因此：

性能更高。

功耗更低。

------

# 八、本系列后续学习路线

理解整个架构以后。

后续课程将按照 Pipeline 顺序展开。

```text
第03章  VI(Video Input)

        ↓

第04章  VPSS

        ↓

第05章  MB Buffer

        ↓

第06章  SYS Bind

        ↓

第07章  RKMPI取图

        ↓

第08章  RKNN

        ↓

第09章  YOLO

        ↓

第10章  OSD

        ↓

第11章  VENC

        ↓

第12章  RTSP
```

最终完成：

完整 AI Camera Pipeline。

------

# 九、本章总结

本章重点建立了整个 RKMPI 的系统架构。

需要牢记以下几点：

**① RKMPI 并不是简单的取图接口，而是一套完整的媒体处理框架。**

**② 整个系统采用 Pipeline（流水线）设计，每个模块负责单一功能。**

**③ 图像可以在多个模块之间同时流转，实现 AI、编码、显示等多种功能并行处理。**

**④ CPU 主要负责控制和配置，真正的数据处理尽可能由专用硬件完成，这也是 RKMPI 高性能的根本原因。**

下一章，我们将正式进入 **VI（Video Input）模块**，学习 RKMPI 中图像进入媒体流水线的第一站，并编写第一个基于 RKMPI 的图像采集程序。