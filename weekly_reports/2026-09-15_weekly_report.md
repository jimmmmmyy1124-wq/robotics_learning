# 第5次进度汇报

**汇报时间：** 2026年9月15日  
**汇报人：** 袁崇皓  
**当前任务：** 论文阅读 

## 一、本周工作概述

本周围绕 **Spatial-Temporal Scene Reconstruction / Scene Memory for Robot Navigation** 方向开展文献调研，重点阅读并整理了两篇具有代表性的工作：

1. **ReMEmbR: Building and Reasoning Over Long-Horizon Spatio-Temporal Memory for Robot Navigation**（ICRA 2025）
2. **3D-Mem: 3D Scene Memory for Embodied Exploration and Reasoning**（CVPR 2025）

---

## 二、ReMEmbR 文献阅读

ReMEmbR 主要研究机器人在长时间运行过程中如何构建和利用长期时空记忆。其核心问题是：机器人会持续积累大量视觉历史，而后续查询往往同时涉及“看到了什么、在哪里、什么时候”，因此需要一种能够长期存储并高效检索时空信息的记忆机制。

该方法将系统分为 **Memory Building** 和 **Querying** 两个阶段。首先利用 VLM/Video Captioner 将机器人历史视频转化为文本 Caption，并结合对应的 Position 和 Timestamp 存入 Vector Database；查询阶段由 LLM 根据用户问题调用文本、位置和时间查询，对相关 Memory 进行多轮检索和推理，最终输出答案或可用于导航的目标位置。

```text
Robot Video
    ↓
VLM / Caption
    ↓
Embedding + Position + Timestamp
    ↓
Vector Database
    ↓
LLM Retrieval & Reasoning
    ↓
Answer / Navigation Goal
```

实验方面，论文使用 NaVQA 对 Spatial、Temporal 和 Descriptive 三类问题进行评测，并在真实 **Nova Carter** 机器人上进行了导航部署。相关代码、Evaluation、ROS Bag Demo 及真实机器人示例均已开源。

**当前理解：** ReMEmbR 的核心是将长时间视觉历史转化为“带空间位置和时间信息的可检索语义记忆”，避免每次查询都重新处理完整视频历史。

---

## 三、3D-Mem 文献阅读

3D-Mem 主要研究如何为具身智能体建立一种紧凑但信息丰富的 **3D Scene Memory**，使机器人既能保存已探索区域的信息，又能利用未知区域继续主动探索和空间推理。

该方法以 **Memory Snapshot** 和 **Frontier Snapshot** 为核心。机器人从 RGB-D Observation 和 Pose 中提取物体信息，通过 Detection、Segmentation、Merging 和 Co-Visibility Clustering 形成具有代表性的 Memory Snapshots，用于描述已探索区域；同时使用 Frontier Snapshots 表示未探索区域。查询时先通过 Prefiltering 筛选与当前任务相关的 Memory，再由 VLM 判断现有信息是否足够：若足够则直接回答，否则选择 Frontier 继续探索并更新 Memory。

```text
RGB-D + Pose
    ↓
Object Detection / Merging
    ↓
Co-Visibility Clustering
    ↓
Memory Snapshots + Frontier Snapshots
    ↓
Prefiltering
    ↓
VLM Reasoning
    ↓
Answer / Continue Exploration
```

实验主要在 Habitat-Sim / HM3D 环境中完成，并在 A-EQA、EM-EQA、GOAT-Bench 等任务上验证场景问答、主动探索和长期导航能力。论文同时展示了真实机器人 Demo，官方代码也已公开。

**当前理解：** 3D-Mem 的核心是用少量高信息量的视觉 Snapshot 压缩长期场景信息，同时显式表示未探索区域，使 Scene Memory 能直接服务于推理和主动探索。

---

## 四、两篇工作的初步对比与认识

通过本周两篇论文的阅读，可以初步看到当前机器人长期 Scene Memory 研究中两种不同的技术思路：ReMEmbR 更偏 长期时空检索和时间推理；3D-Mem 更偏 3D 场景表示、记忆压缩和主动探索。

---
