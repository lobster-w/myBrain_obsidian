# 第01章 为什么学习 RKMPI？—— 从 V4L2 到 AI Camera Pipeline

> **本系列基于 Luckfox RV1106 平台，系统学习 RKMPI、RKNN 与 YOLO，最终搭建完整的 AI Camera Pipeline。**

------

# 一、本章目标

完成本章学习后，你将能够理解：

- 为什么已经学习了 V4L2，还需要学习 RKMPI？
- V4L2 能完成什么？不能完成什么？
- 什么是 RKMPI？
- 为什么 Rockchip 官方 AI Demo 几乎全部基于 RKMPI？
- 本系列课程的整体学习路线是什么？

------

# 二、课程回顾

上一篇系列课程，我们已经完成了 **Linux Camera Framework** 的学习。

主要内容包括：

- Camera 工作原理
- V4L2 Framework
- Video Device
- Buffer Queue
- mmap 零拷贝
- StreamOn / StreamOff
- RAW Bayer 图像
- ISP 基础知识
- 编写自己的 V4L2 采图程序

最终，我们已经能够使用 V4L2 从摄像头采集图像。

整个流程如下：

```text
Camera Sensor
      │
 MIPI CSI
      │
 V4L2 Driver
      │
Application
```

到这里，我们已经能够完成：

✔ 打开摄像头

✔ 获取视频帧

✔ 保存图像

✔ 理解 Linux Camera Framework

那么问题来了。

------

# 三、一个疑问：为什么官方 Demo 几乎不用 V4L2？

当我们查看 Luckfox 官方 SDK 时，会发现一个现象。

无论是：

- AI Demo
- YOLO Demo
- RTSP Demo
- 视频编码 Demo

几乎全部使用：

```
RKMPI
```

而不是：

```
V4L2
```

很多初学者都会产生疑问：

> **是不是学习 V4L2 已经没有意义了？**

答案当然不是。

事实上：

**V4L2 与 RKMPI 根本不是竞争关系。**

它们解决的是两个完全不同层次的问题。

------

# 四、V4L2 到底解决了什么？

V4L2（Video for Linux 2）是 Linux 官方提供的视频设备框架。

它主要解决的问题只有一个：

> **如何从 Linux 获取视频数据。**

例如：

```c
open()

VIDIOC_REQBUFS()

VIDIOC_STREAMON()

VIDIOC_DQBUF()
```

这些 API 最终完成的事情就是：

```
Camera
    │
V4L2
    │
Application
```

因此：

V4L2 更关注的是：

- 摄像头驱动
- Buffer 管理
- 视频采集
- Linux Camera Framework

对于 Linux 驱动开发来说，这是最重要的一层。

------

# 五、仅仅获取图像够吗？

对于普通采图程序来说：

已经足够。

但是对于 AI 项目来说：

仅仅获取图像远远不够。

例如：

我们准备运行一个 YOLO 模型。

完整流程应该是：

```
Camera
    │
获取图像
    │
缩放
    │
颜色转换
    │
送入 RKNN
    │
YOLO 推理
    │
目标框绘制
    │
视频编码
    │
RTSP 推流
```

这里已经不仅仅是：

"获取图像"

而是：

"图像处理"

"AI 推理"

"视频编码"

"网络传输"

整个多媒体系统。

这已经超出了 V4L2 的职责范围。

------

# 六、CPU 能完成这些工作吗？

当然可以。

很多 PC 上的 OpenCV 程序就是这样工作的：

```
Camera
    │
V4L2
    │
CPU
 ├── Resize
 ├── Convert
 ├── Draw Box
 ├── Encode
 └── Network
```

这种方式开发简单。

但是：

对于 RV1106 这种嵌入式 SoC 来说，会带来几个明显的问题：

- CPU 占用高
- 大量 memcpy
- 图像重复拷贝
- 帧率下降
- 功耗增加

因此：

Rockchip 并不推荐所有工作都交给 CPU。

------

# 七、Rockchip 是如何解决这个问题的？

RV1106 内部集成了大量专用硬件。

例如：

```
ISP
VI
VPSS
RGA
VENC
VO
NPU
```

这些模块分别负责：

| 模块      | 功能                     |
| --------- | ------------------------ |
| ISP       | 图像信号处理             |
| VI        | 视频输入                 |
| VPSS      | 图像缩放、裁剪、颜色转换 |
| RGA       | 图像加速                 |
| RKNN(NPU) | AI 推理                  |
| VENC      | H.264/H.265/JPEG 编码    |
| VO        | 视频输出                 |

这些硬件模块都可以直接处理图像。

CPU 不需要参与。

那么：

问题来了。

如何统一管理这么多模块？

答案就是：

> **RKMPI。**

------

# 八、什么是 RKMPI？

RKMPI：

Rockchip Media Process Interface

它是 Rockchip 提供的一套媒体开发框架。

它最大的作用不是：

"取图"

而是：

> **把整个多媒体处理流程连接起来。**

例如：

```
Camera
     │
     VI
     │
    VPSS
     │
   RKNN
     │
   YOLO
     │
   VENC
     │
   RTSP
```

所有模块之间：

几乎都不需要 CPU 拷贝数据。

而是：

直接使用 DMA Buffer。

这也是 RKMPI 最大的价值。

------

# 九、V4L2 与 RKMPI 的关系

很多人第一次学习都会误认为：

```
RKMPI
    │
替代
    │
V4L2
```

实际上并不是。

更加准确的关系应该是：

```
                Linux

           V4L2 Framework

────────────────────────────────

          Rockchip SDK

              RKMPI

────────────────────────────────

           Application
```

可以这样理解：

V4L2 提供：

> **Linux Camera Framework**

而 RKMPI 提供：

> **Rockchip Media Pipeline**

两者关注点完全不同。

因此：

学习 RKMPI 并不是放弃 V4L2。

而是在 V4L2 的基础上继续向上学习。

------

# 十、本系列最终实现什么？

本系列最终目标不是：

"运行一个官方 Demo"

而是：

自己搭建一套完整的 AI Camera Pipeline。

最终的数据流如下：

```
Camera Sensor
        │
    MIPI CSI
        │
       ISP
        │
       VI
        │
      VPSS
   ┌────┴────┐
   │         │
 RKNN      VENC
   │         │
 YOLO      RTSP
   │
 OSD
   │
最终视频输出
```

最终，我们将一步一步完成：

- 使用 RKMPI 获取图像
- VPSS 图像处理
- RKNN 模型加载
- YOLO 推理
- OSD 绘制
- H.264 硬件编码
- RTSP 网络推流

最终实现一个完整的 AI 摄像头系统。

------

# 十一、本章总结

本章重点解决了一个问题：

> **为什么学习 RKMPI？**

需要牢记以下几点：

① **V4L2 负责 Linux Camera Framework。**

② **RKMPI 负责 Rockchip Media Pipeline。**

③ **AI 项目不仅需要获取图像，更需要构建完整的数据处理流水线。**

④ **RKMPI 的核心价值不是取图，而是充分利用 SoC 内部的硬件模块，实现高性能、低功耗的数据流转。**

从下一章开始，我们将正式分析 **RKMPI 的整体架构**，重点理解 **SYS、VI、VPSS、MB、VENC、VO** 等模块之间的关系，以及它们是如何共同组成一条完整的 AI Camera Pipeline。