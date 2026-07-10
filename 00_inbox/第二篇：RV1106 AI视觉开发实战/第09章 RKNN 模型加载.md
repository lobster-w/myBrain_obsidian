# 第09章 RKNN 模型加载——让 AI 真正接入 RKMPI Pipeline

> **从本章开始，RKMPI 系列正式进入“AI阶段”。**
>
> 前面我们已经完成了：
>
> - Camera 取图（VI）
> - 图像处理（VPSS）
> - Zero Copy Pipeline（MB + SYS Bind）
>
> 现在我们要做一件关键的事：
>
> 👉 **把 RKNN（NPU 推理）接入到整个数据流中**

------

# 一、本章目标

完成本章后，你将能够理解：

- RKNN 是什么？
- 为什么 AI 在 RV1106 上必须用 RKNN？
- RKNN 模型是如何加载的？
- `.rknn` 文件到底是什么？
- RKNN 在 RKMPI Pipeline 中的位置
- 如何完成“第一帧 AI 推理”

------

# 二、整体 AI Pipeline 回顾

目前我们的系统已经是：

```text
Camera
  │
ISP
  │
VI
  │
VPSS
  │
(GetFrame)
```

但这只是：

> ❌ 图像采集系统
> ❌ 还不是 AI 系统

------

现在我们要扩展成：

```text
Camera
  │
VI
  │
VPSS
  │
RKNN
  │
YOLO
```

------

# 三、什么是 RKNN？

RKNN：

```text
Rockchip Neural Network
```

它是 Rockchip 提供的：

> **NPU（神经网络加速器）推理框架**

------

## 它的作用很简单：

> 把 AI 模型跑在 NPU 上，而不是 CPU 上

------

## 对比一下：

### ❌ CPU 推理

```text
Image
  ↓
CPU
  ↓
YOLO
  ↓
慢 / 高功耗
```

------

### ✔ RKNN 推理

```text
Image
  ↓
NPU（RKNN）
  ↓
YOLO
  ↓
高性能 / 低功耗
```

------

# 四、为什么必须用 RKNN？

在 RV1106 上：

| 方案      | 结果      |
| --------- | --------- |
| CPU YOLO  | 1~3 FPS   |
| RKNN YOLO | 20~60 FPS |

------

## 结论：

> **没有 RKNN，就没有实时 AI Camera**

------

# 五、RKNN 在 RKMPI 中的位置

RKNN 不是独立系统，而是 Pipeline 的一个节点：

```text
Camera
  │
VI
  │
VPSS
  │
RKNN
  │
YOLO
```

------

## 关键点：

RKNN **不会主动取图**

它只做一件事：

> **吃输入 Tensor → 输出结果**

------

# 六、RKNN 模型是什么？

RKNN 使用的是：

```text
.rknn 文件
```

------

## 它的本质：

| 类型    | 说明            |
| ------- | --------------- |
| .onnx   | 通用模型        |
| .tflite | TensorFlow Lite |
| .rknn   | RKNN 编译后模型 |

------

## RKNN 模型 = 已优化的 NPU 图

它已经包含：

- 权重
- Graph
- NPU 指令
- 内存布局优化

------

# 七、RKNN 生命周期

RKNN 使用非常固定的流程：

```text
1. 初始化
2. 加载模型
3. 设置输入
4. 推理
5. 获取输出
6. 释放
```

------

# 八、RKNN 基本 API

## 1️⃣ 创建上下文

```c
rknn_init()
```

------

## 2️⃣ 加载模型

```c
rknn_load_model()
```

或：

```c
rknn_init(model_path)
```

------

## 3️⃣ 输入数据

```c
rknn_inputs_set()
```

------

## 4️⃣ 执行推理

```c
rknn_run()
```

------

## 5️⃣ 获取输出

```c
rknn_outputs_get()
```

------

# 九、第一个 RKNN Demo（最小加载）

这里只做：

👉 **模型加载 + 一次推理验证**

------

## 示例代码

```c
#include <stdio.h>
#include <stdlib.h>
#include "rknn_api.h"

int main()
{
    printf("RKNN Init Start\n");

    rknn_context ctx;

    // 1. 读取模型
    FILE *fp = fopen("yolo.rknn", "rb");
    fseek(fp, 0, SEEK_END);
    int model_size = ftell(fp);
    rewind(fp);

    void *model = malloc(model_size);
    fread(model, 1, model_size, fp);
    fclose(fp);

    // 2. 初始化 RKNN
    int ret = rknn_init(&ctx, model, model_size, 0, NULL);
    if (ret < 0) {
        printf("rknn_init failed\n");
        return -1;
    }

    printf("RKNN model loaded successfully\n");

    // 3. 获取模型信息
    rknn_sdk_version version;
    rknn_query(ctx, RKNN_QUERY_SDK_VERSION, &version, sizeof(version));
    printf("SDK: %s\n", version.api_version);

    // 4. 释放
    rknn_destroy(ctx);
    free(model);

    printf("RKNN Done\n");

    return 0;
}
```

------

# 十、运行结果

如果成功，你会看到：

```text
RKNN Init Start
RKNN model loaded successfully
SDK: 1.x.x
RKNN Done
```

------

# 十一、这一章你真正理解了什么？

------

## 1️⃣ RKNN 不是算法，而是执行引擎

```text
YOLO / MobileNet / ResNet
        ↓
     RKNN
        ↓
        NPU
```

------

## 2️⃣ RKNN 不负责取图

图像来自：

```text
VI → VPSS
```

RKNN 只负责：

```text
推理
```

------

## 3️⃣ RKNN 是 Pipeline 的终点之一

```text
Camera → VI → VPSS → RKNN → YOLO
```

------

## 4️⃣ RKNN = AI 加速器接口

不是框架，不是模型，而是：

> **NPU 调度层**

------

# 十二、重要认知（非常关键）

很多人误解 RKNN：

------

## ❌ 错误理解：

> RKNN = YOLO

------

## ✔ 正确理解：

> RKNN = YOLO 的运行环境

------

# 十三、本章总结

本章核心结论：

------

## ✔ ① RKNN 是 Rockchip NPU 推理框架

用于在 RV1106 上运行 AI 模型。

------

## ✔ ② RKNN 只负责推理，不负责图像获取

图像来自：

```text
VI + VPSS
```

------

## ✔ ③ RKNN 模型是编译后的 .rknn 文件

已经针对 NPU 优化。

------

## ✔ ④ RKNN 是 AI Camera Pipeline 的核心计算节点

```text
Camera → VI → VPSS → RKNN → YOLO
```

------

# 十四、下一章预告

> 第10章 YOLO 推理——让 RKNN 输出真正的检测结果

下一章我们将进入真正的 AI 应用核心：

- RKNN 如何运行 YOLO
- 如何解析输出 tensor
- 如何得到 bounding box
- 第一帧目标检测结果如何生成

从这一章开始：

> **系统正式从“AI加载阶段”进入“AI识别阶段”**