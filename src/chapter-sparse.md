# 七、稀疏矩阵篇：大规模计算

## 7.1 稀疏矩阵基础

**什么是稀疏矩阵？**

稀疏矩阵是指大部分元素为零的矩阵。在实际应用中，许多大规模矩阵（如有限元刚度矩阵、社交网络邻接矩阵）的非零元素占比通常小于1%。

**存储方式对比**：

| 特性         | 稠密矩阵 (Dense)   | 稀疏矩阵 (Sparse)      |
| ------------ | ------------------ | ---------------------- |
| **存储方式** | 存储所有元素       | 仅存储非零元素及其位置 |
| **内存占用** | O(rows × cols)     | O(nnz) 非零元素数      |
| **访问效率** | O(1) 直接访问      | O(log nnz) 需要查找    |
| **运算效率** | BLAS优化，缓存友好 | 特殊算法，减少零运算   |
| **适用场景** | 大部分元素非零     | 非零元素 < 30%         |

**选择建议**：

```
非零元素占比 > 30%  → 使用稠密矩阵
非零元素占比 < 10%  → 使用稀疏矩阵
中间情况            → 根据具体运算测试选择
```

## 7.2 稀疏矩阵存储格式

Eigen支持两种主要的稀疏矩阵存储格式：

**1. 列压缩存储（CSC，默认）**

```
原始矩阵:          CSC存储:
| 1 0 2 |         values:  [1, 4, 2, 5, 3]
| 0 0 3 |   →     indices: [0, 1, 0, 1, 1]  (行索引)
| 4 5 0 |         starts:  [0, 2, 4, 5]     (每列起始位置)
```

**2. 行压缩存储（CSR）**

```
原始矩阵:          CSR存储:
| 1 0 2 |         values:  [1, 2, 3, 4, 5]
| 0 0 3 |   →     indices: [0, 2, 2, 0, 1]  (列索引)
| 4 5 0 |         starts:  [0, 2, 3, 5]     (每行起始位置)
```

```cpp
#include <Eigen/Sparse>
#include <iostream>

int main() {
    // ========== 创建稀疏矩阵 ==========
    
    // 默认使用列压缩存储（CSC）
    Eigen::SparseMatrix<double> sp_col_major(1000, 1000);
    
    // 行压缩存储（CSR）
    Eigen::SparseMatrix<double, Eigen::RowMajor> sp_row_major(1000, 1000);
    
    // ========== 高效填充方式：三元组列表 ==========
    
    // 预估非零元素数量（优化内存分配）
    sp_col_major.reserve(Eigen::VectorXi::Constant(1000, 3));  // 每列约3个非零元素
    
    std::vector<Eigen::Triplet<double>> triplets;
    triplets.reserve(3000);  // 预分配内存
    
    // 填充三对角矩阵
    for (int i = 0; i < 1000; ++i) {
        triplets.emplace_back(i, i, 2.0);       // 主对角线
        if (i > 0)
            triplets.emplace_back(i, i-1, -1.0); // 下对角线
        if (i < 999)
            triplets.emplace_back(i, i+1, -1.0); // 上对角线
    }
    
    // 批量设置（高效）
    sp_col_major.setFromTriplets(triplets.begin(), triplets.end());
    
    // 压缩存储（移除多余空间）
    sp_col_major.makeCompressed();
    
    std::cout << "矩阵维度: " << sp_col_major.rows() << " x " << sp_col_major.cols() << "\n";
    std::cout << "非零元素数: " << sp_col_major.nonZeros() << "\n";
    std::cout << "稀疏密度: " 
              << 100.0 * sp_col_major.nonZeros() / (sp_col_major.rows() * sp_col_major.cols()) 
              << "%\n";
    
    // 输出:
    // 矩阵维度: 1000 x 1000
    // 非零元素数: 2998
    // 稀疏密度: 0.2998%
    
    // ========== 内存占用对比 ==========
    
    // 稠密矩阵内存: 1000 * 1000 * 8 bytes = 8 MB
    // 稀疏矩阵内存: 2998 * (8 + 4 + 间接开销) ≈ 48 KB
    std::cout << "\n内存节省: " 
              << (1.0 - (double)sp_col_major.nonZeros() / (1000 * 1000)) * 100 
              << "%\n";
    // 输出: 内存节省: 99.7002%
    
    return 0;
}
```

### 5.6.3 稀疏矩阵运算

```cpp
#include <Eigen/Sparse>
#include <iostream>

int main() {
    // 创建两个稀疏矩阵
    Eigen::SparseMatrix<double> A(4, 4);
    std::vector<Eigen::Triplet<double>> triplets;
    
    triplets.emplace_back(0, 0, 1);
    triplets.emplace_back(1, 1, 2);
    triplets.emplace_back(2, 2, 3);
    triplets.emplace_back(3, 3, 4);
    triplets.emplace_back(0, 1, 0.5);
    triplets.emplace_back(1, 0, 0.5);
    
    A.setFromTriplets(triplets.begin(), triplets.end());
    
    // ========== 基本运算 ==========
    
    // 稀疏矩阵 + 稀疏矩阵
    Eigen::SparseMatrix<double> B = A + A;
    
    // 稀疏矩阵 * 标量
    Eigen::SparseMatrix<double> C = 2.0 * A;
    
    // 稀疏矩阵 * 稠密向量
    Eigen::VectorXd v(4);
    v << 1, 2, 3, 4;
    Eigen::VectorXd result = A * v;
    
    std::cout << "A * v = " << result.transpose() << "\n";
    // 输出: A * v = 2 4.5 9 16
    
    // 稀疏矩阵 * 稀疏矩阵
    Eigen::SparseMatrix<double> D = A * A;
    
    // ========== 访问元素 ==========
    
    // 方式1：迭代非零元素
    std::cout << "\n非零元素:\n";
    for (int k = 0; k < A.outerSize(); ++k) {
        for (Eigen::SparseMatrix<double>::InnerIterator it(A, k); it; ++it) {
            std::cout << "  A(" << it.row() << "," << it.col() << ") = " << it.value() << "\n";
        }
    }
    
    // 方式2：检查特定元素（较慢）
    double val = A.coeff(0, 0);  // 返回A(0,0)，不存在则返回0
    
    // 方式3：修改元素（需要非压缩状态）
    A.coeffRef(2, 3) = 1.5;  // 插入新元素
    
    return 0;
}
```

### 5.6.4 稀疏线性求解器

```cpp
#include <Eigen/Sparse>
#include <Eigen/IterativeLinearSolvers>
#include <iostream>

int main() {
    // 创建大规模稀疏矩阵（二维拉普拉斯算子）
    int n = 50;  // 50x50网格
    Eigen::SparseMatrix<double> A(n * n, n * n);
    std::vector<Eigen::Triplet<double>> triplets;
    triplets.reserve(5 * n * n);
    
    for (int i = 0; i < n * n; ++i) {
        triplets.emplace_back(i, i, 4.0);  // 主对角线
        if (i % n > 0) triplets.emplace_back(i, i - 1, -1.0);     // 左邻居
        if (i % n < n - 1) triplets.emplace_back(i, i + 1, -1.0); // 右邻居
        if (i >= n) triplets.emplace_back(i, i - n, -1.0);        // 上邻居
        if (i < n * (n - 1)) triplets.emplace_back(i, i + n, -1.0); // 下邻居
    }
    A.setFromTriplets(triplets.begin(), triplets.end());
    A.makeCompressed();
    
    Eigen::VectorXd b = Eigen::VectorXd::Ones(n * n);
    Eigen::VectorXd x;
    
    std::cout << "矩阵大小: " << n * n << " x " << n * n << "\n";
    std::cout << "非零元素: " << A.nonZeros() << "\n\n";
    
    // ========== 直接求解器 ==========
    
    // SimplicialLLT：对称正定矩阵，Cholesky分解
    Eigen::SimplicialLLT<Eigen::SparseMatrix<double>> llt;
    llt.compute(A);
    if (llt.info() == Eigen::Success) {
        x = llt.solve(b);
        std::cout << "SimplicialLLT 残差: " << (A * x - b).norm() / b.norm() << "\n";
    }
    
    // SimplicialLDLT：更稳定，可处理半正定
    Eigen::SimplicialLDLT<Eigen::SparseMatrix<double>> ldlt;
    ldlt.compute(A);
    if (ldlt.info() == Eigen::Success) {
        x = ldlt.solve(b);
        std::cout << "SimplicialLDLT 残差: " << (A * x - b).norm() / b.norm() << "\n";
    }
    
    // SparseLU：通用求解器，非对称矩阵
    Eigen::SparseLU<Eigen::SparseMatrix<double>> sparse_lu;
    sparse_lu.compute(A);
    if (sparse_lu.info() == Eigen::Success) {
        x = sparse_lu.solve(b);
        std::cout << "SparseLU 残差: " << (A * x - b).norm() / b.norm() << "\n";
    }
    
    // ========== 迭代求解器 ==========
    
    // 共轭梯度法（CG）：对称正定矩阵
    Eigen::ConjugateGradient<Eigen::SparseMatrix<double>> cg;
    cg.compute(A);
    cg.setTolerance(1e-10);
    cg.setMaxIterations(1000);
    x = cg.solve(b);
    
    std::cout << "\n共轭梯度法:\n";
    std::cout << "  迭代次数: " << cg.iterations() << "\n";
    std::cout << "  估计误差: " << cg.error() << "\n";
    std::cout << "  残差: " << (A * x - b).norm() / b.norm() << "\n";
    
    // BiCGSTAB：非对称矩阵
    Eigen::BiCGSTAB<Eigen::SparseMatrix<double>> bicg;
    bicg.compute(A);
    bicg.setTolerance(1e-10);
    x = bicg.solve(b);
    
    std::cout << "\nBiCGSTAB:\n";
    std::cout << "  迭代次数: " << bicg.iterations() << "\n";
    std::cout << "  残差: " << (A * x - b).norm() / b.norm() << "\n";
    
    // ========== 带预处理的迭代求解器 ==========
    
    // 不完全Cholesky预处理
    using Preconditioner = Eigen::IncompleteCholesky<double>;
    Eigen::ConjugateGradient<Eigen::SparseMatrix<double>, Eigen::Lower | Eigen::Upper, Preconditioner> cg_precond;
    cg_precond.compute(A);
    x = cg_precond.solve(b);
    
    std::cout << "\n预处理CG:\n";
    std::cout << "  迭代次数: " << cg_precond.iterations() << " (预处理加速)\n";
    
    return 0;
}
```

### 5.6.5 稀疏求解器选择指南

| 求解器              | 矩阵类型   | 内存 | 速度 | 稳定性 | 推荐场景         |
| ------------------- | ---------- | ---- | ---- | ------ | ---------------- |
| `SimplicialLLT`     | 对称正定   | 中   | 快   | 中     | 结构力学、热传导 |
| `SimplicialLDLT`    | 对称半正定 | 中   | 快   | 高     | 更稳定的Cholesky |
| `SparseLU`          | 任意方阵   | 高   | 中   | 高     | 通用求解器       |
| `SparseQR`          | 任意矩阵   | 高   | 慢   | 最高   | 最小二乘、秩亏   |
| `ConjugateGradient` | 对称正定   | 低   | 快   | 中     | 大规模问题       |
| `BiCGSTAB`          | 非对称     | 低   | 中   | 中     | 一般非对称问题   |
| `GMRES`             | 非对称     | 高   | 可控 | 高     | 困难问题         |

**决策流程**：

```
矩阵是否对称正定？
  ├─ 是 → 规模 < 10000？
  │        ├─ 是 → SimplicialLLT
  │        └─ 否 → ConjugateGradient + 预处理
  └─ 否 → 需要最小二乘解？
           ├─ 是 → SparseQR
           └─ 否 → 规模 < 10000？
                    ├─ 是 → SparseLU
                    └─ 否 → BiCGSTAB + 预处理
```

## 7.3 迭代求解器

对于大规模稀疏线性方程组，迭代求解器通常比直接求解器更高效。

```cpp
#include <Eigen/Sparse>
#include <Eigen/IterativeLinearSolvers>
#include <iostream>

int main() {
    // 创建大规模稀疏矩阵（示例：二维拉普拉斯算子）
    int n = 100;
    Eigen::SparseMatrix<double> A(n * n, n * n);
    std::vector<Eigen::Triplet<double>> triplets;
    
    for (int i = 0; i < n * n; ++i) {
        triplets.push_back(Eigen::Triplet<double>(i, i, 4.0));
        if (i % n > 0) triplets.push_back(Eigen::Triplet<double>(i, i - 1, -1.0));
        if (i % n < n - 1) triplets.push_back(Eigen::Triplet<double>(i, i + 1, -1.0));
        if (i >= n) triplets.push_back(Eigen::Triplet<double>(i, i - n, -1.0));
        if (i < n * (n - 1)) triplets.push_back(Eigen::Triplet<double>(i, i + n, -1.0));
    }
    A.setFromTriplets(triplets.begin(), triplets.end());
    
    Eigen::VectorXd b = Eigen::VectorXd::Ones(n * n);
    Eigen::VectorXd x;
    
    // ========== 共轭梯度法（CG）==========
    // 适用于对称正定矩阵，收敛最快
    Eigen::ConjugateGradient<Eigen::SparseMatrix<double>> cg;
    cg.compute(A);
    cg.setTolerance(1e-10);
    cg.setMaxIterations(1000);
    x = cg.solve(b);
    
    std::cout << "共轭梯度法:\n";
    std::cout << "  迭代次数: " << cg.iterations() << "\n";
    std::cout << "  估计误差: " << cg.error() << "\n";
    std::cout << "  状态: " << (cg.info() == Eigen::Success ? "成功" : "失败") << "\n\n";
    
    // ========== 最小二乘QR法（LSQR）==========
    // 适用于最小二乘问题
    Eigen::LeastSquaresConjugateGradient<Eigen::SparseMatrix<double>> lsqr;
    lsqr.compute(A);
    x = lsqr.solve(b);
    
    std::cout << "最小二乘CG:\n";
    std::cout << "  迭代次数: " << lsqr.iterations() << "\n\n";
    
    // ========== 双共轭梯度稳定法（BiCGSTAB）==========
    // 适用于非对称矩阵
    Eigen::BiCGSTAB<Eigen::SparseMatrix<double>> bicg;
    bicg.compute(A);
    bicg.setTolerance(1e-10);
    x = bicg.solve(b);
    
    std::cout << "BiCGSTAB:\n";
    std::cout << "  迭代次数: " << bicg.iterations() << "\n";
    std::cout << "  估计误差: " << bicg.error() << "\n\n";
    
    // ========== 带预处理的迭代求解器 ==========
    // 预处理可以显著加速收敛
    using Preconditioner = Eigen::IncompleteCholesky<double>;
    Eigen::ConjugateGradient<Eigen::SparseMatrix<double>, Eigen::Lower | Eigen::Upper, Preconditioner> cg_precond;
    cg_precond.compute(A);
    x = cg_precond.solve(b);
    
    std::cout << "预处理CG:\n";
    std::cout << "  迭代次数: " << cg_precond.iterations() << " (预处理后)\n";
    
    return 0;
}
```

### 迭代求解器选择指南

| 求解器              | 适用矩阵类型 | 内存占用         | 收敛速度 | 推荐场景       |
| ------------------- | ------------ | ---------------- | -------- | -------------- |
| `ConjugateGradient` | 对称正定     | 低               | 快       | 椭圆型PDE      |
| `BiCGSTAB`          | 非对称       | 低               | 中等     | 一般非对称问题 |
| `GMRES`             | 非对称       | 高（随迭代增长） | 可控     | 困难问题       |
| `LeastSquaresCG`    | 矩形/超定    | 低               | 中等     | 最小二乘问题   |
| `MINRES`            | 对称不定     | 低               | 中等     | 对称不定矩阵   |

### 预处理技术

```cpp
// 常用预处理类型
using DiagonalPrecond = Eigen::DiagonalPreconditioner<double>;           // 对角预处理
using IncompleteLUT = Eigen::IncompleteLUT<double>;                      // 不完全LU分解
using IncompleteCholesky = Eigen::IncompleteCholesky<double>;            // 不完全Cholesky分解

// 使用预处理
Eigen::BiCGSTAB<Eigen::SparseMatrix<double>, IncompleteLUT> solver;
solver.preconditioner().setDroptol(0.001);  // 设置丢弃容差
solver.compute(A);
x = solver.solve(b);
```

## 常见问题 (FAQ)

**Q: 求解器返回`info() != Success`怎么办？**

A: 检查以下几点：
1. 矩阵是否奇异（行列式为零）
2. 对于Cholesky，矩阵是否正定
3. 数值是否溢出/下溢
4. 尝试更稳定的求解器（如SVD）

**Q: 如何选择合适的求解器？**

A: 决策流程：
```
矩阵是否正定？
  ├─ 是 → 使用LLT（最快）或LDLT
  └─ 否 → 需要最小二乘？
           ├─ 是 → 使用QR或SVD
           └─ 否 → 使用LU
```

**Q: 特征值计算结果与MATLAB不同？**

A: 特征向量的符号和顺序可能不同，这是正常的。验证：
```cpp
// 检查 A*v = lambda*v
assert((A * v - lambda * v).norm() < 1e-10);
```

## 练习题

1. **线性系统**: 实现一个函数，使用LU分解求解多个右端项的线性系统。
2. **PCA实现**: 给定数据矩阵X，实现主成分分析（PCA），返回主成分和方差解释率。
3. **矩阵平方根**: 利用特征值分解计算正定矩阵的平方根。

---
