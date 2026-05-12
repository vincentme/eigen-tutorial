# 二、基础篇：Eigen简介与快速入门

覆盖 Eigen 的整体概览、最小示例、命名空间使用规范。

> 完成安装验证后，继续阅读：
> - [矩阵基础篇](chapter-matrix-basics.md)：矩阵/向量声明、初始化、运算与存储
> - [高级操作篇](chapter-advanced-operations.md)：块操作、`Array`、`Map`、`Ref` 等进阶主题

## 2.1 什么是Eigen？

**Eigen** 是一个高性能的C++模板库，专注于线性代数、矩阵和向量运算。核心特点：

| 特性           | 说明                               |
| -------------- | ---------------------------------- |
| **纯头文件库** | 无需编译，只需包含头文件即可使用   |
| **表达式模板** | 智能优化运算，避免临时对象开销     |
| **向量化支持** | 自动使用SSE、AVX、NEON等SIMD指令集 |
| **零依赖**     | 除C++标准库外无其他依赖            |

**Matrix vs Array**：Eigen提供两种核心类型
- `Matrix`：用于线性代数运算（矩阵乘法、分解、求解线性方程等）
- `Array`：用于逐元素运算（`sin`、`exp`、逐元素乘法等）

> `A * B` 在 `Matrix` 语义下表示**矩阵乘法**，在 `Array` 语义下表示**逐元素乘法**。

## 2.2 快速入门示例

```cpp
#include <iostream>
#include <Eigen/Dense>

int main() {
    // 使用固定大小的矩阵（尺寸在编译期确定，通常更容易被优化）
    Eigen::Matrix3d A;
    A << 1, 2, 3,
         4, 5, 6,
         7, 8, 9;
    
    // 使用动态大小矩阵（尺寸在运行时确定）
    Eigen::MatrixXd B(3, 3);
    B.setRandom();  // 随机初始化

    // 已知尺寸时，也可继续使用固定大小矩阵
    Eigen::Matrix3d B_fixed = Eigen::Matrix3d::Random();
    
    // 向量操作
    Eigen::Vector3d v(1, 2, 3);
    
    // 矩阵运算
    Eigen::MatrixXd C = A * B;        // 固定大小 × 动态大小 -> 结果也更自然地写成动态矩阵
    Eigen::Matrix3d C_fixed = A * B_fixed; // 固定大小 × 固定大小
    Eigen::Vector3d y = A * v;        // 矩阵-向量乘法
    double dot = v.dot(v);            // 点积
    
    std::cout << "A =\n" << A << "\n\n";
    std::cout << "A * B 的尺寸: " << C.rows() << " x " << C.cols() << "\n";
    std::cout << "A * v =\n" << y << "\n";
    std::cout << "v · v = " << dot << "\n";
    
    return 0;
}
```

### 编译命令

> 未完成环境配置的，先完成 [安装篇](chapter-installation.md) 的最小验证。

```bash
# Linux/macOS
g++ -std=c++17 -I/path/to/eigen -O2 -o eigen_intro eigen_intro.cpp

# Windows (MinGW)
g++ -std=c++17 -I"C:\eigen" -O2 -o eigen_intro.exe eigen_intro.cpp
```

## 2.3 命名空间使用指南

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

**最佳实践**：头文件中必须使用完整命名空间，源文件中可使用局部 `using namespace`。示例代码中用 `using namespace Eigen;` 的情况均为**示例场景**，不照搬到公共头文件中。

> 后续章节会逐步引入更多类型、表达式和分解器，保持类型和命名空间书写清晰有助于减少混淆。

## 常见问题 (FAQ)

**Q: Eigen与MATLAB相比有什么优势？**

A: 
- **性能**: C++编译代码执行效率远高于MATLAB解释执行
- **部署**: 可生成独立可执行文件，无需运行时环境
- **集成**: 易于集成到现有C++项目中
- **成本**: 完全开源免费（MPL2许可证）

**Q: 固定大小矩阵为什么比动态大小矩阵更容易被优化？**

A: 主要有两个原因：
1. **栈分配 vs 堆分配**：固定大小矩阵（如 `Matrix4d`）的数据是对象内部的静态数组，直接在栈上分配，没有堆内存分配开销；动态大小矩阵（如 `MatrixXd`）在运行时分配堆内存。
2. **向量化与对齐**：当固定大小矩阵的尺寸是 16 字节的整数倍时（如 `Vector2d`、`Matrix4d`），Eigen 申请 SIMD 要求的对齐（16/32/64 字节），利用对齐的 SSE/AVX 指令高效读写。这类为"固定大小可向量化"（fixed-size vectorizable）类型。

> **固定大小可向量化类型在 `new` 分配、STL 容器、按值传递等场景下需要注意内存对齐问题**。详见 [调试与排错篇](chapter-debugging.md) 9.2 节。

**Q: Eigen支持GPU计算吗？**

A: Eigen 很早就支持在 CUDA 内核中使用固定大小的矩阵、向量和数组类型；这并不是 5.0 才新增的能力。对于动态大小矩阵，通常更适合结合 cuBLAS 等库，或采用 CPU/GPU 混合计算方案。

## 练习题

1. **基础练习**: 创建一个3×3单位矩阵，计算其行列式和逆矩阵。
2. **思考练习**: 解释为什么固定大小矩阵(`Matrix3d`)通常比动态大小矩阵(`MatrixXd`)更容易被优化？（提示：从内存分配和向量化两个角度思考）
3. **衔接练习**: 思考下面三个概念分别会在后续哪一章深入展开：  
   - `Matrix` 与 `Array` 的区别  
   - 可逆矩阵与线性方程组求解  
   - `Map` 与 `Ref` 的用途  

## 小结

- Eigen 是一个面向线性代数的高性能 C++ 模板库
- 支持固定大小和动态大小对象
- `Matrix` 与 `Array` 的语义不同，不能混为一谈
- 教程示例默认使用 C++17
- 头文件中应避免 `using namespace Eigen`

下一步：[矩阵基础篇](chapter-matrix-basics.md)

> **对应官方文档**：[Getting Started](https://eigen.tuxfamily.org/dox/GettingStarted.html)

---
