# 第13章 OSD 绘制——把检测结果画到图像上

> 到本章为止，我们已经完成：
>
> - RKNN YOLO 推理
> - 后处理（NMS + 坐标映射）
>
> 但 AI Camera 还缺少最后一步关键能力：
>
> 👉 **如何把“检测结果”真正画到视频画面上？**

------

# 一、本章目标

完成本章后，你将掌握：

- 什么是 OSD（On Screen Display）
- 如何在图像上绘制框、文字
- YOLO 输出如何映射到原图
- CPU / VPSS 两种 OSD 实现方式
- 将 AI 结果叠加到视频流

------

# 二、什么是 OSD？

OSD（On Screen Display）本质是：

> **在视频帧上叠加可视化信息**

在 AI Camera 中，OSD 通常用于：

- 检测框（Bounding Box）
- 类别名称（person / car）
- 置信度（0.87）
- FPS / 时间戳
- 调试信息

------

# 三、OSD 在 AI Pipeline 中的位置

完整数据流如下：

```
Camera
  │
ISP
  │
VI
  │
VPSS
  │
RKNN（YOLO）
  │
后处理（NMS + 坐标映射）
  │
OSD 绘制
  │
VENC（可选）
  │
RTSP / Display
```

👉 OSD 是 **AI结果 → 视频输出** 的最后融合点

------

# 四、YOLO检测结果长什么样？

典型 YOLO 输出：

```
typedef struct {
    int class_id;
    float score;
    int x;
    int y;
    int w;
    int h;
} bbox_t;
```

例如：

```
person 0.92  320,180,100,200
```

------

# 五、必须解决的核心问题：坐标映射

YOLO 通常是在：

- 640×640 输入上推理

但原图可能是：

- 1920×1080

所以必须做缩放：

```
scale_x = original_width / 640
scale_y = original_height / 640
```

转换后：

```
x1 = x * scale_x;
y1 = y * scale_y;
w  = w * scale_x;
h  = h * scale_y;
```

------

# 六、OSD实现方式一：CPU绘制（通用方式）

------

## 6.1 基本思路

直接在图像 buffer 上画：

- 画线（rectangle）
- 写字（bitmap font）

------

## 6.2 画框逻辑

```
void draw_rect(unsigned char *img, int width, int height,
               int x, int y, int w, int h)
{
    // 上边
    for (int i = x; i < x + w; i++)
        img[y * width + i] = 255;

    // 下边
    for (int i = x; i < x + w; i++)
        img[(y + h) * width + i] = 255;

    // 左右边同理
}
```

------

## 6.3 画文字（简化版）

通常使用：

- bitmap font
- 或 freetype（复杂）

------

## 6.4 优缺点

优点：

- 灵活
- 易调试
- 跨平台

缺点：

- CPU占用高
- 高分辨率性能差

------

# 七、OSD实现方式二：VPSS硬件OSD（推荐）

在 Rockchip RV1106（Rockchip）中：

VPSS 支持硬件叠加：

- rectangle overlay
- bitmap overlay
- alpha blending

------

## 7.1 数据路径

```
VI → VPSS → OSD Layer → 输出
```

------

## 7.2 VPSS OSD特点

- 零CPU占用
- 支持多层叠加
- 支持透明度
- 支持实时更新

------

# 八、YOLO → OSD完整流程

------

## Step 1：推理结果

```
person 0.93 (x,y,w,h)
```

------

## Step 2：映射回原图

```
x = x * scale_x;
y = y * scale_y;
```

------

## Step 3：构建OSD对象

```
osd_region_t region;

region.x = x;
region.y = y;
region.w = w;
region.h = h;
region.label = "person";
region.score = 0.93;
```

------

## Step 4：绘制

CPU 或 VPSS：

```
draw_rectangle(frame, x, y, w, h);
draw_text(frame, x, y, "person 0.93");
```

------

# 九、关键工程问题（非常重要）

------

## 9.1 为什么画框会“错位”？

原因：

- resize方式不一致
- padding没处理
- letterbox影响

------

## 9.2 YOLO坐标类型

常见三种：

- xyxy
- xywh
- normalized（0~1）

必须统一！

------

## 9.3 性能问题

如果 CPU OSD：

- 1080p + 多目标 → CPU爆

------

# 十、工业级优化方案

------

## 推荐架构：

```
RKNN
  │
NPU输出
  │
VPSS OSD（硬件）
  │
VENC
```

------

## 优化原则：

- CPU只做逻辑
- 图像处理交给 VPSS
- AI交给 NPU

------

# 十一、本章总结

OSD 的本质是：

> **把 AI 结果“视觉化”的最后一步**

你必须记住：

1. 没有 OSD → AI只是数据
2. 有 OSD → AI才是产品
3. CPU OSD → 灵活但慢
4. VPSS OSD → 高效但依赖平台

------

# 十二、本章完成后的能力

你已经具备：

- YOLO结果解析
- 坐标映射能力
- 图像叠加能力
- AI可视化基础链路

------

# 下一章预告（第14章）

👉 **VENC 硬件编码——把画好的图变成H.264视频流**

下一步我们会进入：

- H.264编码
- GOP / bitrate
- 硬件编码加速
- RTSP前置准备