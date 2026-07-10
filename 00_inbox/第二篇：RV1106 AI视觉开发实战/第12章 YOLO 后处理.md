# 第12章 YOLO 后处理——从 Tensor 到“真实检测框”

> 到本章为止，我们已经完成了：
>
> - Camera → VI → VPSS → RKNN → YOLO推理
> - YOLO 前处理（输入如何变成 640×640）
>
> 但现在有一个最关键的问题：
>
> 👉 **RKNN 输出的 Tensor，怎么变成“人能看懂的框”？**

------

# 一、本章目标

完成本章后，你将能够：

- 理解 YOLO 输出 Tensor 的真实含义
- 学会解码 bounding box
- 理解 confidence / class score
- 掌握 NMS（非极大值抑制）
- 完成“检测框还原到原图”

------

# 二、AI Pipeline 到这里的状态

现在系统是：

```text
Camera
  │
VI
  │
VPSS
  │
Preprocess
  │
RKNN (YOLO)
  │
Tensor Output
```

------

## ❗ 当前问题：

RKNN 输出不是框，而是：

```text
float tensor / feature map
```

------

# 三、YOLO 输出到底是什么？

以 YOLOv5 为例：

```text
[1, 25200, 85]
```

------

## 每个维度含义：

| 维度  | 含义         |
| ----- | ------------ |
| 1     | batch        |
| 25200 | anchor boxes |
| 85    | 每个预测内容 |

------

## 85 =

```text
x, y, w, h, obj_conf, class_scores(80)
```

------

# 四、YOLO 输出解析流程

YOLO 后处理分 4 步：

```text
1. Decode bbox
2. 置信度筛选
3. NMS
4. 坐标映射回原图
```

------

# 五、Step 1：Decode Bounding Box

YOLO 输出的 x,y,w,h 不是像素值，而是：

```text
相对值（0~1 或 anchor 形式）
```

------

## 转换公式：

```text
x = sigmoid(tx)
y = sigmoid(ty)
w = exp(tw) * anchor_w
h = exp(th) * anchor_h
```

------

## 得到：

```text
bbox (cx, cy, w, h)
```

------

# 六、Step 2：计算置信度

YOLO 最终置信度：

```text
score = objectness × class_probability
```

------

## 示例：

```text
obj = 0.8
person = 0.9

score = 0.72
```

------

## 过滤：

```text
if score < threshold → 丢弃
```

------

# 七、Step 3：NMS（非极大值抑制）

------

## 为什么需要 NMS？

YOLO 会产生：

```text
多个框 → 同一个目标
```

例如：

```text
Person A:
  box1
  box2
  box3
```

------

## NMS 做什么？

```text
保留最优框，删除重叠框
```

------

## IoU 计算：

```text
IoU = 交集 / 并集
```

------

## NMS流程：

```text
1. 按 score 排序
2. 选最高框
3. 删除 IoU > 阈值的框
4. 重复
```

------

# 八、Step 4：坐标映射回原图（关键）

前面我们做了：

```text
1920×1080 → 640×640
```

------

## 现在要反推：

```text
640×640 → 原图坐标
```

------

## 公式：

```text
x_original = (x - pad_x) / scale
y_original = (y - pad_y) / scale
```

------

## 如果不做这一步：

❌ 框会偏移
❌ 框会缩小或拉伸错位

------

# 九、完整后处理流程

```text
RKNN Output Tensor
        ↓
Decode bbox
        ↓
Confidence filter
        ↓
NMS
        ↓
Scale back to original image
        ↓
Final detection boxes
```

------

# 十、最小 YOLO 后处理代码（核心）

```c
typedef struct {
    float x;
    float y;
    float w;
    float h;
    float score;
    int class_id;
} box_t;
```

------

## 1️⃣ 计算 score

```c
score = obj * class_prob;
```

------

## 2️⃣ 过滤低置信度

```c
if (score < 0.5) continue;
```

------

## 3️⃣ IoU计算

```c
float iou(box_t a, box_t b)
{
    float inter = intersection(a, b);
    float uni = union(a, b);
    return inter / uni;
}
```

------

## 4️⃣ NMS核心逻辑

```c
for (int i = 0; i < n; i++) {
    if (used[i]) continue;

    for (int j = i + 1; j < n; j++) {
        if (iou(box[i], box[j]) > 0.5)
            used[j] = 1;
    }
}
```

------

# 十一、YOLO 后处理常见问题

------

## ❌ 1：框偏移

原因：

- 没做 letterbox reverse

------

## ❌ 2：框很多重复

原因：

- 没做 NMS

------

## ❌ 3：类别错误

原因：

- class index 解析错

------

## ❌ 4：框太小

原因：

- scale 计算错误

------

# 十二、完整 AI Camera 输出链路

现在系统完整变成：

```text
Camera
  │
VI
  │
VPSS
  │
Preprocess
  │
RKNN
  │
YOLO Decode
  │
NMS
  │
Coordinate Mapping
  │
OSD / Display
```

------

# 十三、本章核心认知

------

## ✔ ① RKNN 输出不是结果，是“预测数据”

必须解码。

------

## ✔ ② YOLO 后处理 = AI真正“理解阶段”

不是推理，而是解释。

------

## ✔ ③ NMS 是工业必备步骤

没有 NMS = 无法用

------

## ✔ ④ 坐标映射是最容易出错的部分

工业 bug 80% 在这里

------

# 十四、本章总结

------

## ✔ YOLO 后处理本质：

```text
把 AI 数学输出 → 人类可理解结果
```

------

## ✔ 四大步骤：

1. Decode
2. Score filter
3. NMS
4. Scale back

------

## ✔ 系统意义：

> 从“模型输出”到“真实世界理解”

------

# 十五、下一章预告

> 第13章 OSD 绘制——把检测结果画到图像上

下一章我们将完成最后一步视觉闭环：

- 如何在图像上画框
- 如何显示类别名称
- 如何叠加 FPS / 调试信息
- 如何对接 VPSS/VO/显示模块

从这一章开始：

> **AI Camera 进入“可视化闭环阶段”**