# 第11章 YOLO 前处理——VPSS + RGA + CPU 的工业级预处理设计

> 到本章为止，我们已经完成了：
>
> - Camera 取图（VI）
> - 图像处理（VPSS）
> - RKNN 推理
> - YOLO 输出解析（后处理）
>
> 但有一个非常关键的问题还没有解决：
>
> 👉 **YOLO 输入 640×640 是怎么来的？**

------

# 一、本章目标

完成本章后，你将能够理解：

- YOLO 前处理到底在做什么
- 为什么不能直接把 VPSS 输出喂给 RKNN
- Resize / Crop / Letterbox 的正确位置
- VPSS、RGA、CPU 各自负责什么
- 工业级 AI Camera 的前处理架构设计

------

# 二、AI Pipeline 再次升级

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

但真实工业系统是：

```text
Camera
  │
VI
  │
VPSS
  │
Preprocess（前处理）
  │
RKNN
  │
Postprocess
```

------

# 三、YOLO 前处理到底做什么？

YOLO 前处理核心只有三件事：

```text
1. Resize（缩放）
2. Format Convert（格式转换）
3. Letterbox（保持比例填充）
```

------

## 举例：

Camera 输出：

```text
1920 × 1080
```

YOLO 需要：

```text
640 × 640
```

------

## ❗ 直接拉伸会出问题：

```text
1920×1080 → 640×640（拉伸变形）
```

人脸会变形，检测会错误。

------

## ✔ 正确方式：Letterbox

```text
原图
  ↓ 等比缩放
填充黑边
  ↓
640×640
```

------

# 四、前处理在哪里做？

这是整个章节最关键的问题。

------

## ❌ 方案1：CPU 前处理

```text
VPSS → CPU resize → CPU convert → RKNN
```

问题：

- CPU 占用高
- memcpy 多
- 延迟高

------

## ❌ 方案2：全部 VPSS 做

```text
VPSS → 直接 640×640 → RKNN
```

问题：

- 无 letterbox（不好做）
- 灵活性差

------

## ✔ 方案3（工业标准）：VPSS + RGA + CPU 混合

```text
VI
  ↓
VPSS（降分辨率）
  ↓
RGA（Resize + Crop）
  ↓
CPU（Letterbox + Normalize）
  ↓
RKNN
```

------

# 五、各模块职责划分（非常重要）

| 模块 | 职责                           |
| ---- | ------------------------------ |
| VPSS | 硬件缩放（粗处理）             |
| RGA  | 高质量 resize / format convert |
| CPU  | letterbox / normalize          |
| RKNN | 推理                           |

------

# 六、为什么不能只用 VPSS？

VPSS 很强，但它有局限：

------

## ❌ VPSS 不擅长：

- 精细 letterbox
- RGB float normalize
- YOLO 特定 padding 逻辑

------

## ✔ VPSS 擅长：

- 极速 resize
- crop
- multi-stream 输出

------

# 七、RGA 是什么？

RGA：

```text
Rockchip Graphics Accelerator
```

------

## 它的作用：

- 高质量 resize
- format convert
- NV12 → RGB
- crop

------

## 和 VPSS 区别：

| 模块 | 定位          |
| ---- | ------------- |
| VPSS | 视频 pipeline |
| RGA  | 图像处理单元  |

------

# 八、YOLO 标准前处理流程

------

## ✔ 标准工业流程：

```text
1920×1080 (VI)
      ↓
VPSS（缩小到1280×720）
      ↓
RGA（resize + RGB）
      ↓
CPU（letterbox 640×640）
      ↓
RKNN
```

------

# 九、Letterbox 详细逻辑（核心）

------

## 目标：

保持比例 + 不变形

------

## 步骤：

```text
1. 计算缩放比例
2. resize 原图
3. 计算 padding
4. 填充黑边
```

------

## 示例：

```text
原图：1920×1080

目标：640×640
```

------

### 缩放：

```text
scale = min(640/1920, 640/1080)
      = 0.333
```

------

### 得到：

```text
640 × 360
```

------

### padding：

```text
上下填充 = 140
```

------

# 十、CPU 前处理代码（核心）

```c
void letterbox(unsigned char* src, unsigned char* dst,
               int w, int h, int tw, int th)
{
    float scale = fmin((float)tw / w, (float)th / h);

    int nw = (int)(w * scale);
    int nh = (int)(h * scale);

    int dx = (tw - nw) / 2;
    int dy = (th - nh) / 2;

    // 1. resize（这里简化）
    // 2. copy + padding
}
```

------

# 十一、前处理性能对比

------

## ❌ CPU 全处理：

```text
CPU占用：高
延迟：高
功耗：高
```

------

## ✔ VPSS + RGA：

```text
CPU占用：低
延迟：低
功耗：低
```

------

## ✔ 工业结论：

> 前处理必须“硬件优先”

------

# 十二、完整 YOLO Pipeline（工业版）

```text
Camera
  │
VI
  │
VPSS（缩放）
  │
RGA（格式转换）
  │
CPU（letterbox）
  │
RKNN
  │
YOLO decode
  │
NMS
  │
OSD / Display
```

------

# 十三、本章核心认知

------

## ✔ ① YOLO 前处理不是简单 resize

而是：

> “保持几何关系 + 符合模型输入”

------

## ✔ ② VPSS 不是万能的

它负责：

- pipeline
- scale

但不负责：

- AI 专用预处理逻辑

------

## ✔ ③ RGA 是关键加速器

工业系统必用：

> VPSS + RGA + CPU 混合方案

------

## ✔ ④ 性能差距本质来自前处理

很多系统：

> 不是 RKNN 慢，而是前处理慢

------

# 十四、本章总结

------

## ✔ YOLO 前处理本质：

```text
让图像“变成模型能理解的输入”
```

------

## ✔ 工业标准结构：

```text
VPSS + RGA + CPU
```

------

## ✔ 性能关键点：

> 前处理决定整个 AI Camera 上限

------

# 十五、下一章预告

> 第12章 YOLO 后处理——NMS 与检测框还原实战

下一章我们将彻底解决：

- YOLO 输出 tensor 如何解码
- bounding box 如何还原到原图
- NMS 如何实现
- 如何避免“框漂移 / 错位”
- 工业级后处理优化方法

从这一章开始：

> **系统进入“可用级 AI Camera”阶段**