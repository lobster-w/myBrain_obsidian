# 第15章 RTSP 推流——让 AI Camera “可远程观看”

> 到本章为止，我们已经完成：
>
> - YOLO 检测
> - OSD 画框叠加
> - VENC 硬件编码（H.264）
>
> 此时我们已经拿到了一个关键成果：
>
> 👉 **H.264 码流**
>
> 但问题是：
>
> ❗ 这个码流还“只能在本机存在”，无法远程观看
>
> 所以我们需要最后一层能力：
>
> **RTSP 推流**

------

# 一、本章目标

完成本章后，你将掌握：

- 什么是 RTSP 协议
- RTSP 在 AI Camera 中的作用
- H.264 → RTSP 封装流程
- RTSP Server 基本架构
- RV1106 上视频推流完整链路
- VLC / FFplay 拉流播放

------

# 二、RTSP 是什么？

RTSP（Real Time Streaming Protocol）：

> 一种用于实时音视频传输的网络控制协议

它本身**不传输视频数据**，而是负责：

- 建立连接
- 控制播放
- 管理媒体流

真正的数据通过：

- RTP（实时传输协议）

------

# 三、RTSP 在 AI Camera 中的作用

RTSP = **AI Camera 的“输出窗口”**

没有 RTSP：

```
AI系统 = 只能本地显示
```

有 RTSP：

```
AI系统 = 可远程监控系统
```

典型应用：

- 工业视觉监控
- AI安防摄像头
- 机器人视觉系统
- 云端视频接入

------

# 四、完整系统数据流

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
OSD
  │
VENC（H.264）
  │
RTSP Server
  │
Client（VLC / Web / App）
```

------

# 五、RTSP系统组成

一个完整 RTSP 系统通常包含：

## 5.1 Server（服务器）

负责：

- 接收 H.264码流
- 封装 RTP包
- 响应客户端请求

------

## 5.2 Client（客户端）

例如：

- VLC
- FFplay
- Web播放器

------

## 5.3 Session（会话）

控制流：

- PLAY
- PAUSE
- TEARDOWN

------

# 六、RTSP vs HTTP 视频

| 项目     | RTSP    | HTTP |
| -------- | ------- | ---- |
| 实时性   | 高      | 一般 |
| 延迟     | 低      | 较高 |
| 控制能力 | 强      | 弱   |
| 适用     | 监控/AI | 点播 |

------

# 七、RTSP在RV1106上的实现方式

在 Rockchip RV1106 上通常有三种方式：

------

## 方式1：官方 RK RTSP Demo（推荐）

- 基于 VENC
- 直接推流
- 性能最好

------

## 方式2：Live555（通用方案）

- 开源RTSP库
- 灵活
- 但复杂

------

## 方式3：GStreamer

- Pipeline方式
- 适合复杂系统

------

# 八、H.264 → RTSP 关键步骤

------

## Step 1：获取 VENC 码流

```
get_venc_stream(&stream);
```

------

## Step 2：封装 RTP Packet

```
H264 NALU → RTP包
```

------

## Step 3：RTSP Server发送

```
RTP → 网络传输
```

------

# 九、RTSP Server结构（核心）

```
init_rtsp_server();

create_session("/live");

while (1) {
    get_h264_stream();

    rtp_send_packet();
}
```

------

# 十、关键概念（非常重要）

------

## 10.1 SPS / PPS

RTSP必须先发送：

- SPS（序列参数）
- PPS（图像参数）

否则：

❌ 客户端无法解码

------

## 10.2 IDR帧

关键帧：

- 每个 GOP 起点
- 可独立解码

------

## 10.3 关键流程

```
SPS/PPS
  ↓
IDR Frame
  ↓
P Frame
  ↓
P Frame
```

------

# 十一、RTSP 推流完整流程

------

## Step 1：初始化 VENC

```
VPSS → VENC
```

------

## Step 2：创建 RTSP Server

```
rtsp_server_init();
```

------

## Step 3：创建媒体流

```
create_media_session("/live");
```

------

## Step 4：绑定 VENC 输出

```
VENC → RTSP
```

------

## Step 5：客户端连接

```
rtsp://192.168.1.xxx/live
```

------

# 十二、客户端播放

------

## VLC

```
打开网络串流
rtsp://ip/live
```

------

## FFplay

```
ffplay rtsp://ip/live
```

------

# 十三、AI Camera最终输出形态

```
Camera
  │
AI检测（RKNN）
  │
OSD绘制
  │
H.264编码（VENC）
  │
RTSP推流
  │
远程观看
```

------

# 十四、性能优化点（关键）

------

## 1. Zero Copy

避免：

- CPU memcpy
- 数据拷贝

------

## 2. VENC直接接RTSP

```
VENC → RTSP（无中间缓存）
```

------

## 3. GOP控制延迟

- GOP越小 → 延迟越低
- I帧越频繁 → 可恢复性更强

------

# 十五、常见问题（工程必踩坑）

------

## ❌ 1. 黑屏

原因：

- 没发送 SPS/PPS

------

## ❌ 2. 延迟高

原因：

- GOP过大
- buffer太多

------

## ❌ 3. 花屏

原因：

- RTP包丢失
- 帧顺序错误

------

# 十六、本章总结

RTSP 的本质：

> **把 VENC 输出的 H.264 码流变成“可远程访问的视频流”**

你必须记住：

1. RTSP ≠ 视频传输本身
2. VENC 才是数据来源
3. RTSP 只是“分发系统”
4. 没有 RTSP = AI Camera 不完整

------

# 十七、本章完成后的能力

你已经具备：

- RTSP协议理解
- H.264流封装能力
- AI视频远程推流能力
- 工业级监控系统基础

------

# 下一章预告（第16章）

👉 **完整 AI Camera 系统整合（工业级项目落地）**

你将完成：

- VI + VPSS + RKNN + OSD + VENC + RTSP 全链路整合
- 多线程优化
- 性能调优
- 工业级工程结构设计