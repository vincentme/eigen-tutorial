# 二、安装篇：环境配置

## 2.1 快速安装

```bash
# Linux
sudo apt-get install libeigen3-dev

# macOS
brew install eigen

# Windows (vcpkg)
vcpkg install eigen3

# 或下载源码
git clone https://gitlab.com/libeigen/eigen.git
cd eigen && git checkout 5.0.1
```

## 2.2 CMake集成

```cmake
cmake_minimum_required(VERSION 3.5)
project(MyProject)

set(CMAKE_CXX_STANDARD 17)

# 方式1：系统已安装
find_package(Eigen3 5.0 REQUIRED NO_MODULE)
target_link_libraries(myapp PRIVATE Eigen3::Eigen)

# 方式2：FetchContent自动下载（CMake 3.11+）
include(FetchContent)
FetchContent_Declare(eigen GIT_REPOSITORY https://gitlab.com/libeigen/eigen.git GIT_TAG 5.0.1)
FetchContent_MakeAvailable(eigen)
```

**编译选项**：
```bash
g++ -std=c++17 -I/path/to/eigen -O3 -march=native -o myapp myapp.cpp
```

## 2.3 头文件模块说明

| 头文件                   | 包含内容                   | 用途         | 大小估算 |
| ------------------------ | -------------------------- | ------------ | -------- |
| `<Eigen/Core>`           | 矩阵、数组、基本运算       | 最小依赖     | ~2MB     |
| `<Eigen/Dense>`          | Core + LU, QR, SVD, 特征值 | 稠密矩阵     | ~4MB     |
| `<Eigen/Geometry>`       | 旋转、四元数、变换         | 几何运算     | ~1MB     |
| `<Eigen/Sparse>`         | 稀疏矩阵                   | 大规模计算   | ~2MB     |
| `<Eigen/Eigenvalues>`    | 特征值分解                 | 单独使用     | ~1MB     |
| `<Eigen/QR>`             | QR分解                     | 单独使用     | ~0.5MB   |
| `<Eigen/SVD>`            | SVD分解                    | 单独使用     | ~0.5MB   |
| `<Eigen/LU>`             | LU分解                     | 单独使用     | ~0.5MB   |
| `<Eigen/Cholesky>`       | Cholesky分解               | 正定矩阵     | ~0.3MB   |
| `<Eigen/SparseCholesky>` | 稀疏Cholesky分解           | 稀疏正定矩阵 | ~0.5MB   |

**选择建议**：
- **最小依赖**：仅使用`<Eigen/Core>`
- **通用场景**：使用`<Eigen/Dense>`（包含大部分功能）
- **几何应用**：使用`<Eigen/Geometry>`
- **大规模计算**：使用`<Eigen/Sparse>`
- **单独功能**：按需包含特定模块（如`<Eigen/QR>`）

---
