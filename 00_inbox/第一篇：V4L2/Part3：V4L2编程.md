## 第10章 第一个 V4L2 程序：open() 与 VIDIOC_QUERYCAP

### 本章要解决的问题

经过前面的分析，我们已经知道：

- Camera 是字符设备；
- V4L2 是 Linux Camera 的统一框架；
- Camera Pipeline 最终会产生多个 Video Device；
- 我们已经能够找到真正用于采图的 `/dev/videoX`。

那么接下来，第一个问题就是：

> **如何让程序真正打开 Camera？**

---

### 打开 Video Device

Linux 中所有设备都遵循"一切皆文件"的设计思想。

因此，打开 Camera 与打开普通文件几乎没有区别。

```c
int fd = open("/dev/video11", O_RDWR);
```

成功后，系统返回一个文件描述符（File Descriptor）。

以后所有 V4L2 操作，都将通过这个 fd 完成。

---

### Camera 打开之后能做什么？

仅仅打开设备，并不能说明 Camera 可以正常工作。

应用程序首先需要了解：

- 驱动是否支持 V4L2
- 是否支持视频采集
- 驱动名称
- Driver Version
- Bus 信息

这些信息都可以通过：

```c
VIDIOC_QUERYCAP
```

获取。

---

### VIDIOC_QUERYCAP

```c
struct v4l2_capability cap;

ioctl(fd, VIDIOC_QUERYCAP, &cap);
```

典型输出：

```
Driver Name : rkisp_mainpath
Card Name   : rkisp_mainpath
Bus Info    : platform:rkisp
Version     : 6.x.x
Capabilities: VIDEO_CAPTURE_MPLANE
```

这些信息告诉我们：

- 当前 Driver 是什么
- 当前节点是否支持 Video Capture
- Buffer 类型是什么

这是后续所有操作的基础。

---

### 本章小结

本章完成了 Camera 的第一步：

```
open()

↓

VIDIOC_QUERYCAP
```

但是，我们仍然不知道：

Camera 支持哪些图像格式？

支持哪些分辨率？

支持多少帧率？

下一章继续分析。



## 第11章 枚举图像格式、分辨率与帧率

### 本章要解决的问题

Camera 打开以后，并不能立即采图。

因为应用程序还不知道：

- 支持哪些 Pixel Format？
- 支持哪些 Resolution？
- 支持哪些 FPS？

这些都需要查询。

---

### 枚举 Pixel Format

使用：

```c
VIDIOC_ENUM_FMT
```

即可获取 Camera 支持的图像格式。

例如：

```
NV12

YUYV

RGB24

MJPEG

GREY

RG10
```

不同 Camera 支持的格式并不相同。

---

### 枚举 Resolution

知道格式以后，还需要知道：

```
1920×1080

1280×720

640×480
```

这些信息通过：

```c
VIDIOC_ENUM_FRAMESIZES
```

获得。

---

### 枚举 Frame Rate

最后，还需要查询：

```
30 FPS

60 FPS

120 FPS
```

使用：

```c
VIDIOC_ENUM_FRAMEINTERVALS
```

即可获得。

---

### 设置最终格式

查询完成以后，需要告诉 Driver：

应用程序真正想使用什么格式。

例如：

```c
VIDIOC_S_FMT
```

设置：

```
1920×1080

NV12
```

随后可以通过：

```c
VIDIOC_G_FMT
```

再次确认 Driver 实际采用的格式。

注意：

Driver 不一定完全按照应用程序请求的参数工作。

因此，实际开发中通常都会再次读取确认。

---

### 本章小结

完成本章以后，我们已经知道：

- Camera 支持哪些格式；
- 支持哪些分辨率；
- 支持哪些帧率；
- 当前实际采用什么格式。

接下来，就可以正式申请 Buffer 了。



## 第12章 Buffer、mmap、QBUF、STREAMON 全流程

### 本章要解决的问题

很多初学者都会认为：

Camera 拍摄一张照片。

↓

程序读取。

实际上并不是这样。

V4L2 为了提高性能，引入了 Buffer Queue 机制。

---

### Camera Buffer 工作流程

整个采图流程如下：

```
REQBUFS

↓

QUERYBUF

↓

MMAP

↓

QBUF

↓

STREAMON

↓

DQBUF

↓

处理图像

↓

QBUF

↓

循环采集
```

这是 V4L2 最核心的一套机制。

---

### REQBUFS

首先申请多个 Buffer。

```c
VIDIOC_REQBUFS
```

通常申请：

```
4

8

16
```

个 Buffer。

---

### QUERYBUF

查询每个 Buffer 的信息。

包括：

- Offset
- Length
- Index

---

### MMAP

随后通过：

```c
mmap()
```

将 Driver Buffer 映射到用户空间。

以后应用程序无需再次复制图像数据。

---

### QBUF

将 Buffer 放回 Driver。

表示：

> 这个 Buffer 可以开始接收图像。

---

### STREAMON

通知 Driver：

正式开始采图。

此时：

Sensor

↓

ISP

↓

DMA

↓

Memory

开始持续工作。

---

### DQBUF

当 Driver 填满一个 Buffer 后：

应用程序通过：

```c
VIDIOC_DQBUF
```

取出一帧图像。

随后：

再次：

```c
VIDIOC_QBUF
```

继续进入下一轮采集。

因此：

DQBUF 与 QBUF 是一个不断循环的过程。

---

### 本章小结

整个 Buffer 生命周期如下：

```
REQBUFS

↓

QUERYBUF

↓

MMAP

↓

QBUF

↓

STREAMON

↓

DQBUF

↓

处理图像

↓

QBUF

↓

继续采图
```

不过，当程序退出时，还需要释放这些资源。

下一章介绍完整的视频采集生命周期。





## 第13章 一个完整的视频采集生命周期

### 本章要解决的问题

上一章完成了 Camera 图像采集。

但是，一个完整的程序不仅需要启动 Camera，还需要正确关闭 Camera。

否则：

- Buffer 无法释放；
- Driver 无法停止 DMA；
- Camera 资源可能一直被占用。

---

### 完整生命周期

一个典型 V4L2 程序如下：

```
open()

↓

QUERYCAP

↓

ENUM_FMT

↓

S_FMT

↓

REQBUFS

↓

QUERYBUF

↓

MMAP

↓

QBUF

↓

STREAMON

↓

DQBUF

↓

QBUF

↓

……

↓

STREAMOFF

↓

munmap

↓

close()
```

这就是一个完整 Camera 程序的生命周期。

---

### STREAMOFF

停止 Camera 数据流。

```c
VIDIOC_STREAMOFF
```

Driver 将停止 DMA。

Camera 不再输出图像。

---

### munmap

释放 mmap 建立的用户空间映射。

避免内存泄漏。

---

### close()

关闭 Video Device。

释放文件描述符。

整个 Camera 生命周期结束。

---

### 本系列总结

至此，我们已经完整学习了：

- Linux 字符设备
- Device Model
- V4L2 Framework
- Media Controller
- Camera Pipeline
- Video Device
- Buffer Queue
- MMAP
- Streaming

已经具备独立编写 V4L2 Camera 采集程序的能力。

下一阶段，我们将进一步分析：

- Camera Control（曝光、增益、白平衡）
- Media Controller 实战
- V4L2 Driver Framework
- VB2 Buffer Framework
- Linux Camera Driver 开发