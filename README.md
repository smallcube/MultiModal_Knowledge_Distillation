# MultiModal_Knowledge_Distillation

# 2026-10-28 更新
深度学习的真实应用场景中存在如下问题：为了获得最强的泛化能力，研究者通常使用各种强数据增强（如mixup）、特殊训练技巧（label smoothing、dropout等），而这些技巧对student model的训练者来说经常是黑盒。这使得：strong teacher model与student model存在显著的不一致（discrepancies），使用strong student通常带来反作用。 



# 2026-10-14 工作列表
（1）了解logit distillation和Feature distillation，跑通基本的代码。目前使用最广的知识蒸馏工程是[mdistiller](https://github.com/megvii-research/mdistiller)，强烈建议运行其中的经典算法；

（2）阅读论文，包括经典KD论文和最近的顶会顶刊论文。

（3）了解奇异值分解SVD的数学原理以及Pytorch代码实现；


蒸馏损失简单的都仍在这里了<https://github.com/C-can-not-move/MultiKD_CPP/tree/master/src/kd>
