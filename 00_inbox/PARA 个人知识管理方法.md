# PARA 个人知识管理方法

## 概述

PARA 是一种个人知识管理方法，由 Tiago Forte 提出。

PARA 四个字母分别代表：

| 字母 | 含义 | 作用 |
|---|---|---|
| P | Projects | 项目 |
| A | Areas | 领域 |
| R | Resources | 资源 |
| A | Archives | 归档 |

核心思想：

> 将信息按照当前用途组织，而不是按照传统学科分类。

PARA 关注的问题：

- 我现在正在做什么？
- 我长期负责什么？
- 哪些资料未来可能有用？
- 哪些内容已经结束但需要保留？


---

# PARA整体结构

```text
Vault

├── 00_inbox
│
├── 01_projects
│
├── 02_areas
│
├── 03_resources
│
└── 04_archives
```

---

# 1. Projects（项目）

## 定义

Projects 指：

> 有明确目标、有完成时间的任务。


特点：

- 有开始
- 有过程
- 有结束


例如：

```text
01_projects

├── RV1106_AI_Camera
├── Balance_Robot
├── FOC_Controller
└── Vision_System
```


---

## 项目模板

```markdown
# 项目名称


## 项目目标

为什么做？


## 系统架构


## 技术方案


## 关键问题


## 解决方案


## 经验总结


## 后续计划
```


---

## 示例

### RV1106 AI Camera

项目目标：

> 基于RV1106实现低功耗AI视觉系统。


包含：

- MIPI Camera
- VI
- VPSS
- RKNN
- VENC
- RTSP


项目结束后：

移动到：

```text
04_archives/RV1106_AI_Camera
```

---

# 2. Areas（领域）

## 定义

Areas 指：

> 长期关注、持续维护、没有明确结束时间的领域。


特点：

- 长期存在
- 不会完成
- 持续积累


例如：

```text
02_areas

├── Control_Theory
├── Embedded_Linux
├── Motor_Control
├── Computer_Vision
└── Robotics
```


---

## 项目和领域区别


项目：

```
完成RV1106 AI Camera
```

领域：

```
长期提升嵌入式Linux能力
```


项目产生的经验应该沉淀到领域。


例如：

RV1106项目：

```text
01_projects

RV1106_AI_Camera
```

产生知识：

```text
Device Tree
V4L2
DMA
Zero Copy
Linux Driver
```

归入：

```text
02_areas

Embedded_Linux
```

---

# 3. Resources（资源）

## 定义

Resources 指：

> 外部资料和参考信息。


包括：

- 书籍
- 论文
- 数据手册
- 官方文档
- 教程
- 视频


例如：

```text
03_resources

├── Books
├── Papers
├── Datasheets
└── Manuals
```


---

## Resources 与 Areas 区别


Resources：

> 别人的知识


例如：

```
永磁同步电机控制技术.pdf
```

属于：

```text
03_resources/Books
```


自己总结：

```
FOC控制原理
电流环设计经验
参数调试方法
```

属于：

```text
02_areas/Motor_Control
```

---

# 4. Archives（归档）

## 定义

Archives 指：

> 已经结束，但未来可能参考的内容。


例如：

完成：

```
RV1106_AI_Camera
```

之后：

移动：

```
04_archives

└── RV1106_AI_Camera
```


归档不是删除。

原因：

未来可能需要：

- 查看旧架构
- 查找踩坑记录
- 复用代码


---

# Inbox（输入箱）

虽然 PARA 不包含 Inbox，但实际使用中通常增加。


路径：

```text
00_inbox
```


作用：

> 所有未经整理的信息入口。


例如：

- 临时想法
- ChatGPT聊天记录
- 技术资料
- 实验记录


不要一开始纠结分类。


流程：

```text
输入

↓

00_inbox

↓

整理

↓

Projects / Areas / Resources
```

---

# PARA 与 Obsidian结合

## PARA解决：

> 文件放在哪里？


例如：

RV1106项目：

```
01_projects/RV1106_AI_Camera
```


Linux知识：

```
02_areas/Embedded_Linux
```


芯片手册：

```
03_resources/Datasheets
```


旧项目：

```
04_archives
```


---

## MOC解决：

> 如何查看知识之间的关系？


例如：

机器人知识地图：

```
Robotics_MOC

机器人

├── 感知
│   └── Computer Vision
│
├── 控制
│   ├── State Space
│   ├── LQR
│   └── MPC
│
└── 执行
    ├── FOC
    └── ODrive
```


---

## 双向链接解决：

> 知识如何连接？


例如：

RV1106：

```markdown
基于：

[[Embedded Linux]]

使用：

[[RKMPI]]

推理：

[[RKNN]]

应用：

[[Computer Vision]]
```


形成：

```
             Embedded Linux

                    |

Camera —— RV1106 —— RKNN

                    |

               Edge AI
```

---

# PARA核心原则

## 1. 不追求完美分类

现实知识不是树状结构，而是网络结构。


不要纠结：

> RV1106属于Linux还是视觉？


正确：

```
RV1106

链接：

Linux
Camera
AI
Embedded
```


---

## 2. 项目产生知识，知识反哺项目


流程：

```
项目实践

↓

发现问题

↓

总结经验

↓

沉淀到领域

↓

指导新项目
```


---

## 3. AI增强PARA


人工：

- 探索
- 判断
- 创造


AI：

- 总结
- 分类
- 建立链接
- 发现遗漏


目标：

建立：

> AI辅助的个人工程知识系统。


---

# 最终目标

不是保存更多文件。

而是：

让未来的自己能够快速恢复过去的工程状态。


理想状态：

一年后重新开发RV1106：

AI能够根据：

- 项目记录
- 代码
- Debug日志
- 知识节点

快速恢复上下文。


最终形成：

```
个人工程知识操作系统

        AI

         |

     Obsidian

         |

 PARA + MOC + Links

         |

  长期工程经验资产
```

[[AI增强个人工程知识库]]