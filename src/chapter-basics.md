# 一、基础篇：Eigen简介与快速入门

## 1.1 什么是Eigen？

**Eigen** 是一个高性能的C++模板库，专注于线性代数、矩阵和向量运算。核心特点：

| 特性           | 说明                               |
| -------------- | ---------------------------------- |
| **纯头文件库** | 无需编译，只需包含头文件即可使用   |
| **表达式模板** | 智能优化运算，避免临时对象开销     |
| **向量化支持** | 自动使用SSE、AVX、NEON等SIMD指令集 |
| **零依赖**     | 除C++标准库外无其他依赖            |

**Matrix vs Array**：Eigen提供两种核心类型
- `Matrix`：线性代数运算（矩阵乘法、分解等）
- `Array`：逐元素运算（sin, exp, 逐元素乘法等）

## 1.2 快速入门示例

```cpp
#include <iostream>
#include <Eigen/Dense>

int main() {
    // 使用固定大小的矩阵（栈分配，性能最优）
    Eigen::Matrix3d A;
    A << 1, 2, 3,
         4, 5, 6,
         7, 8, 9;
    
    // 使用动态大小矩阵（堆分配）
    Eigen::MatrixXd B(3, 3);
    B.setRandom();  // 随机初始化
    
    // 向量操作
    Eigen::Vector3d v(1, 2, 3);
    
    // 矩阵运算
    Eigen::Matrix3d C = A * B;        // 矩阵乘法
    Eigen::Vector3d y = A * v;        // 矩阵-向量乘法
    double dot = v.dot(v);            // 点积
    
    std::cout << "A =\n" << A << "\n\n";
    std::cout << "A * v =\n" << y << "\n";
    std::cout << "v · v = " << dot << "\n";
    
    return 0;
}
```

### 编译命令

```bash
# Linux/macOS
g++ -std=c++14 -I/path/to/eigen -O2 -o eigen_intro eigen_intro.cpp

# Windows (MinGW)
g++ -std=c++14 -I"C:\eigen" -O2 -o eigen_intro.exe eigen_intro.cpp
```

## 常见问题 (FAQ)

**Q: Eigen与MATLAB相比有什么优势？**

A: 
- **性能**: C++编译代码执行效率远高于MATLAB解释执行
- **部署**: 可生成独立可执行文件，无需运行时环境
- **集成**: 易于集成到现有C++项目中
- **成本**: 完全开源免费（MPL2许可证）

**Q: Eigen支持GPU计算吗？**

A: Eigen 5.0 支持CUDA，可以在GPU内核中使用Eigen的固定大小矩阵类型。对于动态大小矩阵，推荐使用cuBLAS或结合Eigen进行CPU/GPU混合计算。

## 练习题

1. **基础练习**: 创建一个3×3单位矩阵，计算其行列式和逆矩阵。
2. **思考练习**: 解释为什么固定大小矩阵(`Matrix3d`)比动态大小矩阵(`MatrixXd`)性能更好？

## 1.3 命名空间使用指南

```cpp
// 方式1：完整命名空间（推荐，避免命名冲突）
Eigen::Matrix3d A;
Eigen::VectorXd v;

// 方式2：函数内局部using（推荐）
void my_function() {
    using namespace Eigen;
    Matrix3d A;  // 仅在此函数内有效
}

// 方式3：使用using声明
using Eigen::Matrix3d;
Matrix3d A;
```

**最佳实践**：头文件中必须使用完整命名空间，源文件中可使用局部`using namespace`。

---
