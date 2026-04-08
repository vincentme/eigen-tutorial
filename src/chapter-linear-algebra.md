# 五、线性代数篇：矩阵分解与求解

## 5.1 线性方程组求解

Eigen提供多种求解器应对不同类型的线性系统。

### 密集矩阵求解器选择指南

| 求解器                 | 矩阵类型   | 算法           | 适用场景               |
| ---------------------- | ---------- | -------------- | ---------------------- |
| `PartialPivLU`         | 可逆方阵   | 部分主元LU     | 通用，较快             |
| `FullPivLU`            | 任意矩阵   | 全主元LU       | 秩亏矩阵，较慢但更稳定 |
| `HouseholderQR`        | 任意矩阵   | Householder QR | 最小二乘问题           |
| `ColPivHouseholderQR`  | 任意矩阵   | 列主元QR       | 秩亏最小二乘           |
| `FullPivHouseholderQR` | 任意矩阵   | 全主元QR       | 最稳定但最慢           |
| `LLT`                  | 正定矩阵   | Cholesky分解   | 最快，需矩阵正定       |
| `LDLT`                 | 半正定矩阵 | 改进Cholesky   | 可处理半正定           |
| `BDCSVD` / `JacobiSVD` | 任意矩阵   | SVD            | 最稳定，适合病态问题   |

### 线性方程组求解示例

```cpp
#include <Eigen/Dense>
#include <iostream>

int main() {
    // 系统 Ax = b
    Eigen::Matrix3d A;
    A << 2, -1, 0,
         -1, 2, -1,
         0, -1, 2;
    
    Eigen::Vector3d b(1, 2, 3);
    
    // ========== 方法1：直接求逆（不推荐用于大型系统）==========
    Eigen::Vector3d x1 = A.inverse() * b;
    
    // ========== 方法2：LU分解（推荐通用方法）==========
    Eigen::PartialPivLU<Eigen::Matrix3d> lu(A);
    Eigen::Vector3d x2 = lu.solve(b);
    
    // ========== 方法3：Cholesky分解（矩阵正定时最快）==========
    Eigen::LLT<Eigen::Matrix3d> llt(A);
    Eigen::Vector3d x3 = llt.solve(b);
    
    // ========== 方法4：QR分解（处理超定/欠定系统）==========
    Eigen::HouseholderQR<Eigen::Matrix3d> qr(A);
    Eigen::Vector3d x4 = qr.solve(b);
    
    // ========== 验证解的正确性 ==========
    double residual = (A * x2 - b).norm();
    std::cout << "残差范数: " << residual << "\n";
    
    // ========== 多次求解（分解复用）==========
    // 当需要求解多个右端项时，先分解再求解更高效
    Eigen::MatrixXd B(3, 5);
    B.setRandom();
    Eigen::MatrixXd X = lu.solve(B);  // 同时求解5个右端项
    
    return 0;
}
```

### 最小二乘问题

```cpp
#include <Eigen/Dense>
#include <iostream>

int main() {
    // 超定系统：寻找x使得 ||Ax - b||最小
    Eigen::MatrixXd A(10, 3);
    A.setRandom();
    
    Eigen::VectorXd b(10);
    b.setRandom();
    
    // ========== 方法1：正规方程（A^T A x = A^T b）==========
    // 最快但数值稳定性较差
    Eigen::VectorXd x1 = (A.transpose() * A).ldlt().solve(A.transpose() * b);
    
    // ========== 方法2：QR分解（推荐）==========
    // 数值稳定性好，速度适中
    Eigen::VectorXd x2 = A.colPivHouseholderQr().solve(b);
    
    // ========== 方法3：SVD（最稳定）==========
    // 适合病态问题，可处理秩亏情况
    // Eigen 5.0+: bdcSvd() 返回 BDCSVD 对象，使用编译时模板参数
    Eigen::VectorXd x3 = A.bdcSvd<Eigen::ComputeThinU | Eigen::ComputeThinV>().solve(b);
    
    // 计算残差
    double residual = (A * x2 - b).norm();
    std::cout << "最小二乘残差: " << residual << "\n";
    
    return 0;
}
```

## 5.2 特征值分解

```cpp
#include <Eigen/Dense>
#include <iostream>
#include <complex>

int main() {
    Eigen::Matrix3d A;
    A << 4, 2, 1,
         2, 5, 3,
         1, 3, 6;
    
    // ========== 自伴矩阵（对称/厄米特）的特征值分解 ==========
    // 使用SelfAdjointEigenSolver（更快更稳定）
    Eigen::SelfAdjointEigenSolver<Eigen::Matrix3d> eigensolver(A);
    
    if (eigensolver.info() != Eigen::Success) {
        std::cerr << "特征值分解失败!\n";
        return 1;
    }
    
    std::cout << "特征值:\n" << eigensolver.eigenvalues() << "\n\n";
    std::cout << "特征向量（列向量）:\n" << eigensolver.eigenvectors() << "\n";
    
    // 验证：A * v = lambda * v
    Eigen::Vector3d v = eigensolver.eigenvectors().col(0);
    double lambda = eigensolver.eigenvalues()(0);
    std::cout << "\n验证: ||Av - λv|| = " << (A * v - lambda * v).norm() << "\n";
    
    // ========== 一般矩阵的特征值分解 ==========
    Eigen::Matrix3d B;
    B << 1, 2, 3,
         4, 5, 6,
         7, 8, 10;
    
    Eigen::EigenSolver<Eigen::Matrix3d> es(B);
    if (es.info() == Eigen::Success) {
        // 特征值可能是复数
        std::cout << "\n特征值（可能复数）:\n" << es.eigenvalues() << "\n";
        
        // 提取实部（如果矩阵有复数特征值）
        Eigen::VectorXd real_eigenvalues = es.eigenvalues().real();
    }
    
    // ========== 广义特征值问题：Ax = λBx ==========
    Eigen::Matrix3d M;
    M.setRandom();
    M = M * M.transpose();  // 确保正定
    
    Eigen::GeneralizedSelfAdjointEigenSolver<Eigen::Matrix3d> ges(A, M);
    std::cout << "\n广义特征值:\n" << ges.eigenvalues() << "\n";
    
    return 0;
}
```

## 5.3 奇异值分解(SVD)

```cpp
#include <Eigen/Dense>
#include <iostream>

int main() {
    Eigen::MatrixXd A(5, 4);
    A.setRandom();
    
    // ========== BDCSVD（分治SVD，大型矩阵更快）==========
    // Eigen 5.0+: 使用编译时模板参数指定计算选项，而非运行时构造函数参数
    Eigen::BDCSVD<Eigen::MatrixXd, Eigen::ComputeThinU | Eigen::ComputeThinV> svd1(A);
    
    // ========== 检查分解是否成功（Eigen 5.0新增info()方法）==========
    if (svd1.info() != Eigen::Success) {
        std::cerr << "SVD分解失败！可能是数值问题。\n";
        return -1;
    }
    
    // ========== JacobiSVD（Jacobi方法，更精确）==========
    // Eigen 5.0+: 使用编译时模板参数指定计算选项
    Eigen::JacobiSVD<Eigen::MatrixXd, Eigen::ComputeThinU | Eigen::ComputeThinV> svd2(A);
    
    // 同样可以检查分解状态
    if (svd2.info() == Eigen::Success) {
        std::cout << "JacobiSVD分解成功\n";
    }
    
    // 获取结果
    Eigen::VectorXd singular_values = svd1.singularValues();
    Eigen::MatrixXd U = svd1.matrixU();
    Eigen::MatrixXd V = svd1.matrixV();
    
    std::cout << "奇异值:\n" << singular_values << "\n\n";
    
    // ========== 矩阵伪逆（Moore-Penrose逆）==========
    // 对于病态或秩亏矩阵
    // 使用相对容差（相对于最大奇异值），而非绝对容差
    double tolerance = 1e-10 * singular_values(0);  // 相对于最大奇异值
    Eigen::VectorXd singular_values_inv = singular_values;
    for (int i = 0; i < singular_values.size(); ++i) {
        if (singular_values(i) > tolerance)
            singular_values_inv(i) = 1.0 / singular_values(i);
        else
            singular_values_inv(i) = 0;
    }
    
    Eigen::MatrixXd A_pinv = V * singular_values_inv.asDiagonal() * U.transpose();
    
    // ========== 矩阵秩 ==========
    int rank = svd1.rank();
    std::cout << "矩阵秩: " << rank << "\n";
    
    // ========== 矩阵条件数 ==========
    double cond = singular_values(0) / singular_values(singular_values.size() - 1);
    std::cout << "条件数: " << cond << "\n";
    
    // ========== 低秩近似 ==========
    // 保留前k个奇异值
    int k = 2;
    Eigen::MatrixXd A_approx = U.leftCols(k) * 
                               singular_values.head(k).asDiagonal() * 
                               V.leftCols(k).transpose();
    
    double approx_error = (A - A_approx).norm();
    std::cout << "低秩近似误差: " << approx_error << "\n";
    
    return 0;
}
```

**SVD info() 方法说明（Eigen 5.0新增）**：

| 返回值                  | 说明                     |
| ----------------------- | ------------------------ |
| `Eigen::Success`        | 分解成功完成             |
| `Eigen::NumericalIssue` | 数值问题，结果可能不准确 |
| `Eigen::NoConvergence`  | 迭代未收敛               |
| `Eigen::InvalidInput`   | 输入矩阵无效             |

> **最佳实践**：在生产代码中始终检查 `info()` 返回值，特别是在处理用户输入或不确定的矩阵时。

## 5.4 矩阵分解详解与选择

```cpp
#include <Eigen/Dense>
#include <iostream>

int main() {
    Eigen::Matrix3d A;
    A << 4, 2, 1,
         2, 5, 3,
         1, 3, 6;
    
    // ========== Cholesky分解：A = LL^T（A必须正定）==========
    Eigen::LLT<Eigen::Matrix3d> llt(A);
    Eigen::Matrix3d L = llt.matrixL();
    std::cout << "Cholesky L:\n" << L << "\n";
    std::cout << "验证 L*L^T:\n" << L * L.transpose() << "\n\n";
    
    // ========== LDL^T分解（可处理半正定）==========
    Eigen::LDLT<Eigen::Matrix3d> ldlt(A);
    
    // ========== LU分解：PA = LU ==========
    Eigen::PartialPivLU<Eigen::Matrix3d> plu(A);
    Eigen::Matrix3d L_lu = Eigen::Matrix3d::Identity();
    L_lu.triangularView<Eigen::StrictlyLower>() = plu.matrixLU();
    Eigen::Matrix3d U_lu = plu.matrixLU().triangularView<Eigen::Upper>();
    std::cout << "LU分解:\nL =\n" << L_lu << "\nU =\n" << U_lu << "\n\n";
    
    // ========== QR分解：A = QR ==========
    Eigen::HouseholderQR<Eigen::Matrix3d> qr(A);
    Eigen::Matrix3d Q = qr.householderQ();
    Eigen::Matrix3d R = qr.matrixQR().triangularView<Eigen::Upper>();
    std::cout << "QR分解:\nQ =\n" << Q << "\nR =\n" << R << "\n";
    
    return 0;
}
```

---
