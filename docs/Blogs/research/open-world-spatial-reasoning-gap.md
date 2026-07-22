---
title: From Indoor to Open World：MLLM 空间推理差距论文解析
description: OpenBench 的数据构建、评测结果与空间推理瓶颈
icon: material/file-document-outline
comments: true
---

# From Indoor to Open World：MLLM 空间推理差距论文解析

*2026-03-30*

论文：*From Indoor to Open World: Revealing the Spatial Reasoning Gap in MLLMs*

## 一个新的 Benchmark：OpenBench

为了尽可能消除语义上下文对模型理解空间能力判断的影响，本文章自己收集数据做了一个benchmark

关于language prior来自一篇论文Perceiving Beyond Language Priors: Enhancing Visual Comprehension and Attention in Multimodal Models

这篇论文详细讨论了language prior,并提出了增强视觉和减弱语义影响的技术方法

## 1. 数据收集

数据采集：

数据由自己搭建实验设施采集--在手推车上放置RGB相机、雷达、IMU/GPS，在校园、公园等行人视角场景由人手推进行视频收集，共收集近20小时的视频数据

数据由人工仔细审核带有时间戳

benchmark被分为三层级的：这里作者把空间理解能力划分问三个层级，并指出大多数benchmark停留在第一层，近来少数benchmark涉及到第二层

第一层Relational Reasoning，包括relative distance,reletive direction,Qualitative Ego-Motion等任务，多选题，不涉及精确距离等

第二层 Metric Reasoning,包括object localization, absolute distance...精确到米的问题任务

第三层Kinematic Reasoning：连续动态的问题比如速度

## 2. Benchmark 的建立

（1）数据处理 利用采集的数据借助 ORB- SLAM3采集每帧相机姿态；和深度地图depth map

（2）用联合解释模型解释数据（就是包括多种功能的模型）将数据最终转化成3D轨迹图

（3）将得到的参数转化成QA，因为任务分为9类，问题是均匀分布在9类的，约一类1000个，共8736个

## 3. 评估

首先，测评范围涵盖多样的、大量的模型，从头部模型到开源模型，并利用VLMEvalKit确保测评标准的相同

测评结果表示，所有的模型的能力都远远低于人类水平

闭源模型表现最好的是Gemini-2.5-Pro

开源模型表现最好的是Qwen3-VL-32B

Open Bench聚焦于空间感知和推理，基于此将任务划分为三层，并总结了在研究中对现在的MLLMs在空间推理方面弱点的一些**发现**：

1. 与人类相比，模型在空间关系这种在人类看来凭借直觉的问题上远不及人类，但在距离估算等任务上与人类差距更小，这揭示了MLLM与人类的认知差距

2. MLLM的表现呈现不同模型有不同的强项和短板，并且这种强项和短板具有家族性（比如Qwen系列就是一个家族）

3. 所有模型在变化场景下都表现不佳，这方面的弱势是统一的；文章指出接下来的研究应该重点关注动态空间推理

4. 室内空间benchmark很大可能是不准确的,是幻觉，在室外场景下失效了；首先，在VSI bench上，一个模型族的新模型表现远好于旧模型，但在OpenBench上这个差距极小；

其次，OpenBench包括距离推断这类问题，和VSIbench的问题类型相似，但是在相同类型的问题上同一模型的效果差距也很大

作者推断可能新模型直接用VSIbench做了训练，但这也反映现在的模型并没有做到通用的真正的空间理解

## 4. 实验一

作者专门做了关于大模型对语义前提的依赖：做着准备了模拟的室内场景，分为正常组和反常组，任务是测距离和物体尺寸

人类只会有微不足道的表现差异但是大模型在两项测试上性能大大降低，尤其是物体尺寸估计，这说明模型在回答这类问题时会直接略过图片推理回答一个语义上的常规答案

## 5. 实验二

识别空间推理瓶颈的距离度量任务消融实验：公式：d = ||(R·p2 + T) − p1||R，T是相机自运动参数，p1,p2是物体位置

任务是不同帧出现的污泥的距离度量，

实验结果表明：模型从图像中提炼精确几何信息的能力极差；模型没有3D几何知识，你告诉他公式，计算准确率极高，不告诉他公式，计算准确率大幅下滑；

模型的物体定位和相机自运动估计能力都很差

> 结论：真正的进步需要能够以原理性方式推断、存储和操作三维几何量的机制。

## Discussion 翻译

我们的消融研究指向几个有前景的研究方向。在开放世界环境中提升度量深度感知至关重要，因为这是空间任务中的主要瓶颈。整合显式几何表示——无论是通过三维感知架构、多视图一致性，还是混合的对称-神经推理——都可能帮助模型超越表面先验。此外，动态推理仍然明显缺乏充分探索。开发能够保持时间一致的世界模型、推理运动轨迹并利用传感器衍生的自我运动数据表示模型，这对下一步至关重要至关重要。
