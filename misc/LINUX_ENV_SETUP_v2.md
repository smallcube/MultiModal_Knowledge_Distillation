# Linux 环境配置 v2.0 —— C++ / libtorch / CUDA / libjpeg-turbo

> 目标：在一台无 root 的共享服务器上，把 C++ + libtorch 工程编起来并跑通。
> 记录日期：2026-11-05
>
> **相对 v1.0（`SVDKD_CPP_LINUX_MIGRATION.md`）的四处改动：**
>
> | # | v1.0 | v2.0 | 为什么 |
> |---|---|---|---|
> | 1 | CUDA 用 conda 的 `cuda-toolkit=12.6` | **官方 runfile 装到 `~/userFile/cuda_126`** | conda 是两层布局，`FindCUDA` 推不出 include；官方包是标准单层布局 |
> | 2 | 只有 libtorch | **新增 libjpeg-turbo 3.2.1（源码编）** | 项目要自己 `#include <jpeglib.h>` |
> | 3 | cuDNN 用**软链**进 `libtorch/lib` | **改成拷贝**（`cp -L`） | 软链会断，且断链的报错和"缺文件"一模一样，排查成本高 |
> | 4 | `CUDA_PATH` / PATH | 收敛成 **`CUDA_PATH` + `CUDA_HOME` + PATH** | `CUDA_HOME` 是给 `torch.utils.cpp_extension` 用的，和 CMake 认的不是同一个 |

---

## 1. 目标机器

| 项 | 值 |
|---|---|
| 主机 | `ubuntu-ESC8000A-E11`，用户 `chenzhijiang`（**无 root**） |
| CPU | 2 × AMD EPYC 7C13 64-Core → **128 物理核 / 256 逻辑线程 / 2 NUMA node** |
| GPU | 8 张，compute capability **8.9**（Ada Lovelace） |
| 内存 | 257 GB |
| 数据 | `/ssd/tengsiyi_data/datasets/ImageNet`（**共享盘**，别人也在读写） |
| 构建工具 | CMake **4.5.0-rc1** + Ninja **1.13.2**，在 `~/userFile/cmake/bin` |
| 编译器 | `/usr/bin/c++` = **GNU 11.4.0**（系统自带，不用 conda 的 gcc 13.4） |

**一条贯穿全文的原则：全部装在 `~/userFile/` 下，不用 sudo，不碰 `/usr`、`/etc`。**

---

## 2. 最终配置（结论先行）

### 2.1 目录布局

```
~/userFile/
├── cuda_126/                       CUDA Toolkit 12.6.85（官方 runfile，标准布局）
├── C++_lib/
│   ├── libtorch/                   2.14.1+cu126（含 libtorch-shared-with-deps zip）
│   └── libjpeg-turbo/              3.2.1 安装产物（include/ lib/ bin/）
├── cmake/                          CMake 4.5.0-rc1 + Ninja 1.13.2
└── miniConda3/envs/
    ├── cuda126/                    （旧，v2.0 后可弃用）
    └── compile_tools/              只装 nasm 2.16.03，平时不激活
```

### 2.2 环境变量（只有三条）

`~/.bashrc` 里：

```bash
export CUDA_PATH=$HOME/userFile/cuda_126          # CMake 的 FindCUDA 读这个
export CUDA_HOME=$HOME/userFile/cuda_126          # torch.utils.cpp_extension 读这个
export PATH=$HOME/userFile/cuda_126/bin:$HOME/userFile/cmake/bin:$PATH
```

| 变量 | 谁读 | 不要会怎样 |
|---|---|---|
| `CUDA_PATH` | CMake 的 `FindCUDA.cmake` | 得在 CMakeLists 里手写 `CUDA_TOOLKIT_ROOT_DIR` |
| `CUDA_HOME` | `torch.utils.cpp_extension` | `CUDA_HOME` 为 `None` → "No CUDA runtime is found" |
| `PATH` 里的 `cuda_126/bin` | nvcc / **cudafe++** | `sh: 1: cudafe++: not found`（nvcc 内部用裸名字调它，`-I` 补不了） |

**`CUDA_PATH` 和 `CUDA_HOME` 不是同一个变量**，这是最容易混的一处。

### 2.3 CMakeLists 的依赖块

```cmake
# ---------------- libtorch ----------------
if(WIN32)
    set(LIBTORCH_ROOT "D:/DATA/C++_lib/libtorch_cu126")
    # Windows 不写 CUDA 路径：安装器已设 CUDA_PATH，FindCUDA 自己会读
else()
    set(LIBTORCH_ROOT "/home/chenzhijiang/userFile/C++_lib/libtorch")
    set(CUDA_TOOLKIT_ROOT_DIR "/home/chenzhijiang/userFile/cuda_126")
endif()

list(APPEND CMAKE_PREFIX_PATH "${LIBTORCH_ROOT}")
find_package(Torch REQUIRED)

# ---------------- libjpeg-turbo ----------------
if(WIN32)
    set(LIBJPEG_TURBO_ROOT "D:/DATA/C++_lib/libjpeg-turbo")
else()
    set(LIBJPEG_TURBO_ROOT "/home/chenzhijiang/userFile/C++_lib/libjpeg-turbo")
endif()

list(APPEND CMAKE_PREFIX_PATH "${LIBJPEG_TURBO_ROOT}")   # 必须在 find_package 之前
find_package(JPEG REQUIRED)
```

**Linux 分支现在只要两行**（`LIBTORCH_ROOT` + `CUDA_TOOLKIT_ROOT_DIR`）。v1.0 里的

```cmake
set(CMAKE_CUDA_FLAGS "-I${CUDA_TOOLKIT_ROOT_DIR}/include" CACHE STRING "CUDA flags")
```

**可以删掉** —— 它是给 conda 的两层布局打的补丁，官方 runfile 装出来的标准布局不需要。

### 2.4 构建

```bash
cd ~/userFile/myProjects/CPP/<工程名>
cmake --preset linux-relwithdebinfo
cmake --build out/build/linux-relwithdebinfo -j
```

（`cmake --build --preset xxx` 需要 `CMakePresets.json` 里有 `buildPresets` 条目；只写了 `configurePresets` 的话必须给路径 —— 原因见 §2.5。）

### 2.5 CMakePresets 说明

`CMakePresets.json`（schema `version: 3`）里有 7 个 `configurePresets`：

| preset | 平台 | 继承 / 条件 | build type | 用途 |
|---|---|---|---|---|
| `windows-base` | Windows | —（`hidden`） | — | x64/x86 共用基底：Ninja + `cl.exe` |
| `windows-x64` | Windows | `windows-base`（`hidden`） | — | 加 `architecture: x64` |
| **`x64-release`** | Windows | `windows-x64` | **Release** | **Windows 日常用这个** |
| `windows-x86` | Windows | `windows-base`（`hidden`） | — | 32 位基底 |
| `x86-release` | Windows | `windows-x86` | Release | 基本不用 |
| **`linux-relwithdebinfo`** | Linux | 独立，`condition` 限 Linux | **RelWithDebInfo** | **服务器上用这个** |
| `macos-debug` | macOS | 独立，`condition` 限 Darwin | Debug | 没维护 |

三个字段值得记：

- **`binaryDir: ${sourceDir}/out/build/${presetName}`** —— 每个 preset 一个独立构建目录，各有自己的 `CMakeCache.txt`，互不干扰
- **`condition`** —— Windows 上 `linux-relwithdebinfo` 自动失效，反之亦然。所以同一个文件可以跨平台共存，不用维护两份
- **`hidden: true` + `inherits`** —— 基底不进 IDE 列表，只有具体的 preset 才出现

**Linux preset 没有指定编译器**，所以用 `PATH` 上的 `/usr/bin/c++`（11.4）。Windows 基底里显式写了 `cl.exe`，因为 MSVC 的生成器需要它 —— 这就是"同一份 presets 在两个平台上编译器来源不同"的原因。

#### 为什么 Linux 用 RelWithDebInfo 而不是 Release

排查死锁/崩溃时，并行堆栈里全是裸地址就白查了。`RelWithDebInfo` 是 `-O2 -g`：优化不降，符号全在。

（代价：构建目录大一点。但 Linux 靠 **RPATH** 引用 libtorch，不拷贝库，所以 `out/build/linux-relwithdebinfo` 只有几十 MB。Windows 那边 `POST_BUILD` 要拷 **3.86 GB** 的 DLL —— 所以 Windows 上只保留一个构建目录。）

#### 没有 `buildPresets`

当前文件里只有 `configurePresets`，所以：

```bash
cmake --build --preset linux-relwithdebinfo     # ❌ 找不到 preset
cmake --build out/build/linux-relwithdebinfo    # ✅ 必须给路径
```

想省掉路径就补一段（`buildPresets` 需要 schema `version >= 2`，当前是 3，够）：

```json
"buildPresets": [
    { "name": "x64-release",          "configurePreset": "x64-release" },
    { "name": "linux-relwithdebinfo", "configurePreset": "linux-relwithdebinfo" }
]
```

之后 `cmake --build --preset linux-relwithdebinfo` 就能用了。

---

## 3. 安装流程（按实际执行顺序）

| # | 步骤 | 命令 / 要点 |
|---|---|---|
| 0 | 确认磁盘 | `df -h ~` ，要留 20 GB |
| 1 | CMake + Ninja | 解压到 `~/userFile/cmake`，PATH 加 `bin` |
| 2 | libtorch | 解压到 `~/userFile/C++_lib/libtorch` |
| 3 | **CUDA Toolkit** | 官方 runfile → 见 §3.1 |
| 4 | **cuDNN** | 从 PyPI wheel 取 → 见 §3.2 |
| 5 | **libjpeg-turbo** | 源码编 → 见 §3.3 |
| 6 | `.bashrc` | 三条环境变量，见 §2.2 |
| 7 | configure | `cmake --preset linux-relwithdebinfo` |
| 8 | build | `cmake --build out/build/linux-relwithdebinfo -j` |
| 9 | 验收 | 见 §5 |

### 3.1 CUDA Toolkit —— 官方 runfile

```bash
# 下载：developer.nvidia.com/cuda-12-6-0-download-archive
#   Linux → x86_64 → Ubuntu 20.04 → runfile (local)，约 4 GB

cd ~/userFile
sh cuda_12.6.3_*_linux.run --silent --toolkit \
   --toolkitpath=$HOME/userFile/cuda_126 --no-opengl-libs
```

- **只要 `--toolkit`**：不装驱动（驱动是系统已经有的），不需要 root
- `--no-opengl-libs`：OpenGL 库属于驱动组件，`--toolkit` 本来就不装；写上只是把意图写明
- **装之前必须确认 `which g++` 是 `/usr/bin/g++` 11.4** —— CUDA 12.6 支持到 gcc 13.2，conda 的 13.4 会被判 unsupported

装完约 7 GB。

### 3.2 cuDNN —— 从 PyPI wheel 取

Linux 版 libtorch **不带 cuDNN**（PyTorch 在 Linux 上把 CUDA 原生栈拆成独立可分发包）。缺了它，运行时报：

```
Unable to load any of {libcudnn_graph.so.9.10.2, libcudnn_graph.so.9.10, libcudnn_graph.so.9, ...}
```

从 `nvidia-cudnn-cu12` 的 wheel 里取（**wheel 就是 zip**，不用 pip、不用 root）：

```bash
cd ~/userFile/C++_lib
W=nvidia_cudnn_cu12-9.10.2.21-py3-none-manylinux_2_27_x86_64.whl
unzip -q $W -d cudnn-wheel
ls -l cudnn-wheel/nvidia/cudnn/lib/          # 8 个真实文件，共约 1.0 GB

L=~/userFile/C++_lib/libtorch/lib
rm -f $L/libcudnn_*.so.9                     # 只清那 7 个（有下划线的）
cp -L ~/userFile/C++_lib/cudnn-wheel/nvidia/cudnn/lib/libcudnn*.so.9 $L/
```

三个要点：

1. **`cp -L`**：跟随链接拷真实内容。不带 `-L`（或用了 `-a`/`-d`）会把软链复制过去，在新位置变成断链
2. 通配符 `libcudnn_*.so.9` **不会**匹配 `libcudnn.so.9`（那个 125 KB 的调度器，没下划线）
3. wheel 里 **8 个都是真实文件**，包括调度器本身 —— 所以直接 `cp` 全部即可

拷完 `cudnn-wheel` 可以删（v1.0 的软链方案则必须留着）。

### 3.3 libjpeg-turbo —— 源码编

```bash
conda create -n compile_tools -c conda-forge nasm -y     # nasm 2.16.03

cd ~/userFile/C++_lib
tar xf libjpeg-turbo-3.2.1.tar.gz
cd libjpeg-turbo-3.2.1-src && mkdir -p build && cd build

PATH=$HOME/userFile/miniConda3/envs/compile_tools/bin:$PATH \
cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=$HOME/userFile/C++_lib/libjpeg-turbo \
  -DENABLE_SHARED=ON \
  ..
ninja && ninja install
```

**为什么是 3.x**：soname 对应关系

| 版本 | Linux soname | Windows 对应 |
|---|---|---|
| 2.x | `libjpeg.so.8` | `jpeg8.dll` |
| **3.x** | **`libjpeg.so.62`** | **`jpeg62.dll`** |

选 3.x 才能和 Windows 那份保持同一代 ABI。

**为什么 `PATH=...` 前缀而不是 `-DASM_NASM=`**：见 §4.2。

---

## 4. 关键报错与修法

### 4.1 `SIMD extensions disabled: could not find NASM compiler`

```
Looking for a ASM_NASM compiler - NOTFOUND
CMake Warning at simd/CMakeLists.txt:5 (message):
  SIMD extensions disabled: could not find NASM compiler.  Performance will suffer.
```

**根因**：没装 nasm。libjpeg-turbo 的性能几乎全在汇编里（IDCT、颜色转换、上下采样），关掉 SIMD 会慢 **2–4 倍**。

**修法**：装 nasm。注意两点：

1. 装完**必须删 build 目录重配** —— `ASM_NASM-NOTFOUND` 已被写进 `CMakeCache.txt`，直接重跑 cmake 不会重新搜索（`cmake -UASM_NASM` 也行，删目录更省心）
2. **`-DASM_NASM=<绝对路径>` 不生效**，必须用 `PATH=` 前缀 —— 见 §4.6

成功的标志：

```
-- CMAKE_ASM_NASM_OBJECT_FORMAT = elf64
-- CMAKE_ASM_NASM_FLAGS =  -DELF -D__x86_64__ -DPIC
-- SIMD extensions: X86_64 (WITH_SIMD = 1)
```

顺带一个间接证据：`FLOATTEST8` 会从 `no-fp-contract` 变成 `sse` —— 说明编译器特性探测也随之变了。

### 4.2 `-DASM_NASM=<路径>` 为什么不生效

试过：

```bash
cmake ... -DASM_NASM=$HOME/.../compile_tools/bin/nasm ..    # ❌ 仍然 NOTFOUND
PATH=$HOME/.../compile_tools/bin:$PATH cmake ... ..          # ✅ 生效
```

**根因**：`-D` 造出来的缓存条目类型是 `UNINITIALIZED`，而 libjpeg-turbo 里的 `find_program(ASM_NASM nasm)` 在该变量被判定为"可用"之前仍会走自己的检查流程；而且它的 `simd/CMakeLists.txt` 还有自己的验证分支（不只是"找没找到"）。`PATH` 前缀是绕开这一层、让 `find_program` 直接命中的最稳办法。

**经验**：指路程序给"程序路径"时，`PATH` 前缀 > `CMAKE_PROGRAM_PATH` > `-D<变量名>=`。前两个让 CMake 自己找到，最后一个赌变量名和内部逻辑。

### 4.3 cuDNN 文件 0 字节 / 断链

```
ls -l libtorch/lib/libcudnn_adv.so.9
lrwxrwxrwx ... libcudnn_adv.so.9 -> /home/.../cudnn-wheel/nvidia/cudnn/lib/libcudnn_adv.so.9
```

`ls -l` 对软链显示的大小是**目标路径字符串的长度**，`du` 直接显示 0 —— 都不代表内容大小。

**判断方法**：

```bash
stat -L  某文件          # -L = 跟随链接
ls -lL   某文件
readlink -f 某文件       # 打印真实目标
```

**修法**：从原始 whl 重新解一遍，然后 `cp -L` 拷真实内容（§3.2）。搬完验证：

```bash
find $L -maxdepth 1 -name 'libcudnn*.so.9' -type l    # 必须【无输出】
```

### 4.4 CMake 4.x 对老工程的兼容性

```
Compatibility with CMake < 3.5 has been removed
```

**修法**：`-DCMAKE_POLICY_VERSION_MINIMUM=3.5`。（本次 libjpeg-turbo 3.2.1 没踩到，但 CMake 4.x 上这是常见坑。）

### 4.5 Windows 能编、Linux 编不过：`std::memset is not a member of std`

```
decode.cpp:52:14: error: 'memset' is not a member of 'std'
```

**根因**：`jpeglib.h` 会包含 `<stdio.h>`。**MSVC 的 `<stdio.h>` 顺带把 `memset` 放进了 `std`；glibc 的 `<string.h>` 只声明 `::memset`，不放 `std::`。**

**修法**：在用到它的 `.cpp` 里显式

```cpp
#include <cstring>   // std::memset：MSVC 靠 <stdio.h> 间接带进来，glibc 不会
```

**这一类问题的排查方法**：扫一遍源码里所有 `std::mem*` / `std::str*` / `std::printf` / `std::toupper` / `std::sqrt`，逐个确认所属头文件有没有被直接或间接包含。

### 4.6 `find_package(JPEG)` 抓到了系统的 libjpeg

**症状**：configure 输出里

```
-- JPEG include: /usr/include
-- JPEG library: /usr/lib/x86_64-linux-gnu/libjpeg.so
```

**修法**：确认 `list(APPEND CMAKE_PREFIX_PATH "${LIBJPEG_TURBO_ROOT}")` 在 `find_package(JPEG)` **之前**。正确的输出是：

```
-- Found JPEG: /home/.../C++_lib/libjpeg-turbo/lib/libjpeg.so (found version "62")
```

`found version "62"` 这一句同时确认了三件事：找到的是自己编的那份、版本是 3.x、ABI 是 62。

---

## 5. 验收清单

```bash
# ① 编译器：必须是系统的 11.4，不是 conda 的 13.4
which c++ ; c++ --version | head -1

# ② CUDA：必须 12.6.x，且 include 在顶层（标准布局）
~/userFile/cuda_126/bin/nvcc --version | tail -2
ls ~/userFile/cuda_126/include/cuda_runtime.h

# ③ cuDNN：8 个真实文件，且 libtorch/lib 里没有软链
ls -lL ~/userFile/C++_lib/libtorch/lib/libcudnn*.so.9
find ~/userFile/C++_lib/libtorch/lib -maxdepth 1 -name 'libcudnn*.so.9' -type l   # 无输出

# ④ libjpeg：soname 必须是 62
ls -l ~/userFile/C++_lib/libjpeg-turbo/lib/libjpeg.so*
readelf -d ~/userFile/C++_lib/libjpeg-turbo/lib/libjpeg.so.62 | grep SONAME

# ⑤ 环境变量
echo "$CUDA_PATH | $CUDA_HOME" ; which nvcc

# ⑥ configure（关键四行见下）
cmake --preset linux-relwithdebinfo

# ⑦ 链接检查
readelf -d out/build/linux-relwithdebinfo/<exe> | grep -E 'RPATH|NEEDED'
ldd      out/build/linux-relwithdebinfo/<exe> | grep -i 'not found'    # 必须无输出
```

configure 期望的**关键四行**：

```
-- The CUDA compiler identification is NVIDIA 12.6.85 with host compiler GNU 11.4.0
-- Found CUDAToolkit: /home/.../cuda_126/include (found version "12.6.85")
-- PyTorch: CUDA detected: 12.6
-- Found JPEG: /home/.../C++_lib/libjpeg-turbo/lib/libjpeg.so (found version "62")
```

**第一条特别重要**：`with host compiler GNU 11.4.0` —— 说明 nvcc 的宿主编译器和 CMake 的 CXX **是同一个**，v1.0 §7 里"两个 gcc"的隐患正式消除。

两个可以无视的 warning：

```
-- USE_CUDNN is set to 0. Compiling without cuDNN support      ← libtorch 预编译包如此
-- library kineto not found.                                   ← profiler 库，release 版不带
CMake Warning ... pytorch is not compatible with `CMAKE_CUDA_ARCHITECTURES`   ← 见 §6.3
```

---

## 6. 三个机制说明

### 6.1 为什么 CUDA 从 conda 换成 runfile

| | conda 包 | 官方 runfile |
|---|---|---|
| 布局 | **两层**：`$PREFIX/targets/x86_64-linux/{include,lib}` | **单层**：`$PREFIX/{include,lib64,bin}` |
| `FindCUDA` 能否推出 include | ❌ 剥掉 `bin` 得到 `$PREFIX`，那里 `include/` 是 conda 的通用头目录（glib/freetype/curses），没有 CUDA 头 | ✅ 剥掉 `bin` → `$PREFIX/include/cuda_runtime.h` 就在 |
| nvcc 自我定位 | ❌ `_THERE_ == _HERE_ == .../bin` | ✅ 正常 |
| 需要的外部补丁 | `CUDA_TOOLKIT_ROOT_DIR` + `CMAKE_CUDA_FLAGS -I...` + PATH 上的 `cudafe++` | 只要 `CUDA_PATH` |

**conda 不是错**，它是"一条命令装好、版本可控、不碰 `/usr`"。但它的目录布局和 CMake 那个**废弃的** `FindCUDA` 模块对不上 —— 而 libtorch 的 `Caffe2Config.cmake` 写死了要用 `FindCUDA`，改不动它。

v1.0 的两行配置就是在补这个差额。换成官方 runfile 之后，补丁可以删。

### 6.2 三层 CUDA 依赖，别混

| 层 | 谁提供 | 何时需要 |
|---|---|---|
| **驱动** | 系统的 NVIDIA driver | 运行时（GPU 能用） |
| **运行时库**（cudart / cublas / nccl / cuDNN） | libtorch/lib 自带（cuDNN 除外） | 运行时 |
| **Toolkit**（nvcc / 头文件 / cudafe++） | `~/userFile/cuda_126` | **只在 configure 时**，为让 `FindCUDA` 探测通过 |

推论：工程里**没有 `.cu` 文件**时，Toolkit 版本只需要"能让 CMake 认下来"，运行时不参与。（所以 Windows 上 Toolkit 是 13.3、libtorch 是 cu126，也能跑。）

### 6.3 `CMAKE_CUDA_ARCHITECTURES` 为什么被忽略

```
CMake Warning ... pytorch is not compatible with `CMAKE_CUDA_ARCHITECTURES`
and will ignore its value.  Please configure `TORCH_CUDA_ARCH_LIST` instead.
```

- **CMake 官方机制**：`CMAKE_CUDA_ARCHITECTURES`（3.18+），`enable_language(CUDA)` 时自动探测
- **PyTorch 自己的机制**：`TORCH_CUDA_ARCH_LIST`

libtorch 要支持比 CMake 3.18 更老的版本，所以主动把前者置 OFF，按后者拼 `-gencode`。证据就在下一行：

```
-- Autodetected CUDA architecture(s):  8.9 8.9 8.9 8.9 8.9 8.9 8.9 8.9
-- Added CUDA NVCC flags for: -gencode;arch=compute_89,code=sm_89
```

**它自己探测对了**。所以这条 warning 不是配置错误，是"不用管我"。

真要控制就用 `export TORCH_CUDA_ARCH_LIST="8.9"` —— 写 CUDA 算子时它会变重要（决定编几份 device 代码，影响编译时间和二进制体积）。

---

## 7. 已知隐患（记录，暂不处理）

| 隐患 | 说明 | 触发条件 |
|---|---|---|
| **libtorch 升级后要重做 cuDNN** | 补丁是拷进 `libtorch/lib` 的，不在源包里 | 重新解压/升级 libtorch |
| **`~/userFile/cuda_126` 占 7 GB** | 含 manpages | 磁盘紧张时可删 `share/man` |
| **conda 的 `cuda126` 环境** | v2.0 后已不必要，但先留着做对照 | 新配置全通过后可 `conda env remove -n cuda126` |
| **共享盘 `/ssd` 吞吐不可控** | 别人也在读写，同一份代码两轮能差 25% | 测吞吐必须选独占时段，或改用本地盘 |
| **`TORCH_CXX_FLAGS` 为空** | `separate_arguments` 之后 `target_compile_options` 拿到空列表；ABI 双方都是 1，对得上 | 换成用 `_GLIBCXX_USE_CXX11_ABI=0` 编的 libtorch 会炸 |
| **没有 `buildPresets`** | `cmake --build --preset xxx` 用不了，得写 `out/build/<preset>` | 记不住路径时；补一段 JSON 即可 |
| **`macos-debug` 还是 `Debug`** | 唯一一个没改成 RelWithDebInfo 的 preset | 真要上 Mac 时（当前无人维护） |

---

## 8. 排查中走过的弯路

1. **`ls -l` 对软链显示的是目标字符串长度，不是文件大小。** 看到 `28` 别当"链了 28 次"，也别当"链接正常"——只证明字符串写进去了。判断可用性要 `stat -L` 或 `ls -lL`。
2. **软链的故障模式和"缺文件"的报错一模一样。** `Unable to load any of {libcudnn_graph.so.9...}` 既可能是真缺，也可能是断链。单机自用场景，**直接拷比软链省一个故障模式**。
3. **`cp -a` / `cp -d` 会保留软链**，在新位置变成断链。要拷内容必须不带这两个（或用 `-L`）。
4. **`CMakeCache.txt` 是粘性的。** 装好了 nasm 也不会重新搜 —— 怀疑它的时候 `rm -rf` 构建目录最快。
5. **`set(VAR "值" CACHE STRING "doc")` 不覆盖已有缓存值。** "改了 CMakeLists 但行为没变"经常是这个原因，要 `cmake -D...` 或 `cmake -U...`。
6. **`$ENV{...}` 是 configure 期求值的。** 所以 `conda activate` 必须在 configure **之前**，之后激活不管用。
7. **wheel 是 zip。** `unzip` 就能取内容，不需要 pip、不需要 root、不用"安装"。
8. **先找现成的，再造新的。** 缺库时先 `find / -name "<mylib>.h"` 和 `find / -name "TorchConfig.cmake"`，比重新下载省得多。
