# MultiModal_Knowledge_Distillation

# 2026-10-28 更新
深度学习的真实应用场景中存在如下问题：

（1）为了获得最强的泛化能力，研究者通常使用各种强数据增强（如mixup）、特殊训练技巧（label smoothing、dropout等），而这些技巧对student model的训练者来说经常是黑盒；

（2）在训练student model时，各种强数据增强策略通常不能带来显著增益；

（3）strong teacher model与student model存在显著的不一致（discrepancies），使用strong student通常带来反作用。 

《DIST+: Knowledge Distillation From a Stronger Adaptive Teacher》对这种现象进行了分析，发现这一种不一致主要来自distillation loss的巨大差异。据此，作者提出了放松蒸馏目标，将KL-Divergence替换为 Pearson correlation coefficient，从而提升蒸馏效果。

受以上工作的启发，我们接下来的工作：
（1）提出一种能跟强数据增强手段兼容的知识蒸馏，即使在不知道strong teacher model具体训练策略的基础上，依旧能在训练student model时能够使用各种强数据增强并带来增益；

（2）解决teacher与student之间的discrepancies，能充分利用strong teacher的知识，带来提升；

候选方案：
（1）使用SVD构建新的坐标系，在新的坐标系中进行关系蒸馏。

# 下一步工作：
（1）使用ResNet在大规模数据集（例如ImageNet）上进行知识蒸馏，建立基线；

（2）验证我们所提出的思路能否带来提升。
（3）看一下这个TOP-K蒸馏的论文《DeepKD: A Deeply Decoupled and Denoised Knowledge Distillation Trainer》


# 2026-10-14 工作列表
（1）了解logit distillation和Feature distillation，跑通基本的代码。目前使用最广的知识蒸馏工程是[mdistiller](https://github.com/megvii-research/mdistiller)，强烈建议运行其中的经典算法；

（2）阅读论文，包括经典KD论文和最近的顶会顶刊论文。

（3）了解奇异值分解SVD的数学原理以及Pytorch代码实现；


蒸馏损失简单的都仍在这里了<https://github.com/C-can-not-move/MultiKD_CPP/tree/master/src/kd>
