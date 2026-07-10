# 第10章 YOLO 推理——让 RKNN 输出第一帧目标检测结果

> 到本章为止，我们已经完成了：
>
> - Camera 取图（VI）
> - 图像处理（VPSS）
> - Zero Copy Pipeline（MB + SYS Bind）
> - RKNN 模型加载
>
> 现在进入真正关键一步：
>
> 👉 **让 YOLO 在 RKNN 上“跑起来并输出检测结果”**

------

# 一、本章目标

完成本章后，你将能够：

- 理解 YOLO 在 RKNN 中的输入输出结构
- 完成第一帧推理（Inference）
- 看懂 RKNN 输出 Tensor
- 得到 bounding box（检测框）
- 理解 AI Camera 中“检测结果从哪里来”

------

# 二、整体 AI 流程回顾

现在系统已经是：

```text
Camera
  │
VI
  │
VPSS
  │
RKNN
  │
YOLO Output
```

但注意：

> ❗ RKNN 只输出 Tensor，不直接输出“框”

------

# 三、YOLO 在 RKNN 中做了什么？

YOLO 在 RKNN 中其实分三步：

```text
1. Input Tensor（图像）
        ↓
2. Neural Network Forward
        ↓
3. Output Tensor（原始结果）
```

------

## ❗ 重点：

RKNN 输出不是：

```text
[ x, y, w, h, class ]
```

而是：

```text
feature maps（特征张量）
```

------

# 四、YOLO 输入格式（非常关键）

通常 YOLO（如 YOLOv5 / YOLOv8）输入：

| 项目 | 值                |
| ---- | ----------------- |
| 尺寸 | 640×640           |
| 通道 | 3                 |
| 格式 | RGB / NCHW / NHWC |

------

## 但 VPSS 输出：

```text
1920×1080 NV12
```

------

## 所以必须做：

```text
VPSS
  ↓
Resize (640×640)
  ↓
Format Convert (NV12 → RGB)
  ↓
RKNN Input
```

👉 这一部分通常由：

- VPSS（部分 resize）
- 或 CPU / RGA

完成

------

# 五、RKNN 输入准备

RKNN 输入结构：

```c
rknn_input input;
```

------

## 示例：

```c
rknn_input input;
memset(&input, 0, sizeof(input));

input.index = 0;
input.type = RKNN_TENSOR_UINT8;
input.size = width * height * 3;
input.fmt = RKNN_TENSOR_NHWC;
input.buf = image_buffer;
```

------

# 六、推理流程

------

## 1️⃣ 设置输入

```c
rknn_inputs_set(ctx, 1, &input);
```

------

## 2️⃣ 执行推理

```c
rknn_run(ctx, NULL);
```

------

## 3️⃣ 获取输出

```c
rknn_output outputs[NUM_OUTPUTS];

rknn_outputs_get(ctx, NUM_OUTPUTS, outputs, NULL);
```

------

# 七、YOLO 输出结构（核心重点）

YOLO 输出通常是：

```text
Tensor1: bbox
Tensor2: confidence
Tensor3: class score
```

------

## 例如 YOLOv5：

输出 shape：

```text
[1, 25200, 85]
```

------

## 85 含义：

| 内容         | 数量 |
| ------------ | ---- |
| x            | 1    |
| y            | 1    |
| w            | 1    |
| h            | 1    |
| objectness   | 1    |
| class scores | 80   |

------

# 八、后处理的本质（非常重要）

RKNN 不会帮你做：

- NMS
- 阈值过滤
- 框转换

------

## 所以你必须做：

```text
1. decode bbox
2. confidence filter
3. NMS
4. 输出结果
```

------

# 九、最小 YOLO 推理 Demo（核心代码）

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "rknn_api.h"

#define INPUT_SIZE 640

int main()
{
    rknn_context ctx;

    // 1. 加载模型（略：与第09章一致）

    // 2. 假设已经拿到预处理后的图像
    unsigned char *input_image = malloc(INPUT_SIZE * INPUT_SIZE * 3);

    // 3. 输入设置
    rknn_input input;
    memset(&input, 0, sizeof(input));

    input.index = 0;
    input.type  = RKNN_TENSOR_UINT8;
    input.fmt   = RKNN_TENSOR_NHWC;
    input.size  = INPUT_SIZE * INPUT_SIZE * 3;
    input.buf   = input_image;

    rknn_inputs_set(ctx, 1, &input);

    // 4. 推理
    rknn_run(ctx, NULL);

    // 5. 输出
    rknn_output outputs[3];
    memset(outputs, 0, sizeof(outputs));

    rknn_outputs_get(ctx, 3, outputs, NULL);

    printf("Inference Done\n");

    // 6. 打印部分输出
    float *out = (float *)outputs[0].buf;
    printf("First value: %f\n", out[0]);

    // 7. 释放
    rknn_outputs_release(ctx, 3, outputs);
    free(input_image);

    return 0;
}
```

------

# 十、这一章最关键的理解点

------

## 1️⃣ RKNN 不输出“结果”，只输出“数学结果”

```text
RKNN → Tensor
```

------

## 2️⃣ YOLO 的“检测框”是后处理出来的

```text
Tensor → Decode → Box
```

------

## 3️⃣ AI Pipeline 变成三段式结构

```text
VI → VPSS → RKNN → PostProcess
```

------

## 4️⃣ 真正 AI 系统 = 推理 + 后处理

不是 RKNN 本身。

------

# 十一、YOLO 在 RKMPI 中的位置

完整结构：

```text
Camera
  │
VI
  │
VPSS
  │
RKNN
  │
YOLO Decode
  │
NMS
  │
OSD / Display
```

------

# 十二、为什么必须理解后处理？

因为工业项目中：

> **80% bug 出在后处理，而不是 RKNN**

典型问题：

- 框偏移
- 类别错乱
- 置信度异常
- 坐标映射错误

------

# 十三、本章总结

------

## ✔ ① RKNN 只负责推理，不负责检测框

输出是 Tensor，不是 bbox。

------

## ✔ ② YOLO 推理分两部分

- RKNN（forward）
- CPU（decode + NMS）

------

## ✔ ③ 输入必须统一格式（640×640）

VPSS 或 CPU 必须做预处理。

------

## ✔ ④ AI Camera = Pipeline + 推理 + 后处理

```text
VI → VPSS → RKNN → YOLO Decode → Display
```

------

# 十四、下一章预告

> 第11章 YOLO 前处理——VPSS + RGA + CPU 混合预处理设计

下一章我们将解决一个非常关键的问题：

- 如何把 VPSS 输出的 NV12 变成 YOLO 输入
- Resize 应该放 VPSS 还是 CPU？
- Color conversion 最优方案是什么？
- 为什么工业项目几乎不用纯 CPU 预处理？

这一章将决定：

> **你的 AI Camera 性能能不能跑满**