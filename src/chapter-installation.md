# 一、安装篇：环境配置

## 1.1 快速安装

Eigen 是纯头文件库，安装完成后无需额外编译库文件。  
本教程默认示例使用 **C++17**，同时兼容 Eigen 5.0.x 的最低要求 **C++14**。

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

### 安装完成后的最小验证

建议安装后先运行一个最小示例，确认头文件路径和编译选项都正确。

```cpp
#include <Eigen/Dense>
#include <iostream>

int main() {
    Eigen::Matrix2d A;
    A << 1, 2,
         3, 4;

    std::cout << "A =\n" << A << "\n";
    std::cout << "det(A) = " << A.determinant() << "\n";
    return 0;
}
```

```bash
# Linux/macOS
g++ -std=c++17 -I/path/to/eigen -O2 -o eigen_check eigen_check.cpp

# Windows (MinGW)
g++ -std=c++17 -I"C:\path\to\eigen" -O2 -o eigen_check.exe eigen_check.cpp
```

**预期现象**：
- 程序成功编译
- 终端输出矩阵 `A`
- 输出 `det(A) = -2`

**常见失败原因**：
- `-I` 路径没有指向 Eigen 头文件根目录
- C++ 标准过低（应至少为 `C++14`）
- Windows 下路径或引号写法不正确

## 1.2 CMake 集成

下面给出两个**完整可运行**的最小 CMake 示例。

### 方式1：系统已安装 Eigen

```cmake
cmake_minimum_required(VERSION 3.16)
project(MyProject LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(myapp main.cpp)

find_package(Eigen3 5.0 REQUIRED NO_MODULE)
target_link_libraries(myapp PRIVATE Eigen3::Eigen)
```

### 方式2：使用 FetchContent 自动下载

```cmake
cmake_minimum_required(VERSION 3.16)
project(MyProject LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(FetchContent)
FetchContent_Declare(
    eigen
    GIT_REPOSITORY https://gitlab.com/libeigen/eigen.git
    GIT_TAG 5.0.1
)
FetchContent_MakeAvailable(eigen)

add_executable(myapp main.cpp)
target_link_libraries(myapp PRIVATE Eigen3::Eigen)
```

### 对应的最小 `main.cpp`

```cpp
#include <Eigen/Dense>
#include <iostream>

int main() {
    Eigen::Vector3d v(1, 2, 3);
    std::cout << "v = " << v.transpose() << "\n";
    return 0;
}
```

### 构建命令

```bash
cmake -S . -B build
cmake --build build
./build/myapp
```

**命令行直接编译示例**：

```bash
g++ -std=c++17 -I/path/to/eigen -O3 -march=native -o myapp myapp.cpp
```

## 1.3 头文件模块说明

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
