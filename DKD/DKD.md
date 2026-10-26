# DKD_CPP
正在复现<https://github.com/C-can-not-move/DKD_CPP>   
官方代码<https://github.com/megvii-research/mdistiller>

论文中的3.2节的论证部分，其实很重要，复现倒是简单了

```
// named_parameters() 递归进已注册的子模块(register_module)
// 凡是 4 维的参数，用 kaiming_normal 原地填一遍 —— 这里就是 conv。
// BN 的 weight=1 / bias=0 是 PyTorch 默认，不用重复写。
for (auto& item : named_parameters())
{
    if (item.value().dim() == 4)
    {
        torch::nn::init::kaiming_normal_(item.value(), /*a=*/0, torch::kFanOut, torch::kReLU);
    }
}
```
init::kaiming_normal_ 是原地操作，而参数是 requires_grad == true 的 leaf。按理说会报
```
a leaf Variable that requires grad is being used in an in-place operation
```
但是kaiming_normal_ 内部自己包含了 NoGradGuard

上面的初始化是取巧的内容，python的实现是
```
for m in self.modules():
    if isinstance(m, nn.Conv2d):            # ← 拿到的是【模块】
        nn.init.kaiming_normal_(m.weight, mode="fan_out", nonlinearity="relu")
```
标准写法应该是
```
for (auto& m : modules())
{
    if (auto conv = std::dynamic_pointer_cast<torch::nn::Conv2dImpl>(m.value()))
    {
        torch::nn::init::kaiming_normal_(conv->weight, /*a=*/0,torch::kFanOut, torch::kReLU);
    }
}
```
Module::modules() 提供递归遍历（含自身），而 Conv2dImpl 是多态的（继承 Module，有虚函数）

这里的抓参数操作
```
// 优化器必须抓的是这个模型的参数
// 写不出 Tensor<float>，只能写 Tensor + scalar_type() 在运行时问它。
{
	std::unordered_set<const void*> owned;
	for (const auto& g : optimizer.param_groups())
		for (const auto& t : g.params())
			owned.insert(t.data_ptr());

	for (const auto& p : model->parameters())
	{
		TORCH_CHECK(owned.count(p.data_ptr()) != 0,"train_classifier: 优化器抓的不是这个模型的参数 —— "
			                                       "要在模型构造完成之后再建优化器，参数取 model->parameters()");
	}
}
```
是为了确保优化器改变的是模型的参数

优化器是按句柄协议工作的：构造时拿走一批张量，之后只认这批。

模型那边随时可能换掉自己手里的那一批，而没有任何机制会通知优化器——所以这条“一次性建立、之后无人维护”的边，只能靠在训练函数入口对一次账。

## 欠账清单（改动这些地方之前先看一眼）

### 1. 初始化用维度代替类型
`src/model/resnet.cpp` 里的 kaiming 初始化用的是

    if (item.value().dim() == 4)      // 实际含义：这是一个 Conv2d

当前网络里两者等价，但**将来加入非卷积的 4 维参数会静默误初始化**（不报错）。
正解是照 Python 那句直译：

    for (const auto& m : modules())   // std::vector<std::shared_ptr<Module>>
        if (auto conv = std::dynamic_pointer_cast<torch::nn::Conv2dImpl>(m))
            torch::nn::init::kaiming_normal_(conv->weight, 0, torch::kFanOut, torch::kReLU);

（`Module::modules(bool include_self = true)` 在 libtorch 里返回
`std::vector<std::shared_ptr<Module>>`；`Conv2dImpl` 继承自 `Module` 且有虚函数
`clone`，所以 `dynamic_pointer_cast` 可用。）

### 2. SVD_CPP项目里面的蒸馏参数里面包含了模型，这是不对的，DKD已经改了
