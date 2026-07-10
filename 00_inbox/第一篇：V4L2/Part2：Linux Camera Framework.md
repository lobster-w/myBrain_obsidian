## 第5章 什么是 V4L2？为什么需要 V4L2 Framework？

### 本章要解决的问题

前面我们已经知道：

Linux 中 Camera 本质上就是一个字符设备。

但是新的问题来了。

不同厂商生产的 Camera Sensor、USB 摄像头、工业相机，它们的驱动完全不同。

如果每个驱动都设计自己的 API，那么应用程序将无法做到通用。

例如：

```
Camera A
    ↓
Driver A
    ↓
API A

Camera B
    ↓
Driver B
    ↓
API B

Camera C
    ↓
Driver C
    ↓
API C
```

对于 OpenCV、FFmpeg、GStreamer 等应用来说，就需要分别适配每一种 Camera Driver。

显然，这是不可接受的。

于是 Linux 提供了一套统一的视频设备框架——**V4L2（Video for Linux 2）**。

---

### 什么是 V4L2？

V4L2（Video for Linux 2）是 Linux 官方提供的视频设备框架。

它统一规定了：

- Camera Driver 如何实现
- 应用程序如何访问 Camera
- 图像 Buffer 如何管理
- 图像格式如何描述
- Camera 参数如何配置

以后，无论底层是什么 Camera，只要实现了 V4L2 接口，应用程序都可以采用统一的方式访问。

整个架构如下：

```
Application
(OpenCV / FFmpeg / GStreamer)
            │
            ▼
        V4L2 API
            │
            ▼
      Camera Driver
            │
            ▼
      Camera Hardware
```

应用程序不再关心底层驱动来自哪一家厂商，而是统一使用 V4L2 API 进行开发。

---

### V4L2 给我们带来了什么？

统一之后，所有 Camera 基本都支持相同的操作流程：

```
open()
↓
VIDIOC_QUERYCAP
↓
VIDIOC_ENUM_FMT
↓
VIDIOC_REQBUFS
↓
VIDIOC_STREAMON
↓
VIDIOC_DQBUF
```

这也是后面章节将要介绍的主要内容。

---

### 本章小结

V4L2 并不是一个 Camera 驱动。

它更像是一套规范（Framework）。

它定义了：

- Driver 如何实现
- Application 如何调用
- Buffer 如何管理
- Camera 如何配置

这样，不同厂商开发的 Camera Driver，都可以使用统一的 API。

不过，这里又会产生一个新的问题。

既然已经有了 `/dev/video0`，为什么系统里还会出现 `/dev/media0`？

下一章，我们来认识 Linux Camera 中另一个非常重要的 Framework —— **Media Controller**。





## 第6章 什么是 Media Controller？为什么还有 media0？

### 本章要解决的问题

很多人在第一次接触 Linux Camera 时，都会发现系统里除了：

```
/dev/video0
```

还多出了：

```
/dev/media0
```

于是很容易产生疑问：

> 不是已经有 Video Device 了吗？
>
> 为什么还需要一个 Media Device？

要回答这个问题，需要先理解现代 Camera 的组成。

---

### Camera 已经不是一个设备了

早期 Camera 很简单。

```
Camera
    │
    ▼
video0
```

应用程序直接打开 `/dev/video0` 就可以采集图像。

但是随着 Camera 技术的发展，一个 Camera 已经不再是一个独立设备，而是一条完整的数据处理链路。

例如：

```
Image Sensor
      │
      ▼
MIPI CSI Receiver
      │
      ▼
ISP
      │
      ▼
Video Device
```

Linux 将这条完整的数据通路称为：

> **Camera Pipeline**

---

### 什么是 Media Controller？

Media Controller 用来描述整个 Camera Pipeline。

它负责管理：

- Entity（设备节点）
- Pad（连接端口）
- Link（连接关系）
- Pipeline（整条数据流）

可以理解为：

> **整个 Camera 系统的拓扑管理器。**

通常可以使用：

```bash
media-ctl -p
```

查看整个 Camera Pipeline。

---

### 为什么需要 Media Controller？

以前只有一个 Camera。

```
Camera
    │
video0
```

现在一个 Camera 可能变成：

```
Sensor
↓
CSI
↓
ISP
↓
MainPath
↓
Video Device
```

甚至还可能包含：

- 多个 Sensor
- 多个 ISP
- 多路输出
- 多个 Video Device

此时，仅靠一个 `/dev/video0` 已经无法描述整个系统。

因此 Linux 引入了 Media Controller。

---

### 本章小结

V4L2 负责提供统一的视频接口。

Media Controller 则负责管理整个 Camera Pipeline。

二者并不是互相替代，而是共同组成现代 Linux Camera Framework。

不过，新的问题又来了。

既然 Camera 已经变成了一条 Pipeline，那么为什么系统里会同时出现：

```
/dev/media0
/dev/v4l-subdev0
/dev/video11
```

这些设备之间到底有什么区别？

下一章，我们继续分析 Linux Camera 中最容易混淆的三类设备。





## 第7章 为什么 Camera 会产生 media、subdev、video 三类设备？

### 本章要解决的问题

执行下面的命令：

```bash
ls /dev/media*
ls /dev/v4l-subdev*
ls /dev/video*
```

可能得到：

```
/dev/media0

/dev/v4l-subdev0
/dev/v4l-subdev1
/dev/v4l-subdev2

/dev/video0
/dev/video1
/dev/video11
/dev/video12
```

很多初学者都会疑惑：

> 为什么一个 Camera 会产生这么多设备？

事实上，它们承担着完全不同的职责。

---

### Linux Camera 的三类设备

```
                    Camera Pipeline
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
     /dev/mediaX      /dev/v4l-subdevX    /dev/videoX
        │                  │                  │
        │                  │                  │
   管理整个Pipeline      控制硬件模块      图像数据输入输出
```

它们共同组成了 Linux Camera Framework。

---

### Media Device

Media Device 用来描述整个 Camera Pipeline。

负责：

- 查看设备拓扑
- 查看 Entity
- 查看 Link
- 查看 Pipeline

它本身不会产生图像。

---

### Sub-device

Sub-device 表示 Pipeline 中的各个硬件模块。

例如：

- Image Sensor
- MIPI CSI
- ISP
- Lens

主要负责：

- Exposure
- Gain
- HDR
- Crop
- White Balance

它负责控制，而不是采图。

---

### Video Device

Video Device 才是真正负责图像输入输出的设备。

后面所有的 V4L2 API：

```
VIDIOC_REQBUFS
VIDIOC_QBUF
VIDIOC_STREAMON
VIDIOC_DQBUF
```

全部都是针对 Video Device 进行操作。

因此：

> 普通应用程序真正打开的几乎都是 `/dev/videoX`。

---

### 本章小结

三类设备的职责可以简单总结如下：

| 设备   | 作用                     |
| ------ | ------------------------ |
| media  | 管理整个 Camera Pipeline |
| subdev | 控制 Camera 各硬件模块   |
| video  | 图像数据输入输出         |

理解它们之间的职责分工，是理解 Linux Camera Framework 的关键。

不过，还有一个问题没有解决。

它们之间到底是如何连接起来的？

下一章，我们结合 RV1106 的 Camera Pipeline 进行分析。





## 第8章 分析 RV1106 Camera Pipeline（SC3336 → MIPI → ISP → Video Device）

### 本章要解决的问题

上一章介绍了 Linux Camera 中三类设备。

那么，它们在 RV1106 中到底是如何连接起来的？

下面以 SC3336 为例进行分析。

---

### RV1106 Camera Pipeline

```
             SC3336
                │
                ▼
        v4l-subdev0
                │
                ▼
          MIPI CSI
                │
                ▼
        v4l-subdev1
                │
                ▼
            RKISP
                │
                ▼
        v4l-subdev2
                │
        ┌───────┴────────┐
        ▼                ▼
   MainPath         SelfPath
        │                │
        ▼                ▼
  /dev/video11    /dev/video12
```

整个 Pipeline 的拓扑关系由：

```
/dev/media0
```

统一描述。

---

### Camera 数据是如何流动的？

整个数据流如下：

```
Sensor

↓

MIPI CSI

↓

ISP

↓

Video Device

↓

Application
```

应用程序真正获取图像的位置，是 Pipeline 的最后一级：

```
/dev/videoX
```

---

### 本章小结

Media Device 管理整个 Pipeline。

Sub-device 表示各个硬件模块。

Video Device 则负责真正的数据输出。

不过，一个 Camera 往往会对应多个 Video Device。

例如：

```
video0

video1

video11

video12
```

那么：

到底应该打开哪一个？

下一章继续分析。





## 第9章 为什么一个 Camera 会对应多个 /dev/videoX？到底该打开哪一个？

### 本章要解决的问题

很多人在第一次接触 Linux Camera 时都会遇到下面的问题：

```
/dev/video0

/dev/video1

/dev/video11

/dev/video12
```

为什么会有这么多个 Video Device？

到底应该打开哪一个？

---

### 一个 Camera 为什么会有多个 Video Device？

现代 ISP 往往支持多路输出。

例如：

```
            ISP
             │
     ┌───────┴────────┐
     ▼                ▼
 MainPath         SelfPath
     │                │
     ▼                ▼
video11         video12
```

不同 Video Device 可能对应：

- RAW 输出
- ISP 后图像
- 缩略图
- Metadata
- Statistics

因此，一个 Camera 对应多个 `/dev/videoX` 是非常正常的。

---

### 如何确定真正的 Capture Device？

不要凭经验去猜。

推荐按照下面的步骤分析：

```
media-ctl -p

↓

查看 Pipeline

↓

v4l2-ctl --list-devices

↓

查看所有 Video Device

↓

v4l2-ctl --all

↓

查看当前节点能力

↓

确定真正用于采图的 Video Device
```

这样，无论是：

- RV1106
- RK3588
- Jetson Orin
- RDK X5

分析方法都是一样的。

---

### 本章小结

不要死记：

```
video11
```

因为不同平台的设备编号完全可能不同。

真正应该掌握的是：

> **先分析 Pipeline，再确定 Capture Device。**

现在，我们已经知道：

- Camera 为什么会产生多个设备；
- 每种设备分别负责什么；
- 如何找到真正用于采图的 Video Device。

接下来，就可以正式开始编写第一个 V4L2 程序了。