# 第06章 SYS Bind 机制——RKMPI 媒体流水线的核心

> **如果说 MB（Media Buffer）解决了"数据放在哪里"的问题，那么 SYS Bind 解决的就是"数据往哪里流"的问题。**
>
> 在整个 RKMPI 中，`RK_MPI_SYS_Bind()` 是出现频率最高、也是最重要的接口之一。它将 VI、VPSS、VENC、RKNN 等独立模块连接起来，形成一条真正的 Media Pipeline。

------

# 一、本章目标

完成本章学习后，你将能够理解：

- 什么是 SYS？
- 什么是 Bind？
- 为什么需要 Bind？
- Bind 到底绑定了什么？
- 数据为什么能够自动流动？
- Bind 与 Zero Copy 有什么关系？
- 为什么官方 Demo 都大量使用 `RK_MPI_SYS_Bind()`？

------

# 二、回顾前面的章节

目前我们已经学习了：

```text
Camera
   │
 ISP
   │
  VI
   │
 VPSS
```

很多同学会产生一个疑问：

我们只是：

- 创建了 VI
- 创建了 VPSS

为什么：

图像就能够进入 VPSS？

答案就是：

> **实际上，它们并不会自动连接。**

必须：

手动建立连接。

这个连接：

就是：

```text
SYS Bind
```

------

# 三、什么是 SYS？

SYS：

全称：

```text
System Management
```

它不是：

某一个硬件。

也不是：

Camera。

它更像：

> **整个 RKMPI 的资源管理中心。**

SYS 主要负责：

- 初始化媒体系统
- 管理模块关系
- 创建数据通路
- 管理 Buffer 生命周期
- 建立模块之间的数据连接

因此：

整个 RKMPI：

首先都会：

```cpp
RK_MPI_SYS_Init();
```

结束时：

```cpp
RK_MPI_SYS_Exit();
```

SYS：

相当于：

整个媒体框架的调度中心。

------

# 四、为什么需要 Bind？

来看一个例子。

我们现在有：

```text
VI

VPSS
```

这是两个独立模块。

它们彼此不知道：

对方是否存在。

更不知道：

图像应该发给谁。

例如：

```text
VI
```

可能后面连接：

```text
VPSS
```

也可能：

```text
VENC
```

甚至：

```text
VO
```

因此：

不能写死。

必须：

运行时：

动态连接。

------

# 五、什么是 Bind？

所谓 Bind。

其实就是：

> **把一个模块的输出，连接到另一个模块的输入。**

例如：

```text
VI

↓

VPSS
```

实际上：

就是：

```text
Bind

VI → VPSS
```

以后：

VI 每产生一帧图像。

都会：

自动送入：

VPSS。

开发者：

不需要：

GetFrame()

再：

SendFrame()。

------

# 六、Bind 前后的区别

没有 Bind。

程序通常需要：

```text
VI

↓

GetFrame()

↓

CPU

↓

SendFrame()

↓

VPSS
```

CPU：

负责：

不断搬运数据。

而：

Bind 后：

```text
VI

↓

SYS

↓

VPSS
```

数据：

自动流动。

CPU：

无需参与。

------

# 七、Bind 到底绑定了什么？

很多初学者会误认为：

Bind：

绑定的是：

Buffer。

其实：

不是。

Bind：

绑定的是：

> **模块之间的数据通路（Pipeline）。**

例如：

```text
Source

↓

Destination
```

源码里面：

通常对应：

```cpp
MPP_CHN_S
```

这个结构。

例如：

```cpp
MPP_CHN_S src;
MPP_CHN_S dst;
```

里面描述：

```text
Module

Device

Channel
```

例如：

```text
VI

Device0

Channel0
```

绑定到：

```text
VPSS

Group0

Channel0
```

SYS：

根据这些信息。

建立：

数据流。

------

# 八、MPP_CHN_S 是什么？

这是 RKMPI 最重要的数据结构之一。

它表示：

> **媒体模块中的一个节点。**

可以理解成：

```text
Module

+

Device

+

Channel
```

例如：

```text
VI

Device0

Channel0
```

或者：

```text
VPSS

Group0

Channel1
```

Bind：

其实就是：

```text
Node A

↓

Node B
```

建立连接。

------

# 九、一个典型的 Pipeline

下面是官方 Demo 中最常见的数据流：

```text
Camera
    │
ISP
    │
VI
    │
SYS Bind
    │
VPSS
    │
SYS Bind
    │
RKNN
```

CPU：

几乎：

不参与。

图像：

自动流动。

------

# 十、为什么 Bind 能做到 Zero Copy？

上一章。

我们学习：

MB。

现在：

终于可以串起来。

数据流如下：

```text
Camera
    │
 DMA
    │
 MB
    │
 VI
    │
 SYS Bind
    │
 VPSS
    │
 SYS Bind
    │
 RKNN
```

注意：

这里：

没有：

```text
memcpy()
```

为什么？

因为：

Bind：

传递的：

不是：

图像。

而是：

MB。

所有模块：

共享：

同一个：

Media Buffer。

因此：

真正实现：

Zero Copy。

------

# 十一、一个更复杂的 Pipeline

Bind 最大优势就是：

支持：

一个输入。

多个输出。

例如：

```text
             Camera
                │
               ISP
                │
                VI
                │
             SYS Bind
                │
             VPSS Group0
          ┌─────┴──────┐
          │            │
      Channel0     Channel1
          │            │
      SYS Bind    SYS Bind
          │            │
         RKNN        VENC
          │            │
        YOLO        RTSP
```

同一帧数据：

同时：

送入：

AI。

录像。

互不影响。

而且：

无需：

再次复制。

------

# 十二、Bind 与 OpenCV 的区别

很多 PC 程序：

通常：

这样写：

```text
Camera

↓

Read()

↓

Resize()

↓

Inference()
```

每一步：

都是：

CPU。

而：

RKMPI：

```text
VI

↓

Bind

↓

VPSS

↓

Bind

↓

RKNN
```

整个流程：

几乎：

全部：

硬件。

CPU：

只是：

初始化。

因此：

性能优势：

非常明显。

------

# 十三、Bind 的生命周期

一个完整的 Bind 通常包含以下过程：

```text
创建模块
      │
      ▼
配置参数
      │
      ▼
SYS_Bind()
      │
      ▼
Start
      │
      ▼
自动数据流
      │
      ▼
Stop
      │
      ▼
SYS_UnBind()
      │
      ▼
销毁模块
```

其中：

`RK_MPI_SYS_UnBind()`：

用于解除模块之间的连接。

通常在程序退出时调用。

------

# 十四、SYS Bind 的设计思想

站在更高的角度来看。

SYS Bind 的本质就是：

> **把"模块调用"转换成"数据流连接"。**

开发者：

不需要：

自己管理：

每一帧图像。

只需要：

告诉系统：

```text
VI

↓

VPSS

↓

RKNN

↓

VENC
```

剩下的：

全部：

由媒体框架完成。

这也是现代多媒体 SoC 普遍采用的设计思想。

例如：

- Rockchip RKMPI
- Horizon HBMedia
- HiSilicon MPP
- NVIDIA DeepStream

虽然 API 不同。

但核心思想都是：

**Pipeline + Zero Copy。**

------

# 十五、本章总结

本章介绍了整个 RKMPI 中最核心的 SYS Bind 机制。

需要牢记以下几点：

**① SYS 是整个媒体框架的资源管理中心，负责初始化系统并管理各模块之间的数据流。**

**② `RK_MPI_SYS_Bind()` 的作用不是传递数据，而是建立模块之间的数据通路（Pipeline）。**

**③ Bind 的对象不是 Buffer，而是由 Module、Device（或 Group）、Channel 组成的媒体节点（MPP_CHN_S）。**

**④ Bind 建立完成后，图像可以在 VI、VPSS、RKNN、VENC 等模块之间自动流动，CPU 不再需要逐帧搬运数据。**

**⑤ MB（Media Buffer）提供统一的数据载体，SYS Bind 提供统一的数据通路，两者共同实现了 RKMPI 的 Zero Copy 架构，这也是 RKMPI 高性能的根本原因。**

------

# 下一章预告

前六章我们已经完成了整个理论体系：

- Camera Pipeline
- VI
- VPSS
- MB
- SYS Bind

从下一章开始，我们将正式进入代码实践。

> **第07章 第一个 RKMPI 取图程序——从零搭建 VI → VPSS 图像采集流程**

届时将完成：

- 初始化 SYS
- 创建 VI
- 创建 VPSS
- 建立 SYS Bind
- 获取一帧图像
- 保存为 NV12 文件
- 分析程序执行流程

至此，我们将第一次完整跑通属于自己的 **RKMPI 图像采集程序**，真正从理论走向实践。