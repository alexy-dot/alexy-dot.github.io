---
title: 基于任务分类的视频理解空间推理 Benchmark 调研
description: BLINK、SpatialVID、DSR Suite、SAVVY-Bench、MVBench 与 OSI-Bench 调研笔记
icon: material/video-outline
comments: true
---

# 基于任务分类的视频理解空间推理 Benchmark 调研

这篇笔记按任务类型整理视频理解与空间推理相关的 Benchmark，覆盖 2D 视觉感知、3D/4D 动态空间推理、视听空间感知和开放世界户外评测。

## BLINK（Bench‑Ling for Image‑based Knowledge）

会议发表时间：ECCV2024，arXiv 2024(引用量：google scholar456,semanticsholar381)

作者单位：Xingyu Fu, Yushi Hu, Bangzheng Li, Yu Feng, Haoyu Wang, Xudong Lin, Dan Roth, Noah . Smith, Wei-Chiu Ma, Ranjay Krishna

University of Pennsylvania,University of Washington,Allen Institute for AI, University of California, Davis,Columbia University

网址：<https://zeyofu.github.io/blink/>

代码链接：<https://zeyofu.github.io/blink/>

数据集：<https://huggingface.co/datasets/BLINK-Benchmark/BLINK>

近似数据量：3.8k QA(question and answer)

模态：图像（单帧 / 多图）和文字QA

每样例是1张或多张 RGB 图 + 一句空间问题（如“左边物体在右边物体的哪个方向？”），无深度图、点云；适合纯 VLM 或 2D‑CNN/ViT‑based VLM

场景：2D 单帧空间关系 QA

说明： 文章强调一些人类能轻松做到但大语言模型远远不如的视觉感知能力

We introduce Blink, a new benchmark for multimodal lan- guage models (LLMs) that focuses on core visual perception abilities not found in other evaluations. Most of the Blink tasks can be solved by humans “within a blink” (e.g., relative depth estimation, visual cor- respondence, forensics detection, and multi-view reasoning).

这篇文章批判了以MMBench(2023)，Seed-bench(2023)为代表的一系列当时主流的使用多模态 LLMs 来评估感知的基准将感知与语言知识和推理混淆的问题，并提出本基准相较这些基准的合理性

## SpatialVID: A Large-Scale Video Dataset with Spatial Annotations

会议发表时间：arxiv2025,CVPR 2026（Google scholar引用18但是GitHub🌟500+）

作者单位：[Jiahao Wang](https://oiiiwjh.github.io/)1 *, [Yufeng Yuan](https://github.com/FelixYuan-YF)1 *, [Rujie Zheng](https://zrj-cn.github.io/)1 *, [Youtian Lin](https://linyou.github.io/)1, [Jian Gao](https://ygaojiany.github.io/)1, [Lin-Zhuo Chen](https://linzhuo.xyz/)1, [Yajie Bao](https://openreview.net/profile?id=~yajie_bao5)1,

[Yi Zhang](https://github.com/YeeZ93)1, [Chang Zeng](https://github.com/ozchango)1, [Yanxi Zhou](https://github.com/yxzhou217)1, [Xiao-Xiao Long](https://www.xxlong.site/index.html)1, [Hao Zhu](http://zhuhao.cc/home/)1, [Zhaoxiang Zhang](https://zhaoxiangzhang.net/)2, [Xun Cao](https://cite.nju.edu.cn/)1, [Yao Yao](https://yoyo000.github.io/)1 †

1Nanjing University  2Institute of Automation, Chinese Academy of Science

网址：<https://nju-3dv.github.io/projects/SpatialVID/>

论文：<https://arxiv.org/abs/2512.20557>

代码链接：<https://github.com/NJU-3DV/spatialVID>

Hugging Face：<https://huggingface.co/datasets/BLINK-Benchmark/BLINK>

近似数据量：2.7M clips / 7089 hrs duration 127M frames annotated / 8.79T

任务类型：真实世界视频中 3D 空间理解、摄像机运动建模、3D‑aware 视频生成与世界模拟。

模态：视频（RGB）+ 3D 几何（camera pose, depth, dynamic masks, motion instructions），无音频

每个样本是一段短视频 clip，每帧包含：RGB 图像、camera pose、depth map、动态物体 masks、motion instructions（文本/指令），无音频；适合 3D‑aware video LLM、视频扩散模型、SLAM‑style 世界建模。

场景：真实世界视频，更偏动态户外/通用场景，不是单一室内或城市专用。

- ﻿Category Tags: Rural (Traditional Village Street), Bright, Daytime, Sunny, Sparse
- ﻿﻿Motion Trends: forward translate, left translate
- ﻿﻿Scene Description: ... depicts a charming street in a traditional Swiss village, lined with wooden houses adorned with vibrant flower boxes. Two women are present: one stands near the end of the street, while the other walks a dog, ...
- ﻿﻿Camera Description: ... glides steadily forward, translating through the scene with a smooth, consistent pace. As it moves, it subtly shifts left, revealing more of the village street and its wooden houses. The motion remains steady throughout ...
- ﻿﻿Shot Summary: The camera smoothly advances down a cobbled path, flanked by charming wooden houses bursting with flowers. As it moves left, the view expands, revealing more of the village and the distant snow-capped mountains. The serene, sunlit scene unfolds with quiet elegance, capturing the essence of a timeless
- Alpine retreat

说明： 优势在于巨大的真实世界相机拍摄样本，本论文强调自己数据更适合训练泛化能力

1. 收集视频，过滤视频（留下质量达标的视频）给每个视频片段估计逐帧的 camera pose 和 depth map，有些版本还会细化动态区域遮罩和轨迹信息，再把视频内容整理成结构化字幕、场景总结、相机运动描述、运动指令等文本，让模型既能学视觉，也能学语言对齐，从大库里挑出更高质量、更适合评测的子集，比如 SpatialVID-HQ等，用来做训练、验证或比较不同方法

2. 那么用 SpatialVID-HQ等训练后如何验证呢？通过训练后看模型的三个任务指标有没有提升来<u>判断模型有没有更好的学会空间规律</u>

1. **camera-controlled video generation**，
2. **novel-view synthesis / geometric reconstruction**，
3. **pose estimation / spatial reconstruction**。

## DSR Suite: Learning to Reason in 4D: Dynamic Spatial Understanding for Vision Language Models

会议发表时间：arxiv2025, CVPR 2026

作者单位：[Shengchao Zhou](https://scholar.google.com/citations?user=Q5jj-TYAAAAJ&hl=en)1, [Yuxin Chen](https://scholar.google.com/citations?hl=en&user=dEm4OKAAAAAJ)2, [Yuying Ge](https://scholar.google.com/citations?user=hv1LiiEAAAAJ&hl=en&oi=ao)2, [Wei Huang](https://scholar.google.com/citations?user=rZVUlPAAAAAJ&hl=zh-CN)1, [Jiehong Lin](https://scholar.google.com/citations?user=eSkDBYcAAAAJ&hl=zh-CN)1, [Xiaojuan Qi](https://scholar.google.com/citations?hl=en&user=bGn0uacAAAAJ&view_op=list_works&sortby=pubdate)1,

1The University of Hong Kong  2ARC Lab, Tencent PCG

网址：<https://arxiv.org/abs/2512.20557>

代码链接：<https://github.com/TencentARC/DSR_Suite>

近似数据量：bench有1484 人工 QA（DSR‑Bench）

任务类型：4D 动态空间推理（方向/距离/速度）QA

一段视频 clip + 每帧的 camera pose、point clouds、dynamic masks、物体轨迹，以及自然语言问题（如“左车在右车前方多远处？”）。适合 4D‑aware video‑VLM 或带 3D 视频编码器的模型。

模态：视频 + 3D 几何（pose, traj, masks）

场景：偏动态空间理解/4D reasoning，通常对应真实视频中的运动场景

说明： DSR-Train is effective for improving dynamic spatial reasoning capability

*Experiments show that integrating DSR-Train and GSM into Qwen2.5-VL-7B significantly enhances its dy- namic spatial reasoning capability, while maintaining accuracy on general video understanding benchmarks.*

DSR Suite 分两部分：**DSR-Train** 和 **DSR-Bench**。

**DSR-Train**：自动生成的大规模训练数据，用来教模型学 4D 空间关系。

- 性质：训练集。
- 来源：in-the-wild 视频自动构建。
- 标注：camera pose、local point clouds、object masks、orientations、3D trajectories。
- 形式：自动生成多项选择题 QA。
- 目的：训练 VLM 的 4D 动态空间推理能力。

**DSR-Bench**：人工精炼的评测基准，用来测试模型有没有真正学会动态空间推理。

- 性质：评测集。
- 来源：同样的视频管线，但经过人工精炼。
- 标注：高质量 QA，强调视角转换、动态关系、多物体交互。
- 形式：多项选择 / 结构化答案。
- 目的：衡量模型是否真正掌握动态空间推理。

生成流程大致是：

1. 从真实视频中采样帧，提取场景信息。
2. 用视觉基础模型识别物体、分割轨迹、估计相机位姿和 3D 位置。
3. 把这些几何信息转成问题和答案模板，形成 QA。
4. 人工筛选和精炼后，形成 DSR-Bench 作为测试集

## SAVVY‑Bench（Spatial Awareness via Audio‑Visual LLMs）

会议发表时间：NeurIPS 2025

作者单位：Mingfei Chen · Zijun Cui · Xiulong Liu · Jinlin Xiang · Yang Zheng · Jingyuan Li · Eli Shlizerman

网址：<https://neurips.cc/virtual/2025/loc/san-diego/poster/115001>

代码链接：好像没公开

近似数据量：上千

任务类型：QA

每个样本是一段视频 + 同步 3D 空间音频（多声道 / 声场信息），外加自然语言问题（如“声音在摄像头的哪一侧？”）。适合多模态 AV‑VLM（视频+audio）或声源‑视觉联合模型。

模态：视频（RGB）+ 空间音频（3D audio），多模态 AV‑LLM

场景：日常生活场景，室内动态的空间推理

说明： 没找到开源数据不详细看了

## MVBench: A Comprehensive Multi‑modal Video Understanding Benchmark

会议发表时间：CVPR 2024

作者单位：Kunchang Li, Yali Wang, Yinan He, Yizhuo Li, Yi Wang, Yi Liu, Zun Wang, Jilan Xu, Guo Chen, Ping Luo, Limin Wang, Yu Qiao

网址：<https://openaccess.thecvf.com/content/CVPR2024/html/Li_MVBench_A_Comprehensive_Multi-modal_Video_Understanding_Benchmark_CVPR_2024_paper.html>

代码链接：<https://github.com/OpenGVLab/Ask-Anything>

近似数据量：20个任务每个任务数千QA

任务类型：20个复杂视频任务，准们用来衡量MLLM的**时序理解能力**

每个样本是短视频 clip + 对应音频 + 字幕 / ASR，问题覆盖 spatial、temporal、audio‑text 等；适合纯视频‑LLM 或多模态 MLLM（视频+audio+text）

模态：视频 + 音频 + 字幕

场景：通用视频理解（时序理解）

说明：从已有的公开视频数据集自动convert，包括视频本身、时序动作标注、事件标签、字幕等；根据已定义的20个任务模版把已标注的视频自动转化为QA多选题；

核心目标：评估 MLLM 的**时序理解能力**，而非仅静态空间理解。

## OSI-Bench（Open Spatial Intelligence Bench）

会议发表时间：CVPR 2026

作者单位：[Mingrui Wu](https://scholar.google.com/citations?user=Y1TtPL8AAAAJ&hl=en)1, [Zhaozhi Wang](https://scholar.google.com/citations?user=CkDanj8AAAAJ&hl=en)1, [Fangjinhua Wang](https://scholar.google.com/citations?user=ysTmrEsAAAAJ&hl=en)2, [Jiaolong Yang](https://scholar.google.com/citations?user=GuqoolgAAAAJ&hl=en)3, [Marc Pollefeys](https://scholar.google.com/citations?user=YYH0BjEAAAAJ&hl=en)2, [Tong Zhang](https://scholar.google.com/citations?user=kCy8JG8AAAAJ&hl=en)1,*

1University of Chinese Academy of Sciences, UCAS  2ETH Zürich  3Microsoft Research Asia 

网址：<https://mingrui-wu.github.io/openbench/>

论文：<https://arxiv.org/abs/2512.19683>

代码链接：<https://github.com/mingrui-wu/OSI-Bench>

近似数据量：metric‑precise 的户外数据集 + 8,736 个 QA 对

任务类型：开放世界户外空间推理 gap

行人视角相机视频 + LiDAR/IMU/GPS 等传感器提供的 3D 空间轨迹，自动构建 spatio‑temporal QA

模态：视频 + 3D 传感器（LiDAR/GPS）

场景：户外场景

说明： 为了避免在室内场景大语言模型可以通过piror knowledges推断问题，而户外场景这种依赖大大降低，模型只能更多的依赖图片理解，所以选择了户外场景
