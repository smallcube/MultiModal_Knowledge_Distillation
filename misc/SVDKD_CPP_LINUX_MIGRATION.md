# SVDKD_C++ Linux 迁移记录

> 目标：把 Windows 上已经跑通的 SVDKD_C++（CMake + libtorch）迁到 Linux 服务器并能运行。
> 记录日期：2026-10-19
> 本文只记录**实际发生过**的事：每条报错都是原文，每条修法都验证过。

---

## 1. 环境

| 项目 | 值 |
|---|---|
| 主机 | `ubuntu-ESC8000A-E11`，x86_64，用户 `chenzhijiang` |
| GPU | 8 张，compute capability **8.9**（configure 时自动探测到 8 × `8.9`） |
| CMake | **4.5.0-rc1**，`~/userFile/cmake/bin/cmake` |
| Ninja | 1.13.2，同在 `~/userFile/cmake/bin` |
| 编译器 | gcc / g++ **11.4.0**（`/usr/bin/cc`、`/usr/bin/c++`） |
| libtorch | **2.14.1+cu126**，`~/userFile/C++_lib/libtorch` |
| CUDA Toolkit | **12.6**（conda 装的），`~/userFile/miniConda3/envs/cuda126` |
| cuDNN | **9.10.2.21**（从 PyPI wheel 取），`~/userFile/C++_lib/cudnn-wheel` |
| 工程 | `~/userFile/myProjects/CPP/SVDKD_CPP` |
| 数据 | `~/userFile/dataset_all/cifar-10-batches-bin`（6 个 `.bin`） |

**一条贯穿全文的原则：全部装在 `~/userFile/` 下，不用 sudo，不碰 `/usr`、`/etc`。**

---

## 2. 最终配置（先看结论）

### 2.1 `CMakeLists.txt` 的依赖块

```cmake
if(WIN32)
    set(LIBTORCH_ROOT "D:/DATA/C++_lib/libtorch_cu126")
else()
    set(LIBTORCH_ROOT "/home/chenzhijiang/userFile/C++_lib/libtorch")
    set(CUDA_TOOLKIT_ROOT_DIR "/home/chenzhijiang/userFile/miniConda3/envs/cuda126/targets/x86_64-linux")
    set(CMAKE_CUDA_FLAGS "-I${CUDA_TOOLKIT_ROOT_DIR}/include" CACHE STRING "CUDA flags")
endif()

list(APPEND CMAKE_PREFIX_PATH "${LIBTORCH_ROOT}")
find_package(Torch REQUIRED)
```

三行各管一件事，**一行都不能省**：

| 行 | 作用 | 去掉会怎样 |
|---|---|---|
| `LIBTORCH_ROOT` | 告诉 `find_package(Torch)` 去哪找 | 直接找不到 Torch |
| `CUDA_TOOLKIT_ROOT_DIR` | 给 `FindCUDA` 一个**正确的根** | `missing: CUDA_INCLUDE_DIRS` |
| `CMAKE_CUDA_FLAGS = -I...` | 替 nvcc 补上它自己算错的头文件目录 | `cuda_runtime.h: No such file` |

> Windows 侧仍然只需要 `LIBTORCH_ROOT` 一行。这个不对称是**真实的**，原因见 §5.1。

### 2.2 `~/.bashrc`

```bash
export PATH=$HOME/userFile/miniConda3/envs/cuda126/bin:$PATH
```

该目录同时提供 `nvcc` 和 `cudafe++`（`$CONDA_PREFIX/nvvm/bin/cudafe++` 不存在）。

### 2.3 cuDNN 补齐（软链，非拷贝）

```bash
L=$HOME/userFile/C++_lib/libtorch
D=$HOME/userFile/C++_lib/cudnn-wheel/nvidia/cudnn/lib
for f in "$D"/libcudnn_*.so.9; do ln -sfn "$f" "$L/lib/"; done
```

结果：`libtorch/lib` 里原有 `libcudnn.so.9`（125,136 B，调度器）保持不动，另加 7 个软链：

```
libcudnn_adv.so.9            libcudnn_heuristic.so.9
libcudnn_cnn.so.9            libcudnn_ops.so.9
libcudnn_engines_precompiled.so.9
libcudnn_engines_runtime_compiled.so.9
libcudnn_graph.so.9
```

### 2.4 构建

```bash
cd ~/userFile/myProjects/CPP/SVDKD_CPP
cmake --preset linux-release
cmake --build out/build/linux-release -j
```

---

## 3. 迁移流程（按实际执行顺序）

| # | 步骤 | 说明 |
|---|---|---|
| 0 | 修掉两个"Windows 上隐形"的问题 | `cmake_minimum_required(VERSION 4.1)`；`target_compile_options` 必须在 `add_executable` **之后** |
| 1 | 装 CMake + Ninja 到 `~/userFile/cmake` | 解压即用，`PATH` 加 `~/userFile/cmake/bin` |
| 2 | 放 libtorch 到 `~/userFile/C++_lib/libtorch` | 2.14.1+cu126 |
| 3 | conda 建 `cuda126` 环境装 CUDA Toolkit | `conda create -n cuda126 -c conda-forge cuda-toolkit=12.6 -y` |
| 4 | `CMakeLists.txt` 写 `LIBTORCH_ROOT` + `CUDA_TOOLKIT_ROOT_DIR` + `CMAKE_CUDA_FLAGS` | 见 §2.1 |
| 5 | `~/.bashrc` 加 CUDA 的 `bin` 到 `PATH` | 见 §2.2 |
| 6 | 补 cuDNN（libtorch 的 Linux 包不带） | 见 §2.3 |
| 7 | `cmake --preset linux-release` | 通过 |
| 8 | `cmake --build out/build/linux-release -j` | `[10/10] Linking CXX executable SVDKD_C++` |
| 9 | 检查动态链接 | `ldd` 无 `not found`；`RPATH` 含 `libtorch/lib` 与 CUDA 的 `lib` |
| 10 | 运行 | `./out/build/linux-release/SVDKD_C++` |

---

## 4. 遇到的报错（按出现顺序，含原文与根因）

迁移过程中报错是**一级一级往后推进**的，不是同一个问题反复出现。每次失败点都往后挪一格，这就是进度。

### 4.1 `-- Could NOT find CUDA (missing: 四项)`

```
-- Could NOT find CUDA (missing: CUDA_TOOLKIT_ROOT_DIR CUDA_NVCC_EXECUTABLE CUDA_INCLUDE_DIRS CUDA_CUDART_LIBRARY)
CMake Warning at .../Caffe2/public/cuda.cmake:40 (message): PyTorch: CUDA cannot be found.
CMake Error at .../Caffe2/Caffe2Config.cmake:88 (message):
  Your installed Caffe2 version uses CUDA but I cannot find the CUDA libraries.
```

**链路**：`CMakeLists.txt` 的 `find_package(Torch)` → `TorchConfig.cmake:62` → `Caffe2Config.cmake:84` → `cuda.cmake:29` 的 `find_package(CUDA)` 失败 → `cuda.cmake:44` 置 `CAFFE2_USE_CUDA OFF` 并 return → `Caffe2Config.cmake:87-88` 抛 FATAL。

**根因**：服务器上**没有 CUDA Toolkit**，只有驱动。

**修法**：conda 装 `cuda-toolkit=12.6`（不是 `cudatoolkit` —— 后者不含 `nvcc`，两个名字差一个字母）。

**`FindCUDA` 的验收条件**（`upstream/FindCUDA.cmake:1069-1077`）：

```cmake
find_package_handle_standard_args(CUDA REQUIRED_VARS
    CUDA_TOOLKIT_ROOT_DIR  CUDA_NVCC_EXECUTABLE  CUDA_INCLUDE_DIRS  ${CUDA_CUDART_LIBRARY_VAR}
  VERSION_VAR CUDA_VERSION)
```

四项缺一即 `CUDA_FOUND=FALSE`。

### 4.2 `-- Found CUDA` 之后紧接着 `No CMAKE_CUDA_COMPILER could be found`

```
-- Found CUDA: /home/.../cuda126/targets/x86_64-linux (found version "12.6")
-- The CUDA compiler identification is unknown
CMake Error at .../cuda.cmake:56 (enable_language):
  No CMAKE_CUDA_COMPILER could be found.
```

**根因**：`enable_language(CUDA)` 在缓存里找不到就**去 `PATH` 里搜 `nvcc`**，而没用 `conda activate`，`PATH` 里没有。

> 注意这一句和第 4.4 节那句是**两种病**：这里是「不知道编译器在哪」，连一次编译都没发生；4.4 是「找到了，但它编不动」。

### 4.3 `missing: CUDA_INCLUDE_DIRS`

```
-- Could NOT find CUDA (missing: CUDA_INCLUDE_DIRS) (found version "12.6")
```

**根因**：conda 的 CUDA 包是**两层布局** —— 头文件在 `$PREFIX/targets/x86_64-linux/include/`，而 `FindCUDA` 只认 `$PREFIX/include/`。而 `$PREFIX/include` 是 conda 环境的**通用**头目录（glib、freetype、curses、alsa 都在里面），**没有 CUDA 头**。

**关键结论（可复用）**：即使 `nvcc` 在 `PATH` 上，`FindCUDA` 也**推不出**正确的根。`FindCUDA.cmake:686-696` 的做法是找出 `nvcc` 的路径，然后**剥掉结尾的 `bin`**：

```
$PREFIX/bin/nvcc  →  $PREFIX/bin  →  $PREFIX
```

于是 `<root>/include` 又指回那个通用头目录。**`CUDA_TOOLKIT_ROOT_DIR` 必须显式写死成 `targets/x86_64-linux`，`PATH` 帮不上这一格。**

**修法**：`set(CUDA_TOOLKIT_ROOT_DIR ".../cuda126/targets/x86_64-linux")`。这个目录本身是完整的标准布局：

```
targets/x86_64-linux/bin/nvcc              (软链 → ../../../bin/nvcc)
targets/x86_64-linux/include/cuda_runtime.h
targets/x86_64-linux/lib/libcudart.so      (软链 → libcudart.so.12.6.77)
```

### 4.4 `Compiling the CUDA compiler identification source file ... failed`

```
CMake Error at .../CMakeDetermineCompilerId.cmake:1029 (message):
  Compiling the CUDA compiler identification source file "CMakeCUDACompilerId.cu" failed.
  Compiler: /home/.../cuda126/targets/x86_64-linux/bin/nvcc
  Build flags:
  Id flags: --keep;--keep-dir;tmp -v
  The output was:
  #$ _HERE_ =/home/.../cuda126/targets/x86_64-linux/bin
  #$ _THERE_=/home/.../cuda126/targets/x86_64-linux/bin
  #$ gcc -D__CUDA_ARCH_LIST__=520 ... -include "cuda_runtime.h" -m64 ...
  <command-line>: fatal error: cuda_runtime.h: No such file or directory
```

**读法**：

1. 报错名字里有 "compiler identification"，容易误读成「找不到编译器」—— 但 `Compiler:` 那行明明打印了路径，说明它**被找到了**。这是**编译失败**。
2. `_HERE_` 和 `_THERE_` 两个值**完全相同**。`_HERE_` 该是 nvcc 所在目录，`_THERE_` 该是工具包根（上一级）。两者相等 ⇒ nvcc 把自己的根认成了 `bin` 目录本身。
3. 那一行 `gcc` 里**一个 `-I` 都没有** —— nvcc 推导出的头文件目录不存在，于是干脆不带。

**修法**：`set(CMAKE_CUDA_FLAGS "-I${CUDA_TOOLKIT_ROOT_DIR}/include" CACHE STRING "CUDA flags")`

- 必须 `CACHE`：报错原文写的是 **"the CMake cache entry `CMAKE_CUDA_COMPILER`"**，编译器身份探测读的是**缓存**；普通 `set()` 造的是普通变量，`find_program` 不会因此跳过搜索。
- 效果可见：修好后 `Build flags:` 那一栏会出现 `-I/home/.../targets/x86_64-linux/include`，gcc 那行随即通过。

### 4.5 `cudafe++: not found`

```
#$ gcc ... -include "cuda_runtime.h" ...          ← 这一步过了
#$ cudafe++ --c++17 --gnu_version=110400 ...      ← 这一行没有路径，只有名字
sh: 1: cudafe++: not found
# --error 0x7f                                     ← 127 = 命令不存在
```

**根因**：`cudafe++` 是 nvcc 的**内部工具**（把 gcc 预处理出的 `.ii` 拆成 host 与 device 两半）。nvcc 调它时**只写名字不写路径**，所以只认 `PATH`。而 `-I` 只能补头文件，补不了「可执行文件在哪」。

**修法**：把 `$CONDA_PREFIX/bin` 加进 `PATH`（里面有 `nvcc` 也有 `cudafe++`）。

### 4.6 运行时：`Unable to load any of {libcudnn_graph.so.9...}` / `Aborted`

```
[train] train [640, 3, 32, 32] Byte   val [320, 3, 32, 32] Byte
Unable to load any of {libcudnn_graph.so.9.10.2, libcudnn_graph.so.9.10, libcudnn_graph.so.9, libcudnn_graph.so}
Invalid handle. Cannot load symbol cudnnGetVersion
Aborted (core dumped)
```

**注意这不是「抢不到 GPU」**。抢不到卡是 `no CUDA-capable device is detected` / `CUDA driver version is insufficient` 这类。

**根因**：`libtorch/lib` 里只有 **`libcudnn.so.9`（125,136 B）** —— 那是 cuDNN 的**调度器（dispatcher）**。cuDNN 9 把实现拆成了多个库，这里**一个都没有**：

| 应存在 | Windows 对应文件的大小 |
|---|---|
| `libcudnn_engines_precompiled.so.9` | 513,926,688 B |
| `libcudnn_adv.so.9` | 282,445,872 B |
| `libcudnn_ops.so.9` | 126,508,576 B |
| `libcudnn_heuristic.so.9` | 56,823,328 B |
| `libcudnn_engines_runtime_compiled.so.9` | 20,201,008 B |
| `libcudnn_cnn.so.9` | 4,618,272 B |
| `libcudnn_graph.so.9` | 2,420,256 B |
| （调度器）`cudnn64_9.dll` | 266,288 B |

同版本（`2.14.1+cu126`）的 Windows 包有 8 个，Linux 包只有 1 个；两个平台解压后的大小差约 0.9–1.0 GB，正好是这一组。

**修法**：从 PyPI 的 `nvidia-cudnn-cu12==9.10.2.21` 取 Linux 的 `.so`，软链进 `libtorch/lib`（该目录已在可执行文件的 `DT_RPATH` 里，所以 `dlopen` 能找到）。

```
nvidia_cudnn_cu12-9.10.2.21-py3-none-manylinux_2_27_x86_64.whl
706,758,467 B
sha256 949452be657fa16687d0930933f032835951ef0892b37d2d53824d1a84dc97a8
```

wheel 本质是 zip，`unzip` 即可取 `nvidia/cudnn/lib/*`，**不需要 pip、不需要 root、不用装**。

> **不能**用 Windows 的 `.dll` 顶替 —— PE 格式在 Linux 上加载不了。

### 4.7 `CMAKE_CUDA_ARCHITECTURES / CMP0104`（Windows 侧，但影响 config 写法）

```
CMake Error in .../CMakeFiles/CMakeTmp/CMakeLists.txt:
  CUDA_ARCHITECTURES is empty for target "cmTC_65b40"
```

**根因**：`cmake_minimum_required()` 太低 ⇒ 策略 `CMP0104` 是 OLD ⇒ `enable_language(CUDA)` 不会自动填 `CMAKE_CUDA_ARCHITECTURES` ⇒ CUDA 的 try-compile 临时工程失败。

**修法**：`cmake_minimum_required(VERSION 4.1)`（≥3.18 即可）。

真正的机制是**缓存变量继承**：`CMAKE_CUDA_ARCHITECTURES` 是缓存变量，`try_run` 的临时工程会继承父工程的缓存变量。所以只要主工程把它填上，临时工程就有值 —— **policy 版本不影响**。不需要显式设 arch。

---

## 5. 背景说明（为什么两个平台不一样）

### 5.1 为什么 Windows 不需要配 CUDA，Linux 需要

| | 谁把 CUDA 的 `bin` 放进 `PATH` |
|---|---|
| Windows 安装器 | **永久写进系统 PATH**，并设好 `CUDA_PATH` |
| conda | **不做**，只在 `conda activate` 时临时注入 |

所以「Windows 上不用做」并不是「不需要做」，而是**已经有人做过了**。

而绝对路径（`CUDA_TOOLKIT_ROOT_DIR`）能管住 `FindCUDA` 找的三样（`bin` / `include` / `lib`），**管不住 nvcc 内部用裸名字调 `cudafe++` / `cicc` / `ptxas`** —— 那一层只认 `PATH`。两者是互补的，不是二选一。

### 5.2 为什么 Linux 的 libtorch 不带 cuDNN

`torch 2.14.1` 在 PyPI 上的 Linux 依赖是这样的：

```
cuda-toolkit[cublas,cudart,cufft,cufile,cupti,curand,cusolver,cusparse,nvjitlink,nvrtc,nvtx]==13.0.3  ; platform_system == "Linux"
nvidia-cudnn-cu13==9.24.0.43                                       ; platform_system == "Linux"
nvidia-cusparselt-cu13==0.8.1                                      ; platform_system == "Linux"
nvidia-nccl-cu13==2.30.7                                           ; platform_system == "Linux"
nvidia-nvshmem-cu13==3.4.5                                         ; platform_system == "Linux"
triton~=3.8.0                                                      ; platform_system == "Linux"
```

**每一条都带 `platform_system == "Linux"`** —— 也就是说，在 Linux 上 PyTorch 把整个 CUDA 原生栈拆成了一批独立可分发包；Windows 上没有这套拆分，所以 Windows 的包必须自己全带上。

至于「为什么 libtorch 的 zip 带了 `cublas` / `nccl` / `nvshmem` 却独独不带 cuDNN」，没有查到官方说明，只能列可能的原因（**均为推测**）：

1. **许可证**：cuDNN 长期是「需要用户自行接受 NVIDIA 条款」的组件，再分发窗口比 CUDA runtime 窄；
2. **体积**：这一组约 1 GB，其中 `engines_precompiled` 一个就 514 MB；
3. **平台惯例**：Linux 上有 apt / cuda-keyring 这条正规渠道，Windows 没有。

**对使用者的影响是确定的**：Linux 上 libtorch 缺 cuDNN 需要自己补，且 Linux 侧只有一个 `libtorch-shared-with-deps` 变体，**没有"带 cuDNN 的那个包"可选**。

---

## 6. 验收清单

```bash
# ① 配置：应出现 Found CUDA 与 CUDA 12.6.85
cmake --preset linux-release 2>&1 | head -n 40

# ② 编译：应出现 [10/10] Linking CXX executable SVDKD_C++
cmake --build out/build/linux-release -j

# ③ 动态链接：RPATH 应含 libtorch/lib 与 CUDA 的 lib
readelf -d out/build/linux-release/SVDKD_C++ | grep -E 'RPATH|RUNPATH|NEEDED'

# ④ 缺库检查：这一条应当【没有输出】
ldd out/build/linux-release/SVDKD_C++ | grep -i 'not found'

# ⑤ 运行
./out/build/linux-release/SVDKD_C++
```

期望的关键输出：

```
-- Found CUDA: .../cuda126/targets/x86_64-linux (found version "12.6")
-- The CUDA compiler identification is NVIDIA 12.6.85 with host compiler GNU 13.4.0
-- Found CUDAToolkit: .../targets/x86_64-linux/include (found version "12.6.85")
-- PyTorch: CUDA detected: 12.6
-- PyTorch: Header version is: 12.6
-- Autodetected CUDA architecture(s):  8.9 8.9 8.9 8.9 8.9 8.9 8.9 8.9
-- Added CUDA NVCC flags for: -gencode;arch=compute_89,code=sm_89
-- Configuring done
-- Generating done
```

`RPATH` 里有 `libtorch/lib`，另有四条前缀 `/lib/intel64`、`/lib/intel64_win`、`/lib/win-x64`（libtorch 的 `TorchConfig.cmake` 带来的 MKL 遗留路径），**在 Linux 上不存在、无害**。

`libcudart.so.12` **不在** `NEEDED` 列表里 —— 它是 `libtorch_cuda.so` 的间接依赖。它能被找到，是因为这里用的是 `DT_RPATH`（不是 `DT_RUNPATH`），前者对**间接依赖**也生效。

---

## 7. 已知隐患（记录，暂不处理）

| 隐患 | 说明 | 触发条件 |
|---|---|---|
| **两个 gcc** | `The CUDA compiler identification is NVIDIA 12.6.85 with **host compiler GNU 13.4.0**`，而 CMake 的 C/CXX 是 **GNU 11.4.0** | 现在工程里 0 个 `.cu`，nvcc 不参与实际编译 ⇒ 无害。一旦开始写 `.cu`，CUDA 侧宿主代码与 `.cpp` 出自不同 libstdc++，链接期可能撞 ABI |
| **libtorch 缺 cuDNN** | 已用软链补上，但补丁在 `libtorch/lib` 里，不在源包里 | 重新解压/升级 libtorch 后需重做 §2.3 |
| **nvcc 自定位错误** | `_THERE_ == _HERE_ == .../bin`，`nvcc.profile` 里的 `INCLUDES` / `PATH` 都按错的根推导 | 靠 `-I` 与 `PATH` 绕开，未根治（根治要改 conda 包里的 `nvcc.profile`，会被升级覆盖） |
| **`CMAKE_CUDA_ARCHITECTURES` 被忽略** | libtorch 的 `cuda.cmake:321-326` 会主动把该变量置 OFF，改用 `TORCH_CUDA_ARCH_LIST` | 只是 warning，无需处理 |
| **`library kineto not found`** | profiler 库，release 版 libtorch 不带 | 只是 warning，无需处理 |
| **`TORCH_CXX_FLAGS` 为空** | `separate_arguments` 之后 `target_compile_options` 拿到空列表，CMake 没报错；libtorch 官方 Linux 版默认 `_GLIBCXX_USE_CXX11_ABI=1`，gcc 默认也是 1，**对得上** | 若将来换成用 `=0` 编译的 libtorch，会在这里炸 |

---

## 8. 排查中走过的弯路（给后来人省时间）

1. **`ls -l` 对符号链接显示的大小是「目标字符串的长度」，不是文件大小。**
   看到 `28 ... include -> targets/x86_64-linux/include` 里的 `28`，不应理解为"链接了 28 次"，也不该直接当成"链接正常" —— 它只证明字符串写进去了。判断链接是否可用，要 `ls -l <link>/<具体文件>`。

2. **`ls -ld` 的第二个字段是硬链接数。**
   对目录，`硬链接数 = 2 + 直接子目录数`。`drwxrwxr-x 28 ... include` 表示这是**真目录**、有 26 个子目录 —— 绝不能 `rm` 或改名。

3. **`ln -s <target> <name>` 的语义取决于 `<name>` 是否已存在。**
   若 `<name>` 已经是目录，会在它**里面**建一个链接。相对目标（`targets/x86_64-linux/include`）只从**链接自己所在的那一级**解析 —— 位置一错就是断链。**建跳板一律用绝对路径。**

4. **`FindCUDA` 探测的头文件是 `device_functions.h`，不是 `cuda_runtime.h`。**
   见 `upstream/FindCUDA.cmake:796-805`。用错文件名去验证，会得出错误结论。

5. **CMake 报错要整段贴，尤其 `-- Could NOT find X (missing: ...)` 那一行。**
   它出现在 `message(WARNING)` **之前**，最容易在「我从 Warning 开始复制」时被切掉 —— 而那一行通常直接写明缺的是哪几项。

6. **诊断脚本不要加 `set -e`。**
   `set -e` 遇到非零返回就退出，而诊断脚本里 `ls` 一个不存在的文件正是它要报告的结果 —— 加了会静默中止，看起来像「脚本正常结束」。

---
