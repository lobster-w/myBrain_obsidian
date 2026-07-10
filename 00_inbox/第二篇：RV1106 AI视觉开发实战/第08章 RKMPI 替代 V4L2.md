# 第08章 RKMPI 替代 V4L2——为什么工业 AI Camera 不再直接用 V4L2？

> **在完成第07章之后，我们已经跑通了 RKMPI 的最小取图链路。**
>
> 本章将回答一个非常关键的问题：
> 👉 **既然 V4L2 已经能取图，为什么还需要 RKMPI？**
>
> 更进一步：
> 👉 **RKMPI 到底“替代”了 V4L2 的什么？**

------

# 一、本章目标

完成本章后，你将能够理解：

- V4L2 在 AI Camera 中的真实定位
- RKMPI 与 V4L2 的关系（不是替代，而是分层）
- 为什么工业项目几乎不用 V4L2 直接做 AI
- RKMPI 相比 V4L2 的性能优势在哪里
- 一条完整 AI Pipeline 的正确打开方式

------

# 二、先纠正一个常见误解

很多初学者会认为：

```text
RKMPI = 替代 V4L2
```

这是错误的。

正确关系应该是：

```text
            User Space
                │
            RKMPI / MPP
                │
        -------------------
        │                 │
     VENC               VPSS
        │                 │
        └──────┬──────────┘
               │
             V4L2
               │
        Kernel Driver
               │
           Camera Sensor
```

------

## ✔ 关键结论：

> **V4L2 没有被替代，它仍然是 Linux Camera 的底层框架。**
> **RKMPI 是“在 V4L2 之上构建的多媒体 AI 管线框架”。**

------

# 三、V4L2 在 AI 场景中的问题

我们先看一个“纯 V4L2 + OpenCV”的经典流程：

```text
Camera
  │
V4L2
  │
mmap
  │
User Buffer
  │
memcpy
  │
OpenCV resize
  │
memcpy
  │
RKNN
```

------

## ❌ 问题1：CPU 参与过多

每一帧都要：

- memcpy
- resize
- color convert

CPU 变成“搬运工”。

------

## ❌ 问题2：数据链路断裂

每个模块都是“独立应用逻辑”：

```text
V4L2 → OpenCV → RKNN → Encoder
```

没有统一 pipeline。

------

## ❌ 问题3：无法利用硬件模块

例如：

- VPSS（缩放）
- RGA（图像加速）
- VENC（编码）

全部被绕过。

------

# 四、RKMPI 的解决方式

RKMPI 的思路非常直接：

> **不要让 CPU 处理图像流转，让硬件模块自己流动**

------

## RKMPI 结构：

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
RKNN / VENC / VO
```

------

## 核心变化：

| 对比    | V4L2          | RKMPI            |
| ------- | ------------- | ---------------- |
| 数据流  | 程序控制      | Pipeline自动流动 |
| CPU负担 | 高            | 极低             |
| Resize  | CPU           | VPSS             |
| Buffer  | mmap + memcpy | MB Zero Copy     |
| 架构    | 分散          | 统一             |

------

# 五、V4L2 的真实角色（非常重要）

很多人误以为：

> V4L2 = 取图接口

实际上在 Rockchip 平台：

V4L2 的真实角色是：

> **底层 Camera Driver 框架**

------

## 它做的事情是：

```text
Sensor
  │
MIPI CSI
  │
Kernel Driver (V4L2)
  │
ISP / VI 输入
```

------

## 它“不做”的事情：

- 不做 AI
- 不做 Resize
- 不做 Zero Copy Pipeline
- 不管理 VPSS / VENC / RKNN

------

# 六、RKMPI 为什么“看起来替代了 V4L2”？

原因是：

👉 RKMPI 把“上层开发复杂度”全部接管了

------

## 开发体验对比：

### ❌ V4L2 开发：

```c
open("/dev/video0")
ioctl(REQBUF)
mmap()
streamon()
dqbuf()
memcpy()
```

然后：

```text
OpenCV处理
RKNN推理
```

------

### ✔ RKMPI 开发：

```c
RK_MPI_SYS_Init();

RK_MPI_VI_CreateChn();
RK_MPI_VPSS_CreateGrp();

RK_MPI_SYS_Bind();

RK_MPI_VI_StartStream();
```

然后：

```text
直接GetFrame
```

------

## 本质差异：

| 项目           | V4L2     | RKMPI        |
| -------------- | -------- | ------------ |
| 控制粒度       | 很底层   | 系统级       |
| 目标           | 获取图像 | 构建Pipeline |
| 是否AI友好     | ❌        | ✔            |
| 是否零拷贝体系 | ❌        | ✔            |

------

# 七、为什么 AI Camera 必须用 RKMPI？

因为 AI Camera 不是“取一张图”：

而是：

```text
实时视频流 + AI + 编码 + 显示
```

例如：

```text
Camera
  │
VPSS
  ├── RKNN（检测）
  ├── VENC（编码）
  ├── VO（显示）
  └── OSD（叠加）
```

------

## 如果用 V4L2：

你必须自己做：

- 线程调度
- buffer同步
- 多路复制
- CPU resize
- AI输入预处理
- 编码管理

------

## 如果用 RKMPI：

你只需要：

```text
Bind 一下
```

系统自动完成：

- 数据流动
- buffer共享
- 硬件调度

------

# 八、一个关键认知：谁才是“真正的控制器”？

我们可以用一句话总结：

------

## ❌ V4L2 模型：

> CPU 控制数据流

------

## ✔ RKMPI 模型：

> 硬件控制数据流，CPU 只负责配置

------

# 九、Zero Copy 在两者中的差异

------

## V4L2：

```text
Kernel Buffer
  ↓ mmap
User Buffer
  ↓ memcpy
OpenCV Buffer
  ↓ memcpy
RKNN Input
```

------

## RKMPI：

```text
MB (共享Buffer)
  ↓
VI
  ↓
VPSS
  ↓
RKNN
```

------

## 核心区别：

| 项目       | V4L2   | RKMPI        |
| ---------- | ------ | ------------ |
| Copy次数   | 多次   | 0            |
| Buffer归属 | 用户态 | 系统统一管理 |
| 数据流     | 分散   | Pipeline     |

------

# 十、什么时候还需要 V4L2？

RKMPI 并不是完全替代 V4L2。

------

## V4L2 仍然适用：

- USB Camera
- 非 Rockchip ISP pipeline
- 调试 Camera driver
- 自定义 kernel driver

------

## RKMPI 适用：

- AI Camera
- RKNN 推理
- 视频编码
- 多路输出
- RTSP/IPC 产品

------

# 十一、本章总结

本章核心结论只有四点：

------

## ✔ ① V4L2 没有被替代

它仍然是 Linux Camera Driver 框架。

------

## ✔ ② RKMPI 是“上层媒体管线框架”

负责：

- VI
- VPSS
- RKNN
- VENC

------

## ✔ ③ AI Camera 的核心不是“取图”，而是“Pipeline”

V4L2 只能取图
RKMPI 才能构建系统

------

## ✔ ④ 真正的差异是架构，而不是 API

| 思维     | V4L2     | RKMPI        |
| -------- | -------- | ------------ |
| 模型     | 逐帧处理 | 流式处理     |
| 控制方式 | CPU控制  | Pipeline控制 |

------

# 十二、下一章预告

> 第09章 RKNN 模型加载——让 AI 真正跑在 RKMPI Pipeline 上

下一章开始，我们将进入真正的 AI 部分：

- RKNN 模型是什么？
- 如何加载 YOLO 模型？
- 输入如何从 VPSS 接入 RKNN？
- 第一帧 AI 推理如何跑通？

从这一章开始：

> **系统从“取图阶段”进入“智能阶段”**