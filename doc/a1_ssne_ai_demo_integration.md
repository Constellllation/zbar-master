# A1 `ssne_ai_demo` 集成 ZBar（项目内无 `zbar.h`）排查与修复

你当前的错误里有两类问题：

- `fatal error: include/qr_decoder.hpp: No such file or directory`
- `fatal error: zbar.h: No such file or directory`

其中第二个错误说明：**你的 demo 工程并没有可用的 ZBar 头文件/库路径**。如果项目里没有 `zbar.h`，需要先从 ZBar 源码引入依赖。

## 1) 先修复你自己的头文件包含方式

如果 `src/qr_decoder.cpp` 里写的是：

```cpp
#include "include/qr_decoder.hpp"
```

建议改成：

```cpp
#include "qr_decoder.hpp"
```

并在 CMake 中确认有：

```cmake
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)
```

---

## 2) 从官方仓库拉取 ZBar 源码

官方地址：`https://github.com/mchehab/zbar`

在 A1 Docker 容器里执行（示例放到 `third_party`）：

```bash
cd /home/smartsens_flying_chip_a1_sdk/A1_SDK_SC035HGS
mkdir -p third_party
cd third_party
git clone --depth 1 https://github.com/mchehab/zbar.git
```

拉取后，头文件在：

- `third_party/zbar/include/zbar.h`

---

## 3) 给 `ssne_ai_demo` 提供可用的 include/lib

你至少要满足下面两个条件：

1. 编译 `ssne_ai_demo` 时，编译器能搜到 `zbar.h`；
2. 链接阶段能找到目标架构的 `libzbar`（`arm` 架构，不是宿主机 x86）。

### 推荐做法（A1 SDK / Buildroot 思路）

把 zbar 作为 SDK 内部包构建并安装到 staging，然后在 `ssne_ai_demo` 里链接 staging 的 zbar。

### 临时做法（先让 demo 通过编译）

将 zbar 的头和库整理到 demo 的第三方目录，例如：

```text
app_demo/
  third_party/zbar/
    include/zbar.h
    lib/libzbar.so   (或 libzbar.a，必须是 arm 目标架构)
```

CMake 增加：

```cmake
target_include_directories(ssne_ai_demo PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${CMAKE_CURRENT_SOURCE_DIR}/third_party/zbar/include
)

target_link_directories(ssne_ai_demo PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/third_party/zbar/lib
)

target_link_libraries(ssne_ai_demo PRIVATE zbar)
```

> 注意：如果 `libzbar.so` 不是 A1 目标架构，会在链接或运行时失败。

---

## 4) 在 A1 容器内的最小排查命令

```bash
cd /home/smartsens_flying_chip_a1_sdk/A1_SDK_SC035HGS/smartsens_sdk/output/build/ssne_ai_demo
find . -name qr_decoder.hpp
cat CMakeFiles/ssne_ai_demo.dir/flags.make
find /home/smartsens_flying_chip_a1_sdk/A1_SDK_SC035HGS -name zbar.h
find /home/smartsens_flying_chip_a1_sdk/A1_SDK_SC035HGS/smartsens_sdk/output/staging -name 'libzbar*'
```

重点检查：

- `flags.make` 里是否有 zbar 的 `-I.../include`；
- staging 或你指定的目录里是否有 ARM 版 `libzbar`。

---

## 5) 重新编译

```bash
cd /home/smartsens_flying_chip_a1_sdk/A1_SDK_SC035HGS/smartsens_sdk
make ssne_ai_demo-dirclean
make ssne_ai_demo-rebuild V=1
```

如果仍失败，把 `V=1` 输出里包含 `qr_decoder.cpp.o` 和最终链接命令那两段发出来，就能继续精确定位是 `-I` 还是 `-l`/`-L` 问题。
