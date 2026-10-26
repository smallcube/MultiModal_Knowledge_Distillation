# PyTorch → libtorch

---

## 结论

可交接的只有两样，各扣一样东西：

| 交出 | 拿回 | 扣掉 |
|---|---|---|
| **`state_dict`**（权重） | 全语义 | **结构** —— C++ 侧必须手写同构类 |
| **`TorchScript`**（图） | 结构 | **状态** —— 训练模式切换失效 |

**本项目选 `state_dict`**，因为要在 C++ 里**训练**（TorchScript 的 traced 图把 `self.training` 烤成常量，BN 无法切回训练模式）。

> 注：工业界部署侧多半用 ONNX Runtime / TensorRT，不是 libtorch —— 本项目 `out/build/x64-release/` 的依赖就有约 4 GB。

---

## 为什么 `state_dict` 能跨语言

两边写出来是**同一套 zip 布局**，顶层都是「名字 → 张量」的扁平字典：

```
Python   torch.save(sd, "x.pt")          →  zip{ data.pkl(OrderedDict), data/N }
C++      OutputArchive.write(key, tensor) →  zip{ data.pkl(Dict),         data/N }
```

**不是兼容，是同构。** 所以 `InputArchive::read(key, tensor)` 对两种文件行为一致。

两个坑：

```
① 0 维张量（bn.num_batches_tracked）
   → copy_(src.reshape(target.sizes()))
② dtype 不一致 → copy_ 会抛，这类会崩，不用怕
```

---

## 「预训练」是两个东西

| | **ImageNet 预训练** | **作者发布的 teacher** |
|---|---|---|
| 谁训的 | 第三方 | 论文作者 |
| 结构 | 要查，版本可能对不上 | **就在仓库里** |
| 类别数 | 1000 | **等于任务本身** |
| 要砍头吗 | 要 | **不要** |
| 要反推架构吗 | 要 | **不要** |

本项目属于右边（`cifar_teachers/resnet32x4_vanilla`）。

取基准的正确做法不是「看权重形状去猜」，而是：

```
import 作者的模型类 → 建一个 → 读它的 state_dict
```

前者是猜，后者是同一个函数算出来的，可信度差一个量级。

---

## 「砍头」的判据

```
权重类别数 ≠ 任务类别数   ⇒   需要砍
```

本项目**不需要**：

```
teacher 的 fc.weight  =  (100, 256)
模型的    fc          =  Linear(256, 100)
                          ↑ 严丝合缝
```

---

## 验证指纹

`state_dict` 不带结构 ⇒ **「加载成功」≠「加载对了」**。可用的判据只有两个：

| 模型 | 键数 | 参数量（100 类） |
|---|---|---|
| `resnet32x4` | **206** | **7,433,860** |
| `resnet8x4` | **62** | **1,233,540** |

> 反例（某项目实际发生）：文件名 `resnet34-cifar10-distill.pt`，内容是 **74** 个张量 = 学生模型（教师是 218）。而 `save_checkpoint` 打印的 `saved N tensors` 本来就能当场戳穿它。

---

## 一句话

> **结构不是猜的，是取的。**
> **「加载对了没有」不靠相信，靠两个能算出来的数。**
