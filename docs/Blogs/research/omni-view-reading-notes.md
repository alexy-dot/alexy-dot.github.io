---
title: Omni-View 阅读笔记：生成与几何监督如何促进空间理解
description: Omni-View 的模型结构、两阶段训练、消融实验与空间推理能力分析
icon: material/cube-scan
comments: true
---

# Omni-View 阅读笔记

## 概述

Omni-View 在 BAGEL 的理解与图像生成能力上增加 Geometry Module，并通过联合训练图像生成、深度估计和相机姿态预测，让生成任务学到的时空与几何信息反过来改善3D空间理解。

Omni-View 包含三个部分：

* Understanding Model：输入多视角RGB图像和问题，生成文字答案;由预训练的BAGEl-7B初始化

* Texture Module(Generation Model)：根据参考图像、文字和目标相机位姿，生成新视角RGB图像。由预训练的BAGEl-7B初始化

* Geometry Module(Generation Model)：由Texture Module 的新视角图像 latent、Understanding Model 的特征、深度噪声和相机查询 token，预测深度以及相机内参和外参

    ```
    Understanding Model 多层特征 F_und
             ↓ cross-attention
    Geometry Module
    反向回传影响Understanding Model(通过F_und cross-attention),Texture Module(通过F_tex)
    Texture Module 最后一层特征 F_tex
             ↓
    Geometry Module
    ```

## 研究动机

* 现有方法的问题：现有 3D 场景理解模型通常依赖显式 3D 输入，例如点云、深度或 3D 坐标，因此空间理解能力较强，但适用范围和输入形式受到限制。另一方面，统一多模态模型通常具有图像理解与生成能力，却缺少对深度、相机姿态和跨视角几何一致性的显式建模。
* 作者的核心假设：如果生成模型在生成连续新视角时被迫学习时空变化、深度和相机运动，那么这些几何与时空信息可以通过共享的 Understanding Model 反过来改善 3D 空间理解。

## 训练方法

stage1:三个模块同时训练

Texture Module：

* 使用 Flow Matching 生成新视角图像。
* 采用<span style="color: rgb(121, 83, 227);">自回归方式</span>，生成第 n 帧时只能看到前 n-1 帧。
* 使用 Diffusion Forcing 改善多视角一致性。
* 使用 D2S，从较多参考视角逐渐减少到较少参考视角，提高单/少视图条件下的生成能力。

Geometry Module：

* 从真实图像和生成图像预测深度及相机姿态。
* 深度使用 MSE loss。
* 相机内外参使用 Huber loss。

Stage 1 的核心目的：

> 让纹理生成带来的时空建模能力，以及深度和位姿预测带来的几何能力，通过联合训练进入理解模型。

stage2:提升生成能力

* Understanding Model 不再训练。
* Geometry Module 不再接收 Understanding Model 的 cross-attention 特征,主要基于 RGB-Depth-Pose 生成链继续训练。
* 只训练 Texture Module 和 Geometry Module。
* 使用 RGB-Depth-Pose 联合学习。
* 根据单张参考图和深度重建初始点云。
* 从新视角投影点云，为纹理生成提供条件。
* Geometry Module 再从生成图像预测深度和相机姿态。

Stage 2 的核心目的：

> 让生成的新视角RGB、深度和相机姿态彼此一致。

## Generation Facilitates Understanding

论文提供的证据：

Table 4：加入 Texture 后，Appearance Order 提升约 `4.1`；

加入 Depth 后，Relative Distance 和 Object Size 明显提升；

Table 5：自回归生成后，Absolute Distance 提升 `5.8`，Appearance Order 提升 `4.4`；

Table 6：D2S 训练整体优于固定 dense、固定 sparse 和 random mask；

Table 2：完整 Omni-View 在 VSI-Bench 平均分上比 BAGEL-7B-FT 高 `9.1`。

## 结果

在 VSI-Bench 上，Omni-View-7B 平均得分为 `55.4`，BAGEL-7B-FT 为 `46.3`，提升 `9.1`。最大提升出现在 Relative Distance：`46.1 → 65.9`，提升 `19.8`。

绝对距离从 `36.3 → 46.4`，但数值题使用 MRA，不表示有 46.4% 的样本完全答对。

## 消融实验

| 表格      | 研究问题                          | 主要结论                                   |
| ------- | ----------------------------- | -------------------------------------- |
| Table 4 | Texture、Depth、Camera 各自有什么作用？ | Texture 帮助时空理解，Depth 帮助相对几何，独立模块优于共享参数 |
| Table 5 | 自回归生成是否有用？                    | 自回归明显帮助绝对距离、相对距离和出现顺序                  |
| Table 6 | D2S 是否有用？                     | 由 dense 到 sparse 优于固定或随机遮挡             |
| Table 7 | Stage 2 是否有用？                 | Stage 2 明显提升 NVS 和场景生成指标               |

## 记录

1. omni-view只会给模型输入具有时间连续性的图片,模型输入的序列长度本身会告诉它当前输入了多少帧，但论文没有显式提供真实时间戳、帧间物理时间间隔或相机运动速度。它学习的是有序多帧图像中的时空关系，而不是完整的物理时间建模,所以文章说他有spatiotemporal能力只是根据具有时间关系的图片的能力,不过说法没问题,注意文章的具体做法就好

2. omni-view关于深度的绝对深度尺度没法把握,能分辨的是相对深度.对绝对距离预测帮助有限

3. 输入信息更像是针对室内场景,相机随拍摄者走动但是物体基本上是静止的,但是室外场景可能有动态物体,这只是我觉得可能的一个问题

4. 论文主要展示了新视角生成指标和最终空间问答成绩，没有完整报告深度、相机估计的独立准确率表。这是论文证据上的一个不足

5. Stage 2 在冻结 understanding model参数 后进一步移除 geometry module 对 Fund 的 cross-attention 条件。论文未报告“保留冻结 Fund”与“完全移除 Fund”的对比，因此无法判断断开连接对生成性能是否必要。

6. Omni-View 的几何生成训练能够改善相对空间推理，但由于其深度表示缺少可靠绝对尺度，且训练目标不包含明确的物体运动度量，其能力<span style="color: rgb(121, 83, 227);"><span style="background-color: rgba(162, 138, 229, 0.5);">可能</span></span>难以迁移至 OSI-Bench 的绝对距离、位移和速度任务。

7. 实验不能只看总分，必须看每个子任务的分数变化。

8. 论文说明了上述整体流程，但**没有完整公开 Stage 2 的训练代码，也没有像 Stage 1 那样详细写出完整的 Stage 2 联合损失公式和所有实现步骤**。因此我们可以确认它联合训练 RGB、Depth、Pose 及其数据流，从而利用几何一致性约束提高跨视角场景生成质量.但不能从论文自行补出完整算法细节。

9. 关于为什么模型对绝对距离尺度帮助有限,是因为论文在训练geometry module的时候采用texture module的帧通过 Voyager数据流程合成深度图,这里深度并不来自深度感知器,合成深度缺少 absolute metric,对绝对距离帮助有限

10. understanding model,generation model不共用训练数据,作者想排除模型理解能力提升来源于记住了相同的图片信息,但严格证明理解能力是否提升需消融实验

11. Omni-View 在不需要显式3D输入的方法中表现很强，问答能力接近部分3D专用模型，但在精确3D grounding 上仍明显落后于使用点云或3D坐标的模型。这里明确体现模型的grounding 和绝对尺度不足。

### 结论

Omni-View 的核心贡献不是直接恢复一个精确的 metric 3D 世界，而是把多视角理解、视频式新视角生成、深度估计和相机姿态预测放进一个统一训练框架中。实验表明，生成和几何监督能够改善空间理解，尤其是相对距离和时空顺序任务；但它对绝对尺度、精确 3D grounding、动态物体和室外场景的能力仍缺少充分证据。
