## 第一章：为什么 Linux 没有设备管理器？

### 前言

对于接触过 Windows 的开发者来说，查看硬件设备最熟悉的方法就是打开**设备管理器**。

设备管理器中会列出系统已经识别的各种硬件，例如：

- 摄像头（Camera）
- USB 设备
- 网卡
- 声卡
- 串口
- 显卡

开发者通常只需要安装对应驱动即可使用这些设备。

但是，当我们来到 Linux 后，却发现系统中并没有类似 Windows "设备管理器" 的软件。

那么问题来了：

> **Linux 是如何管理这些硬件设备的？**

这也是学习 V4L2 之前必须理解的第一个问题。

---

### Linux 如何管理硬件？

Linux 并没有一个统一的"设备管理器"。

取而代之的是，大多数能够被用户空间访问的硬件，都会对应一个**设备节点（Device Node）**。

这些设备节点通常位于：

```bash
/dev
```

例如执行：

```bash
ls /dev
```

可以看到类似下面的输出：

```text
console
null
ttyS0
ttyS1
i2c-1
spidev0.0
video0
video1
media0
random
zero
...
```

这里的每一个文件，都对应着 Linux 内核中的一个设备。

> **注意：这里所说的"文件"并不是普通文本文件，而是 Linux 为设备提供的一种统一访问接口。**

---

### 为什么 Linux 要把设备设计成"文件"？

Linux 的设计目标之一，就是让应用程序尽量不用关心底层硬件的实现方式。

例如，访问一个普通文本文件通常需要：

```c
open();
read();
write();
close();
```

而访问串口、I2C、SPI、Camera 等设备时，同样也是通过这些系统调用完成。

例如：

| 硬件         | 常见设备节点     |
| ------------ | ---------------- |
| UART         | `/dev/ttyS0`     |
| I2C 控制器   | `/dev/i2c-1`     |
| SPI 控制器   | `/dev/spidev0.0` |
| Video Device | `/dev/video0`    |

对于一些设备，还会使用 `ioctl()` 来完成更加复杂的控制。

因此，无论访问的是普通文件还是硬件设备，应用程序都可以使用统一的接口进行操作。

这就是 Linux 中非常著名的设计思想：

> **Everything is a File（一切皆文件）**

需要注意的是，这句话并不是说硬件真的变成了普通文件，而是 Linux 为各种设备提供了类似文件的访问方式。

---

### 为什么设备节点都放在 /dev？

Linux 将不同类型的数据按照功能分别存放在不同目录。

例如：

| 目录    | 作用           |
| ------- | -------------- |
| `/bin`  | 常用命令       |
| `/etc`  | 系统配置文件   |
| `/home` | 用户目录       |
| `/proc` | 内核运行信息   |
| `/sys`  | Linux 设备模型 |
| `/dev`  | 设备节点       |

因此，当应用程序需要访问硬件时，通常都会首先进入 `/dev` 目录。

---

### Camera 为什么不是 camera0？

细心的读者可能已经发现：

在 `/dev` 目录中，并没有看到类似：

```text
camera0
```

而是：

```text
video0
video1
video2
...
```

为什么 Linux 不直接命名为 Camera？

原因在于：

Linux V4L2 管理的并不是"摄像头"这个硬件，而是**视频设备（Video Device）**。

一个完整的摄像头系统通常包括：

```text
Image Sensor
        │
     MIPI CSI
        │
        ISP
        │
   Video Device
```

真正提供给应用程序访问的，通常只是最后的 **Video Device**。

因此，我们看到的是：

```text
/dev/video0
```

而不是：

```text
/dev/camea0
```

至于为什么一个摄像头可能对应多个 `video` 设备，我们将在后面的章节详细介绍。

---

### 初识 Video Device

查看 Video Device 最简单的方法就是：

```bash
ls -l /dev/video*
```

例如：

```text
crw-rw---- 1 root video 81, 0 Jan  1 00:00 video0
crw-rw---- 1 root video 81, 1 Jan  1 00:00 video1
```

这里已经出现了许多新的内容，例如：

- 为什么前面是 `c`？
- `81` 表示什么？
- `0` 又表示什么？
- 为什么会有多个 Video Device？

这些问题暂时不用着急。

因为它们都和 Linux 中最重要的一类设备——**字符设备（Character Device）**——有关。

---

### 本章小结

本章主要介绍了 Linux 如何管理硬件设备，并了解了 V4L2 学习过程中最重要的入口——`/dev/video*`。

我们知道了：

- Linux 没有 Windows 那样的设备管理器；
- 大多数设备都会对应一个设备节点（Device Node）；
- 设备节点通常位于 `/dev` 目录；
- Linux 使用统一的系统调用访问各种设备；
- V4L2 管理的是 Video Device，而不是 Camera。

不过，一个新的问题也随之出现：

> `/dev/video0` 究竟是什么？

为什么它前面有一个 `c`？

为什么还有 `81,0`？

这些设备节点又是谁创建出来的？

下一章，我们将正式认识 Linux 中最重要的一类设备——**字符设备（Character Device）**。





## 第二章：什么是字符设备（Character Device）？

### 前言

上一章，我们认识了 Linux 中的设备节点（Device Node）。通过下面的命令，我们可以查看当前系统中的 Video Device：

```bash
ls -l /dev/video*
```

例如：

```text
crw-rw---- 1 root video 81, 0 Jan  1 00:00 video0
crw-rw---- 1 root video 81, 1 Jan  1 00:00 video1
```

相比普通文件，这里的输出多了许多陌生的信息，例如：

- `c`
- `81`
- `0`

其中，第一个出现的 **c** 就代表了 Linux 中非常重要的一类设备——**字符设备（Character Device）**。

那么，什么是字符设备？为什么摄像头属于字符设备？

---

### Linux 中的文件类型

很多开发者第一次接触 Linux 时，都会认为：

> 文件就是文件。

实际上并不是。

Linux 中的文件不仅仅指普通文本文件，还包括目录、链接文件、设备文件等多种类型。

可以通过下面的命令查看：

```bash
ls -l
```

例如：

```text
-rw-r--r-- 1 root root   1024 test.txt
drwxr-xr-x 2 root root   4096 image
lrwxrwxrwx 1 root root      9 lib -> /usr/lib
crw-rw---- 1 root video 81,0 video0
brw-rw---- 1 root disk   8,0 sda
```

大家会发现，每一行最前面都有一个字符。

这个字符表示文件类型。

---

### Linux 常见文件类型

Linux 中最常见的文件类型如下：

| 标识 | 类型     | 说明                  |
| ---- | -------- | --------------------- |
| `-`  | 普通文件 | 文本、图片、程序等    |
| `d`  | 目录     | 文件夹                |
| `l`  | 软链接   | 类似 Windows 快捷方式 |
| `c`  | 字符设备 | 串口、Camera、I2C 等  |
| `b`  | 块设备   | 硬盘、SD 卡等         |

例如：

```text
-rw-r--r--
```

表示：

普通文件。

而：

```text
crw-rw----
```

表示：

这是一个字符设备。

---

### 什么是字符设备？

字符设备（Character Device）是一种**按照字符流（Byte Stream）进行访问**的设备。

所谓字符流，可以理解为：

> 数据按照先后顺序，一个字节一个字节地进行传输。

例如：

串口接收数据：

```text
A
↓
B
↓
C
↓
D
```

程序按照收到的顺序依次读取。

再例如：

I2C：

```text
发送地址
↓
发送寄存器
↓
读取数据
```

SPI：

```text
发送一个字节
↓
接收一个字节
↓
继续发送
```

这些设备的数据通常都是连续流动的，因此统称为**字符设备**。

---

### Camera 为什么也是字符设备？

很多同学第一次看到这里都会产生疑问：

> 摄像头输出的是一张图片，为什么也是字符设备？

实际上，摄像头并不是一次输出一张图片。

而是持续不断地输出视频流。

例如：

```text
Frame1
↓
Frame2
↓
Frame3
↓
Frame4
↓
......
```

每一帧图像都会不断送到应用程序。

对于应用程序来说，它获取的是一个连续不断的数据流，因此 Linux 同样将 Video Device 设计成字符设备。

也正因为如此，我们才能通过统一的接口访问 Camera。

---

### 字符设备是如何访问的？

Linux 为字符设备提供了统一的访问接口。

例如：

```c
int fd = open("/dev/video0", O_RDWR);
...
close(fd);
```

如果只是普通字符设备，例如串口，还可以直接使用：

```c
read();
write();
```

而对于 Camera，由于需要设置分辨率、像素格式、申请缓冲区等复杂操作，仅靠 `read()` 和 `write()` 已经无法满足需求。

因此，Linux 又提供了另外一个更加强大的系统调用：

```c
ioctl()
```

后续 V4L2 的所有控制命令，几乎都是通过 `ioctl()` 完成的。

因此，现在可以先记住一句话：

> **打开设备使用 `open()`，控制设备使用 `ioctl()`。**

后面的章节我们会详细介绍。

---

### 字符设备和块设备有什么区别？

Linux 中还有另外一种非常重要的设备：

**块设备（Block Device）**。

例如：

```text
/dev/sda
/dev/mmcblk0
/dev/nvme0n1
```

这些都是块设备。

字符设备和块设备最大的区别如下：

| 字符设备                         | 块设备                    |
| -------------------------------- | ------------------------- |
| 数据连续流动                     | 数据按块存储              |
| 一般不支持随机访问               | 支持随机访问              |
| 典型设备：串口、I2C、SPI、Camera | 典型设备：硬盘、U盘、SD卡 |

举个例子：

读取串口：

```text
A
↓
B
↓
C
↓
D
```

必须按顺序读取。

而读取硬盘：

```text
第100块
↓
第20块
↓
第500块
```

可以任意跳转。

因此，两者采用了完全不同的驱动模型。

---

### 本章小结

本章我们认识了 Linux 中一种非常重要的设备类型——字符设备（Character Device）。

主要了解了以下内容：

- Linux 中存在多种文件类型；
- `c` 表示字符设备；
- Camera、串口、I2C、SPI 都属于字符设备；
- 字符设备通过 `open()`、`read()`、`write()`、`ioctl()` 等系统调用进行访问；
- Camera 主要使用 `ioctl()` 进行控制。

不过，新的问题又出现了。

重新观察下面这条命令：

```bash
ls -l /dev/video0
```

输出类似：

```text
crw-rw---- 1 root video 81,0 video0
```

现在，我们已经知道：

- `c` 表示字符设备。

但是：

- `81` 是什么意思？
- `0` 又表示什么？
- Linux 又是如何根据这两个数字找到真正的驱动程序？

下一章，我们将继续分析字符设备中的另一个重要概念——**主设备号（Major）与次设备号（Minor）**。





## 第三章：81,0到底是什么？（Major / Minor）

### 前言

上一章我们认识了字符设备（Character Device），并知道：

```text
crw-rw---- 1 root video 81, 0 video0
```

最前面的 `c` 表示这是一个字符设备。

但是后面两个数字：

```text
81, 0
```

依然是一个“黑盒”。

很多教程会直接告诉你：

> 81 是主设备号（Major）
> 0 是次设备号（Minor）

但问题是：

> **为什么需要这两个数字？它们到底在解决什么问题？**

这一章，我们从“为什么需要编号”开始讲。

---

### 一、一个核心问题：Linux 如何找到驱动？

当我们执行下面这条命令时：

```c
open("/dev/video0");
```

Linux 内核其实需要解决一个问题：

> **用户打开的是 video0，但内核如何知道应该交给哪个驱动？**

因为系统中可能存在：

- 多个摄像头
- 多个 video 设备
- 不同厂商的驱动

例如：

```text
/dev/video0  → 摄像头A
/dev/video1  → 摄像头B
/dev/video11 → ISP统计
```

那么问题来了：

> Linux 不能靠名字判断设备属于谁。

因此，需要一种更底层的机制来“快速定位驱动”。

---

### 二、解决方案：用数字标识设备类型

Linux 的解决方式非常直接：

> **用两个数字来唯一标识一个设备入口**

这就是：

```text
Major Number（主设备号）
Minor Number（次设备号）
```

例如：

```text
81, 0
```

含义是：

| 数值 | 含义                            |
| ---- | ------------------------------- |
| 81   | 设备类型（Video4Linux）         |
| 0    | 该类型下的第 0 个设备（video0） |

---

### 三、验证：81到底是什么？

我们可以在系统中验证这个结论。

执行：

```bash
cat /proc/devices
```

你会看到类似输出：

```text
Character devices:
  1 mem
  4 /dev/vc/0
 10 misc
 81 video4linux
 89 i2c
116 alsa
180 usb
```

我们发现：

```text
81 → video4linux
```

这说明：

> `/dev/video*` 属于 video4linux 这一类设备。

---

### 四、Minor Number（0,1,2）是什么？

继续观察：

```bash
ls -l /dev/video*
```

可能看到：

```text
81, 0 → video0
81, 1 → video1
81, 2 → video2
```

那么问题来了：

> **为什么同一个 81 会有多个设备？**

答案是：

> **Minor 用来区分同一类设备中的不同实例**

例如在 RV1106 上：

```text
video0 → 主视频流（main path）
video1 → 辅助视频流（sub path）
video11 → metadata / statistics
```

这些设备：

- 都属于 video4linux（81）
- 但用途不同

因此：

```text
81 → 类型
0  → 实例
1  → 实例
11 → 实例
```

---

### 五、为什么不能只用文件名？

你可能会想：

> Linux 已经有 `/dev/video0` 这个名字了，为什么还要 Major/Minor？

原因是：

#### 1. 文件名只是“人类可读名称”

```text
video0
```

只是一个字符串。

Linux 内核不会依赖字符串来匹配设备。

---

#### 2. 内核需要“快速索引”

当执行：

```c
open("/dev/video0");
```

内核流程是：

```text
路径解析
   ↓
找到 inode
   ↓
读取 (Major, Minor)
   ↓
找到对应 driver
```

如果用字符串匹配：

- 太慢
- 不可靠
- 不统一

---

#### 3. Major 才是“驱动入口”

可以理解为：

```text
Major = 找驱动
Minor = 找设备
```

---

### 六、一个非常重要的类比（一定要理解）

可以把 Linux 设备模型类比成：

#### 公司系统

| 概念        | 类比                  |
| ----------- | --------------------- |
| Major       | 部门（Camera部门）    |
| Minor       | 员工编号（第0号设备） |
| /dev/video0 | 员工名字              |
| 驱动        | 部门负责人            |

当你说：

> 我要找 video0

系统实际流程是：

```text
video0
 ↓
81（Camera部门）
 ↓
找 video4linux 驱动
 ↓
再找到第0号设备
```

---

### 七、现在我们已经能回答一个关键问题

回到最初的问题：

> open("/dev/video0") 时发生了什么？

现在可以回答一部分了：

```text
1. Linux 找到 /dev/video0
2. 读取 Major = 81
3. 找到 video4linux 驱动
4. 再用 Minor = 0 找到具体设备
```

但是还有一个问题没有解决：

> **这些 81,0 是谁分配的？**

---

### 八、本章小结

本章我们解决了一个关键问题：

> `/dev/video0` 中的 `81,0` 到底是什么？

我们得到了以下结论：

- Major Number 表示设备类型
- Minor Number 表示同类设备中的实例
- `81` 对应 video4linux
- `0/1/2` 对应不同 video 设备
- Linux 通过 Major/Minor 定位驱动，而不是文件名
- `/dev/video0` 本质是一个“映射入口”

---

但是，一个新的问题出现了：

> **这些设备节点是谁创建出来的？**
>
> 为什么我们插入摄像头后 `/dev/video0` 会自动出现？

下一章，我们将进入 Linux 设备模型的核心机制：

> **设备节点的创建过程（udev + sysfs + kernel）**







## 第四章：/dev/video0 是谁创建的？（udev / sysfs / 内核设备模型）

### 前言

在前面的章节中，我们已经知道：

```text
/dev/video0 → 81,0 → video4linux 驱动
```

但是这里有一个更本质的问题：

> **Linux 是怎么“自动出现” /dev/video0 的？**

因为我们并没有手动创建这个文件，例如：

```bash
mknod /dev/video0 c 81 0
```

但系统启动或插入摄像头后，它却自动出现了。

那么问题来了：

> **是谁在创建 /dev/video0？**

---

### 一、先做一个实验：删除 /dev/video0 会发生什么？

我们可以尝试（在开发板上）：

```bash
rm /dev/video0
```

然后查看：

```bash
ls /dev/video*
```

通常会发现：

> video0 很快又出现了，或者重启后自动恢复。

这说明：

> `/dev/video0` 不是普通文件，它是“动态生成的”。

---

### 二、/dev 不是内核直接管理的

很多初学者会误以为：

> Linux 内核直接创建 /dev/video0

这是错误的。

实际上分工是这样的：

```text
内核（Kernel）
    ↓
提供设备信息（sysfs）
    ↓
用户空间守护进程（udev）
    ↓
创建 /dev 下的节点
```

---

### 三、关键组件：sysfs

Linux 内核中有一个非常重要的机制：

> **sysfs（/sys 文件系统）**

我们可以查看：

```bash
ls /sys/class/video4linux/
```

可能看到：

```text
video0
video1
video11
```

再继续：

```bash
ls /sys/class/video4linux/video0
```

你会看到：

```text
device
name
dev
power
subsystem
```

---

#### 重点来了

在这里有一个关键文件：

```bash
cat /sys/class/video4linux/video0/dev
```

输出类似：

```text
81:0
```

---

### 四、关键真相：/dev/video0 来源于 sysfs

我们现在可以拼出完整链路：

```text
内核创建 video device
        ↓
sysfs 导出信息
        ↓
/sys/class/video4linux/video0/dev
        ↓
udev 读取 sysfs
        ↓
创建 /dev/video0
```

---

### 五、udev 是什么？

udev 是 Linux 用户空间的一个守护进程。

它的作用只有一句话：

> **监听内核设备事件，并动态创建 /dev 设备节点**

---

#### udev 的工作流程

当系统检测到一个设备时：

```text
Kernel 产生设备事件（uevent）
        ↓
udev 接收事件
        ↓
读取 /sys/class/xxx/dev
        ↓
获取 Major/Minor（81:0）
        ↓
调用 mknod()
        ↓
创建 /dev/video0
```

---

### 六、为什么不能直接由内核创建 /dev？

这是一个非常重要的设计思想：

#### Linux 分层设计原则

Linux 将职责拆分为两部分：

| 层级          | 职责                   |
| ------------- | ---------------------- |
| 内核 Kernel   | 管理硬件、提供设备信息 |
| 用户空间 udev | 管理设备节点文件       |

---

#### 为什么要这样设计？

原因有三个：

#### 1. 灵活性

不同系统可以有不同命名规则：

```text
video0
camera0
front_camera
rear_camera
```

---

#### 2. 安全性

内核不直接操作文件系统，减少风险。

---

#### 3. 可配置性

可以通过规则文件：

```bash
/etc/udev/rules.d/
```

自定义设备名称。

例如：

```text
SC3336 → /dev/front_cam
```

---

#### 七、RV1106 上的真实流程

结合你的 RV1106（SC3336）：

```text
SC3336 Sensor
        ↓
MIPI CSI
        ↓
RK ISP Driver
        ↓
V4L2 Video Device
        ↓
sysfs (/sys/class/video4linux/video0)
        ↓
udev
        ↓
/dev/video0
```

---

### 八、现在可以回答核心问题了

#### /dev/video0 是谁创建的？

答案是：

> **udev（用户空间）创建的，而不是内核直接创建的**

但更完整的答案是：

```text
内核负责“描述设备”
用户空间负责“生成设备文件”
```

---

### 九、为什么我们一定要理解这一层？

因为后面 V4L2 会涉及：

- 多个 video node
- media0 拓扑
- 设备绑定关系
- 驱动 probe
- 设备热插拔

如果不理解这一层，会出现：

> “video0 到底是哪一个摄像头？”

这种问题无法解释。

---

### 十、本章小结

本章我们解决了一个关键问题：

> `/dev/video0 是如何产生的？`

结论如下：

- `/dev/video0` 并不是内核直接创建的
- 内核通过 sysfs 暴露设备信息
- udev 读取 sysfs 并创建设备节点
- 设备节点本质是 Major/Minor 的映射
- 整个过程是 Linux 设备模型的一部分

---

### 下一章预告

现在我们已经知道：

- video0 是怎么来的
- 81,0 是怎么来的
- /dev 是谁创建的

但是新的问题来了：

> **为什么 Camera 不只是 video0，而是一个“复杂系统”？**

例如：

```text
video0
video11
media0
rkisp
rkcif
```

下一章我们将进入：

> **V4L2 的真正核心：Media Controller（摄像头不是一个设备，而是一条链路）**

