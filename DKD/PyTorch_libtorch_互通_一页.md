libtorch 的加载方法用的torchsctipt，和python不互通，除非直接字节级别操作。AI还是太好用了

this->layer1 = register_module("layer1", make_layer(in_channel, 64, n, /*stride=*/1));

这里的"layer1"，就是保存时存储的名字
```
layer1.0.bn1.bias f32 64 106776 256
layer1.0.bn1.num_batches_tracked i64 0 107032 8
layer1.0.bn1.running_mean f32 64 107040 256
layer1.0.bn1.running_var f32 64 107296 256
layer1.0.bn1.weight f32 64 107552 256
layer1.0.bn2.bias f32 64 107808 256
layer1.0.bn2.num_batches_tracked i64 0 108064 8
layer1.0.bn2.running_mean f32 64 108072 256
layer1.0.bn2.running_var f32 64 108328 256
layer1.0.bn2.weight f32 64 108584 256
layer1.0.conv1.weight f32 64,32,3,3 108840 73728
layer1.0.conv2.weight f32 64,64,3,3 182568 147456
layer1.0.downsample.0.weight f32 64,32,1,1 330024 8192
layer1.0.downsample.1.bias f32 64 338216 256
layer1.0.downsample.1.num_batches_tracked i64 0 338472 8
layer1.0.downsample.1.running_mean f32 64 338480 256
layer1.0.downsample.1.running_var f32 64 338736 256
layer1.0.downsample.1.weight f32 64 338992 256
layer1.1.bn1.bias f32 64 339248 256
layer1.1.bn1.num_batches_tracked i64 0 339504 8
layer1.1.bn1.running_mean f32 64 339512 256
layer1.1.bn1.running_var f32 64 339768 256
layer1.1.bn1.weight f32 64 340024 256
layer1.1.bn2.bias f32 64 340280 256
layer1.1.bn2.num_batches_tracked i64 0 340536 8
layer1.1.bn2.running_mean f32 64 340544 256
layer1.1.bn2.running_var f32 64 340800 256
layer1.1.bn2.weight f32 64 341056 256
layer1.1.conv1.weight f32 64,64,3,3 341312 147456
layer1.1.conv2.weight f32 64,64,3,3 488768 147456
layer1.2.bn1.bias f32 64 636224 256
layer1.2.bn1.num_batches_tracked i64 0 636480 8
layer1.2.bn1.running_mean f32 64 636488 256
layer1.2.bn1.running_var f32 64 636744 256
layer1.2.bn1.weight f32 64 637000 256
layer1.2.bn2.bias f32 64 637256 256
layer1.2.bn2.num_batches_tracked i64 0 637512 8
layer1.2.bn2.running_mean f32 64 637520 256
layer1.2.bn2.running_var f32 64 637776 256
layer1.2.bn2.weight f32 64 638032 256
layer1.2.conv1.weight f32 64,64,3,3 638288 147456
layer1.2.conv2.weight f32 64,64,3,3 785744 147456
layer1.3.bn1.bias f32 64 933200 256
layer1.3.bn1.num_batches_tracked i64 0 933456 8
layer1.3.bn1.running_mean f32 64 933464 256
layer1.3.bn1.running_var f32 64 933720 256
layer1.3.bn1.weight f32 64 933976 256
layer1.3.bn2.bias f32 64 934232 256
layer1.3.bn2.num_batches_tracked i64 0 934488 8
layer1.3.bn2.running_mean f32 64 934496 256
layer1.3.bn2.running_var f32 64 934752 256
layer1.3.bn2.weight f32 64 935008 256
layer1.3.conv1.weight f32 64,64,3,3 935264 147456
layer1.3.conv2.weight f32 64,64,3,3 1082720 147456
layer1.4.bn1.bias f32 64 1230176 256
layer1.4.bn1.num_batches_tracked i64 0 1230432 8
layer1.4.bn1.running_mean f32 64 1230440 256
layer1.4.bn1.running_var f32 64 1230696 256
layer1.4.bn1.weight f32 64 1230952 256
layer1.4.bn2.bias f32 64 1231208 256
layer1.4.bn2.num_batches_tracked i64 0 1231464 8
layer1.4.bn2.running_mean f32 64 1231472 256
layer1.4.bn2.running_var f32 64 1231728 256
layer1.4.bn2.weight f32 64 1231984 256
layer1.4.conv1.weight f32 64,64,3,3 1232240 147456
layer1.4.conv2.weight f32 64,64,3,3 1379696 147456
```
python这边是self.layer1 = self._make_layer(64, n)

实际操作是

self.add_module('layer1', nn.Sequential(*layers))            # 子模块

self.register_parameter('weight', nn.Parameter(torch.ones(64)))  # 参数

self.register_buffer('running_mean', torch.zeros(64))        # 缓冲

有**语法糖兜底__setattr____getattr__ 
