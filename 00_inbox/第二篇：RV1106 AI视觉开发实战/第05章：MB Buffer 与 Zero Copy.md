# 第05章 MB Buffer 与 Zero Copy——为什么 RKMPI 可以做到高性能？

> **如果说 VPSS 是整个 AI Camera 的图像处理中心，那么 MB（Media Buffer）就是整个 RKMPI 的"血液系统"。**
>
> 所有模块之间传递的数据，几乎都依赖 MB Buffer。
>
> **理解 MB，也就理解了 RKMPI 为什么能够做到 Zero Copy（零拷贝）。**

------

# 一、本章目标

完成本章学习后，你将能够理解：

- 什么是 MB（Media Buffer）？
- 什么是 MB_BLK？
- 为什么 RKMPI 不使用 malloc()？
- 什么是 DMA Buffer？
- 什么是 Zero Copy？
- 为什么 Zero Copy 能提高性能？
- MB 在整个 AI Camera Pipeline 中起什么作用？

------

# 二、回顾 Pipeline

目前我们的数据流已经来到：

```text
Camera
    │
ISP
    │
VI
    │
VPSS
```

很多人认为：

VPSS 输出以后：

直接返回：

```cpp
unsigned char *buffer;
```

实际上并不是。

RKMPI 返回的是：

```text
Media Buffer
```

也就是：

```text
MB（Media Buffer）
```

------

# 三、什么是 Buffer？

先不要讨论 MB。

先来看：

什么叫 Buffer。

Buffer：

就是：

> **一块用于存放数据的内存。**

例如：

```c
char buf[1024];
```

或者：

```c
void *ptr = malloc(1024);
```

这都是：

Buffer。

如果存放的是图像。

例如：

```text
1920 × 1080

NV12
```

那么：

这块 Buffer 就保存了一帧图像。

------

# 四、普通 Buffer 有什么问题？

例如：

```c
void *image = malloc(size);
```

得到的是：

普通内存。

如果：

Camera：

输出图像。

通常流程如下：

```text
Camera

↓

DMA

↓

Kernel Buffer

↓

memcpy()

↓

User Buffer

↓

OpenCV

↓

RKNN
```

注意：

这里至少发生了一次：

```text
memcpy()
```

如果：

一帧：

3MB。

30FPS：

就是：

```text
3MB × 30

≈90MB/s
```

如果：

4K：

60FPS。

内存带宽消耗会非常惊人。

CPU：

大量时间：

都浪费在：

复制数据。

------

# 五、什么是 DMA？

DMA：

Direct Memory Access。

中文：

> **直接内存访问。**

DMA 最大特点：

> **不经过 CPU，就可以完成数据搬运。**

例如：

普通方式：

```text
Camera

↓

CPU

↓

Memory
```

CPU：

负责：

搬运。

DMA：

则变成：

```text
Camera

↓

DMA Engine

↓

Memory
```

CPU：

不用搬。

这样：

效率大幅提升。

------

# 六、为什么 DMA 还不够？

很多同学会问：

既然：

DMA 已经把数据放到内存。

为什么：

还需要 MB？

原因：

因为：

后面还有：

```text
VPSS

RKNN

VENC

VO
```

如果：

每经过一个模块：

都：

```text
memcpy()
```

例如：

```text
VI

↓

memcpy

↓

VPSS

↓

memcpy

↓

RKNN

↓

memcpy

↓

VENC
```

性能依然很差。

所以：

需要：

整个 Pipeline：

共享同一块内存。

------

# 七、什么是 Media Buffer（MB）？

MB：

Media Buffer。

可以理解为：

> **整个媒体框架统一管理的共享缓冲区。**

所有模块：

都使用：

同一套 Buffer。

例如：

```text
Camera

↓

VI

↓

MB

↓

VPSS

↓

MB

↓

RKNN

↓

MB

↓

VENC
```

注意：

这里：

没有：

```text
memcpy()
```

所有模块：

拿到的：

都是：

同一块 Buffer。

------

# 八、什么是 MB_BLK？

很多 Demo：

都会看到：

```c
frame.pMbBlk
```

很多初学者都会疑惑：

这是什么？

实际上：

MB_BLK：

可以理解成：

> **一个 Media Buffer 的句柄（Handle）。**

它不是：

真正的数据。

而是：

管理 Buffer 的对象。

真正的数据地址：

通常通过：

```c
RK_MPI_MB_Handle2VirAddr()
```

转换后获得。

也就是说：

```text
MB_BLK

↓

Virtual Address

↓

Image Data
```

------

# 九、什么是 Zero Copy？

Zero Copy：

中文：

> **零拷贝。**

它并不是：

没有数据。

而是：

**不复制数据。**

例如：

传统方式：

```text
VI

↓

memcpy

↓

VPSS

↓

memcpy

↓

RKNN
```

RKMPI：

```text
VI

↓

MB

↓

VPSS

↓

MB

↓

RKNN
```

所有模块：

共享：

同一个 Buffer。

因此：

整个过程中：

没有：

```text
memcpy()
```

这就是：

Zero Copy。

------

# 十、Zero Copy 为什么快？

假设：

一帧：

```text
1920 × 1080

NV12

≈3MB
```

传统方式：

```text
VI

↓

Copy

↓

VPSS

↓

Copy

↓

RKNN
```

两次 Copy。

每秒：

30FPS。

需要：

复制：

180MB。

如果：

4K：

60FPS。

数据量：

甚至达到：

GB/s。

CPU：

几乎全部时间：

都在搬运数据。

而：

Zero Copy：

```text
VI

↓

VPSS

↓

RKNN
```

只是传递Buffer没有复制因此CPU几乎不参与。

------

# 十一、MB 在整个 Pipeline 中的位置

现在可以重新画整个 Pipeline。

```text
                Camera
                   │
                  ISP
                   │
                   VI
                   │
        ┌────────────────────┐
        │   Media Buffer(MB) │
        └────────────────────┘
                   │
                 VPSS
                   │
        ┌────────────────────┐
        │   Media Buffer(MB) │
        └────────────────────┘
                   │
                 RKNN
                   │
                 YOLO
```

实际上MB一直存在只是开发者平时感觉不到。

------

# 十二、MB 与 V4L2 mmap 有什么区别？

很多同学学完 V4L2 后，会想到：

```text
mmap

是不是也是 Zero Copy？
```

答案是：**部分是。**

V4L2：

```text
Camera
↓
Kernel Buffer
↓
mmap
↓
User Space
```

减少了一次：

Kernel → User：

Copy。但是后面：OpenCV：RKNN：依然可能继续：

```text
memcpy()
```

而：

RKMPI：

整个 Pipeline：

都使用：

MB。

因此：

真正实现：

模块之间：

Zero Copy。

------

# 十三、本章总结

本章重点介绍了 MB 与 Zero Copy 的设计思想。

需要牢记以下几点：

**① MB（Media Buffer）是整个 RKMPI 中统一的数据载体。**

**② MB_BLK 并不是图像数据，而是 Media Buffer 的管理句柄。**

**③ Zero Copy 的本质不是"没有数据"，而是多个模块共享同一块 Buffer，而不是不断复制数据。**

**④ DMA 负责高效搬运数据，MB 负责统一管理数据，两者共同构成了 RKMPI 高性能媒体框架的基础。**

**⑤ 与传统的 V4L2 + OpenCV 方案相比，RKMPI 通过 MB 和 Zero Copy，大幅减少了 CPU 参与数据搬运的次数，使更多计算资源留给 AI 推理和业务逻辑。**

------

# 下一章预告

理解了 MB 之后，我们终于可以学习整个 RKMPI 最核心的机制：

> **第06章 SYS Bind——一行代码实现整个 AI Camera Pipeline**

届时将重点分析：

- 什么是 SYS？
- 为什么 `RK_MPI_SYS_Bind()` 如此重要？
- Bind 到底绑定了什么？
- 为什么 Bind 能够实现真正的 Zero Copy？
- VI → VPSS → RKNN 的数据是如何自动流动的？

学习完下一章，你就会真正理解 **为什么官方 Demo 几乎都离不开 `RK_MPI_SYS_Bind()`**，以及 RKMPI 媒体框架背后的整体设计思想。