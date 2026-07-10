# 第07章 第一个 RKMPI 取图程序——跑通 VI → VPSS 数据流

> **前六章我们已经建立了完整理论体系，从本章开始进入真正的代码实践。**
>
> 本章目标只有一个：
> 👉 **让一帧图像真正从 Camera → VI → VPSS → 用户空间完整跑通**

------

# 一、本章目标

完成本章后，你将获得：

- 一个最小可运行 RKMPI 取图程序
- 成功从 VPSS 获取图像 Frame
- 理解 SYS Bind 如何真正生效
- 理解 VI → VPSS 数据流是否打通
- 输出一帧 NV12/YUV 数据用于验证

------

# 二、整体数据流回顾

本章我们要实现的最小 Pipeline：

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
GetFrame (User Space)
```

注意：

👉 **我们这一章不做 RKNN、不做编码，只做“取图验证链路”**

------

# 三、关键设计思想（非常重要）

在 RKMPI 中：

> ❗不是“调用 VPSS 去取 VI 的数据”
> ❗而是“VI 自动推数据到 VPSS”

核心是：

```text
VI 产生 Frame
   ↓
SYS Bind
   ↓
VPSS 自动接收
   ↓
用户从 VPSS 取
```

------

# 四、完整程序结构

我们先看整体结构：

```c
int main()
{
    RK_MPI_SYS_Init();

    init_vi();     // 创建 VI
    init_vpss();   // 创建 VPSS

    bind_vi_vpss(); // SYS Bind

    start_vi();
    start_vpss();

    get_frame_loop();

    unbind();
    deinit();

    RK_MPI_SYS_Exit();
}
```

------

# 五、完整代码（最小可运行版本）

> ⚠️ 这是一个“教学最小闭环版本”，用于验证数据流

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include "rk_mpi_sys.h"
#include "rk_mpi_vi.h"
#include "rk_mpi_vpss.h"

#define VI_CHN 0
#define VPSS_GRP 0
#define VPSS_CHN 0

int main()
{
    RK_S32 ret;

    printf("=== RKMPI Demo Start ===\n");

    /***********************
     * 1. SYS INIT
     ***********************/
    ret = RK_MPI_SYS_Init();
    if (ret != 0) {
        printf("SYS init failed\n");
        return -1;
    }

    /***********************
     * 2. CREATE VI
     ***********************/
    VI_CHN_ATTR_S vi_attr;
    memset(&vi_attr, 0, sizeof(vi_attr));

    vi_attr.u32Width = 1920;
    vi_attr.u32Height = 1080;
    vi_attr.enPixelFormat = RK_FMT_YUV420SP;

    ret = RK_MPI_VI_SetChnAttr(0, VI_CHN, &vi_attr);
    ret |= RK_MPI_VI_EnableChn(0, VI_CHN);

    if (ret != 0) {
        printf("VI init failed\n");
        return -1;
    }

    /***********************
     * 3. CREATE VPSS
     ***********************/
    VPSS_GRP_ATTR_S vpss_attr;
    memset(&vpss_attr, 0, sizeof(vpss_attr));

    vpss_attr.u32MaxW = 1920;
    vpss_attr.u32MaxH = 1080;
    vpss_attr.enPixelFormat = RK_FMT_YUV420SP;

    ret = RK_MPI_VPSS_CreateGrp(VPSS_GRP, &vpss_attr);
    ret |= RK_MPI_VPSS_StartGrp(VPSS_GRP);

    VPSS_CHN_ATTR_S chn_attr;
    memset(&chn_attr, 0, sizeof(chn_attr));

    chn_attr.u32Width = 1920;
    chn_attr.u32Height = 1080;
    chn_attr.enPixelFormat = RK_FMT_YUV420SP;

    ret = RK_MPI_VPSS_EnableChn(VPSS_GRP, VPSS_CHN, &chn_attr);

    if (ret != 0) {
        printf("VPSS init failed\n");
        return -1;
    }

    /***********************
     * 4. SYS BIND (核心)
     ***********************/
    MPP_CHN_S src, dst;

    src.enModId = RK_ID_VI;
    src.s32DevId = 0;
    src.s32ChnId = VI_CHN;

    dst.enModId = RK_ID_VPSS;
    dst.s32DevId = VPSS_GRP;
    dst.s32ChnId = VPSS_CHN;

    ret = RK_MPI_SYS_Bind(&src, &dst);
    if (ret != 0) {
        printf("Bind failed\n");
        return -1;
    }

    printf("Bind VI -> VPSS success\n");

    /***********************
     * 5. START STREAM
     ***********************/
    RK_MPI_VI_StartStream(0, VI_CHN);

    /***********************
     * 6. GET FRAME LOOP
     ***********************/
    VIDEO_FRAME_INFO_S frame;

    for (int i = 0; i < 10; i++) {

        ret = RK_MPI_VPSS_GetChnFrame(VPSS_GRP, VPSS_CHN, &frame, 1000);

        if (ret == 0) {
            printf("Get frame success: %d x %d\n",
                   frame.stVFrame.u32Width,
                   frame.stVFrame.u32Height);

            // 必须释放，否则 buffer 会耗尽
            RK_MPI_VPSS_ReleaseChnFrame(VPSS_GRP, VPSS_CHN, &frame);
        } else {
            printf("Get frame timeout\n");
        }

        usleep(100000);
    }

    /***********************
     * 7. CLEAN
     ***********************/
    MPP_CHN_S src_unbind, dst_unbind;

    RK_MPI_SYS_UnBind(&src, &dst);

    RK_MPI_VPSS_StopGrp(VPSS_GRP);
    RK_MPI_VPSS_DestroyGrp(VPSS_GRP);

    RK_MPI_VI_DisableChn(0, VI_CHN);

    RK_MPI_SYS_Exit();

    printf("=== RKMPI Demo End ===\n");

    return 0;
}
```

------

# 六、运行结果应该是什么？

如果一切正常，你会看到：

```text
Bind VI -> VPSS success
Get frame success: 1920 x 1080
Get frame success: 1920 x 1080
...
```

说明：

✔ Camera 已启动
✔ VI 正常输出
✔ VPSS 正常接收
✔ SYS Bind 生效
✔ Zero Copy Pipeline 已打通

------

# 七、这一章你真正学到了什么？

## 1️⃣ VI → VPSS 不是调用关系，而是数据流关系

不是：

```text
VPSS 去拿 VI
```

而是：

```text
VI 自动推送给 VPSS
```

------

## 2️⃣ SYS Bind 是真正的“开关”

没有 Bind：

```text
VI 和 VPSS 是两个孤岛
```

有 Bind：

```text
VI → VPSS 自动流动
```

------

## 3️⃣ GetFrame 只是“观察窗口”

你只是：

> 从 VPSS “看” 到流动中的数据

不是：

> 去“拉取”数据

------

## 4️⃣ MB 已经在背后工作

虽然你没看到 MB：

但每一帧：

都是：

```text
MB_BLK 在流动
```

------

# 八、常见问题（非常重要）

## ❓1：为什么 GetFrame 会超时？

可能原因：

- 没有 Start VI Stream
- Bind 没成功
- VPSS 没 Enable Channel

------

## ❓2：为什么图像是绿的？

说明：

- ISP 没完全生效
- 或 format 不匹配

------

## ❓3：为什么一定要 ReleaseFrame？

因为：

> MB 是循环复用的

不释放：

会：

```text
buffer 用尽 → 卡死
```

------

# 九、本章总结

这一章你真正完成了三件事：

------

## ✔ 1. 第一次跑通 RKMPI Pipeline

```text
VI → VPSS → User
```

------

## ✔ 2. 理解 SYS Bind 的真实作用

不是 API，而是：

> 数据通路构建器

------

## ✔ 3. 验证 Zero Copy 已经生效

数据没有 memcpy，而是在 MB 中流动

------

# 十、下一章预告

> 第08章 RKMPI 替代 V4L2

下一章我们将做一件很关键的事情：

- 对比 V4L2 取图 vs RKMPI 取图
- 分析性能差异
- 看 CPU 占用差多少
- 为什么工业项目几乎不用 V4L2 直接做 AI

这一章会让你彻底理解：

> **为什么 RKMPI 是 AI Camera 的真正工程方案，而不是 V4L2**