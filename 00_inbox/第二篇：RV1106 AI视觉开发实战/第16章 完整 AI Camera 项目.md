# 第16章 完整 AI Camera 项目——从“技术链路”到“可运行产品”

> 到本章为止，我们已经把整条链路拆开讲完了：
>
> - V4L2 / RKMPI 采集
> - VPSS 图像处理
> - RKNN YOLO 推理
> - OSD 叠加显示
> - VENC 硬件编码
> - RTSP 推流
>
> 但这些“模块知识”还不是一个产品。
>
> 👉 本章的目标是：**把所有模块整合成一个真正可运行的 AI Camera 系统**

------

# 一、本章目标

完成本章后，你将具备：

- 完整 AI Camera 工程结构设计能力
- 多线程 pipeline 架构
- VI → VPSS → RKNN → OSD → VENC → RTSP 全链路整合
- 工业级运行逻辑（不卡顿、不掉帧）
- 可扩展 AI 视频产品基础架构

------

# 二、最终系统架构（工业级）

```
Camera
  │
ISP
  │
VI
  │
VPSS
  │
┌───────────────┬───────────────┐
│               │               │
▼               ▼               ▼
RKNN         OSD Overlay     Resize Channel
│
▼
YOLO Result
  │
  ▼
OSD Merge
  │
  ▼
VENC（H.264）
  │
  ▼
RTSP Server
  │
  ▼
Client（VLC / APP / Web）
```

------

# 三、工程级目录结构设计

一个真正可维护的 AI Camera 工程应该是：

```
ai_camera/
│
├── app/
│   ├── main.c
│   ├── pipeline.c
│   ├── pipeline.h
│
├── camera/
│   ├── vi.c
│   ├── vi.h
│
├── isp/
│   ├── isp_config.c
│
├── vpss/
│   ├── vpss.c
│   ├── vpss.h
│
├── rknn/
│   ├── yolov5.c
│   ├── preprocess.c
│   ├── postprocess.c
│
├── osd/
│   ├── osd.c
│   ├── draw.c
│
├── venc/
│   ├── venc.c
│
├── rtsp/
│   ├── rtsp_server.c
│
├── utils/
│   ├── buffer.c
│   ├── log.c
│
└── Makefile
```

------

# 四、核心设计思想（非常重要）

------

## 1. Pipeline化（数据流，而不是函数调用）

❌ 错误理解：

```
main() 里顺序调用所有函数
```

------

✅ 正确理解：

```
VI → VPSS → RKNN → OSD → VENC → RTSP
```

每一帧数据都是“流水线流动的”。

------

## 2. 多线程模型

AI Camera 必须拆线程：

```
Thread 1: VI采集
Thread 2: RKNN推理
Thread 3: OSD绘制
Thread 4: VENC编码
Thread 5: RTSP推流
```

------

## 3. RingBuffer（零拷贝核心）

避免：

- memcpy
- 数据复制

使用：

- buffer queue
- MB pool（Rockchip Media Buffer）

------

# 五、完整数据流（工程级）

```
Frame Capture (VI)
        │
        ▼
Frame Queue
        │
        ▼
VPSS Process
        │
        ├──────────────┐
        ▼              ▼
   RKNN Inference   OSD Draw
        │              │
        └──────┬───────┘
               ▼
         Frame Merge
               │
               ▼
           VENC Encode
               │
               ▼
           RTSP Stream
```

------

# 六、主程序架构（核心代码思想）

```
int main()
{
    init_vi();
    init_vpss();
    init_rknn();
    init_osd();
    init_venc();
    init_rtsp();

    start_pipeline_threads();

    while (1) {
        sleep(1);
    }
}
```

------

# 七、线程模型设计（关键）

------

## 7.1 VI线程（取图）

```
while (running) {
    frame = vi_get_frame();
    push_to_queue(frame);
}
```

------

## 7.2 AI推理线程

```
while (running) {
    frame = pop_queue();

    preprocess(frame);

    rknn_run(frame);

    postprocess(results);
}
```

------

## 7.3 OSD线程

```
while (running) {
    frame = get_ai_result_frame();

    draw_boxes(frame);

    send_to_venc(frame);
}
```

------

## 7.4 VENC线程

```
while (running) {
    frame = get_osd_frame();

    encode_h264(frame);

    push_to_rtsp(stream);
}
```

------

# 八、关键性能设计点（工业级核心）

------

## 1. Zero Copy

整个链路必须：

- 不复制图像
- 只传 buffer pointer

------

## 2. Pipeline并行

不能：

```
AI完成 → 才能取下一帧
```

必须：

```
流水线同时跑多帧
```

------

## 3. 帧同步机制

避免：

- AI帧落后
- RTSP卡顿

解决：

- timestamp对齐
- frame ID同步

------

# 九、性能目标（工业标准）

在 RV1106 上：

- 1080p@30fps
- YOLO实时检测
- CPU < 30%
- 延迟 < 200ms

------

# 十、系统启动流程

```
系统启动
  │
初始化 VI / ISP
  │
初始化 VPSS
  │
加载 RKNN 模型
  │
启动 RTSP Server
  │
启动所有线程
  │
进入运行状态
```

------

# 十一、常见工程问题（非常重要）

------

## ❌ 1. 卡顿

原因：

- 队列阻塞
- 没有pipeline并行

------

## ❌ 2. 延迟高

原因：

- buffer堆积
- GOP太大
- AI线程太慢

------

## ❌ 3. CPU爆

原因：

- OSD在CPU做
- memcpy太多

------

# 十二、系统优化路线

------

## 第一阶段（能跑）

- 单线程 pipeline

------

## 第二阶段（可用）

- 多线程拆分

------

## 第三阶段（工业级）

- zero copy
- ring buffer
- VPSS/OSD/VENC全硬件化

------

# 十三、最终成果

你将得到一个完整系统：

```
AI Camera Product
│
├── 实时视频
├── YOLO检测
├── 目标框OSD
├── H.264编码
├── RTSP远程观看
└── 工业级稳定运行
```

------

# 十四、本系列总结（核心能力闭环）

你已经完成从0到1的完整链路：

## 1. 底层能力

- V4L2 / ISP / VI

## 2. 媒体框架

- RKMPI / VPSS / MB

## 3. AI能力

- RKNN / YOLO

## 4. 视频能力

- VENC / H.264

## 5. 网络能力

- RTSP streaming

------

# 十五、最终一句话总结

> **AI Camera 的本质不是算法，而是“完整视频数据流水线工程”**