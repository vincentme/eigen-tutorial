# 十、实战案例篇

本章通过多个独立案例展示 Eigen 在统计学习、几何计算、状态估计和图像处理中的实际用法。

> **阅读说明**
> 1. 本章每一节示例都是**独立程序**，都包含各自的 `main()`，请不要直接把整章代码拼接为单个源文件编译。
> 2. 本章示例默认按 **C++17** 编写，因为部分代码使用了结构化绑定等语法；如果你希望统一到 C++14，需要手动改写对应示例。
> 3. 本章更偏向“教学演示”而不是“生产级实现”，重点是展示 Eigen 的用法与建模思路。
> 4. 阅读时建议结合前文相关章节，尤其是矩阵基础、线性代数、几何变换与调试篇。
> 5. 阅读本章时，请特别留意不同案例中的**数据组织方式**：有的示例使用“每行一个样本”，有的使用“每列一个点”，还有的直接把矩阵当作二维网格或图像。

## 10.0 案例路线图

| 案例 | 主题 | 建议先读 | 数据组织方式 | 适合读者 |
| --- | --- | --- | --- | --- |
| 10.1 PCA | 统计分析、降维 | 3.3、5.2 | 每行一个样本 | 想把 Eigen 用于数据分析的读者 |
| 10.2 线性回归与多项式拟合 | 最小二乘建模 | 3.3、5.1 | 每行一个样本 / 一维向量 | 想练习回归建模的读者 |
| 10.3 刚体变换与 ICP | 几何计算、点云配准 | 5.3、6.2、6.4 | 每列一个点（`3 x N`） | 机器人、视觉、图形学方向读者 |
| 10.4 卡尔曼滤波 | 状态估计 | 3.3、5.1 | 状态向量 + 协方差矩阵 | 想理解滤波基本实现的读者 |
| 10.5 图像处理 | 卷积、梯度计算 | 4.1、4.2 | 二维矩阵作为图像 | 想练习矩阵块操作与逐元素计算的读者 |

**阅读建议**：
- 如果你更关注**数据分析**，建议先读 `10.1` 和 `10.2`
- 如果你更关注**几何/机器人**，建议先读 `10.3`
- 如果你更关注**控制与估计**，建议先读 `10.4`
- 如果你更关注**矩阵操作与图像卷积**，建议先读 `10.5`

## 10.1 主成分分析（PCA）

**前置知识**：
- 矩阵与向量基本运算
- 协方差矩阵
- 特征值分解 / `SelfAdjointEigenSolver`

**数据约定**：
- 本节示例采用“**每行一个样本**”的数据组织方式

**编译建议**：
- 推荐使用 **C++17**
- 本节代码为**独立示例程序**

```cpp
#include <Eigen/Dense>
#include <iostream>
#include <vector>

/**
 * PCA实现
 * @param X 数据矩阵（每行一个样本）
 * @param n_components 保留的主成分数量
 * @return 包含投影矩阵和方差解释率的pair
 */
std::pair<Eigen::MatrixXd, Eigen::VectorXd> pca(
    const Eigen::MatrixXd& X, 
    int n_components) {
    
    // 1. 数据中心化
    Eigen::VectorXd mean = X.colwise().mean();
    Eigen::MatrixXd X_centered = X.rowwise() - mean.transpose();
    
    // 2. 计算协方差矩阵
    Eigen::MatrixXd cov = (X_centered.transpose() * X_centered) / (X.rows() - 1);
    
    // 3. 特征值分解
    Eigen::SelfAdjointEigenSolver<Eigen::MatrixXd> solver(cov);
    if (solver.info() != Eigen::Success) {
        throw std::runtime_error("PCA特征值分解失败");
    }
    
    // 4. 按特征值降序排序
    // SelfAdjointEigenSolver返回升序排列的特征值
    // 需要降序时，同时反转特征值和对应的特征向量列
    Eigen::VectorXd eigenvalues = solver.eigenvalues().reverse();
    Eigen::MatrixXd eigenvectors = solver.eigenvectors().rowwise().reverse();
    
    // 5. 选择前n_components个主成分
    Eigen::MatrixXd components = eigenvectors.leftCols(n_components);
    
    // 6. 计算方差解释率
    double total_var = eigenvalues.sum();
    Eigen::VectorXd explained_variance_ratio = eigenvalues.head(n_components) / total_var;
    
    return {components, explained_variance_ratio};
}

// 使用示例
int main() {
    // 生成示例数据
    Eigen::MatrixXd X(100, 3);
    X.setRandom();
    // 添加一些相关性
    X.col(1) = X.col(0) * 0.8 + X.col(1) * 0.2;
    
    auto [components, variance_ratio] = pca(X, 2);
    
    std::cout << "主成分:\n" << components << "\n\n";
    std::cout << "方差解释率: " << variance_ratio.transpose() << "\n";
    std::cout << "累计解释率: " << variance_ratio.sum() * 100 << "%\n";
    
    // 数据投影
    Eigen::VectorXd mean = X.colwise().mean();
    Eigen::MatrixXd X_projected = (X.rowwise() - mean.transpose()) * components;
    std::cout << "\n投影后数据维度: " << X_projected.rows() << " x " << X_projected.cols() << "\n";
    
    return 0;
}
```

## 10.2 线性回归与多项式拟合

**前置知识**：
- 矩阵乘法与转置
- 最小二乘问题
- QR 分解
- `CompleteOrthogonalDecomposition`

**数据约定**：
- 线性回归部分采用“**每行一个样本**”的数据组织方式
- 多项式拟合部分使用一维向量构造范德蒙德矩阵

**编译建议**：
- 推荐使用 **C++17**
- 本节代码为**独立示例程序**

```cpp
#include <Eigen/Dense>
#include <iostream>
#include <vector>

/**
 * 多元线性回归
 * 求解: y = X * beta + epsilon
 * 最小化: ||y - X*beta||^2
 */
class LinearRegression {
public:
    Eigen::VectorXd coefficients;
    double intercept = 0;
    
    void fit(const Eigen::MatrixXd& X, const Eigen::VectorXd& y) {
        // 添加偏置列（全1）
        Eigen::MatrixXd X_with_bias(X.rows(), X.cols() + 1);
        X_with_bias.col(0).setOnes();
        X_with_bias.rightCols(X.cols()) = X;
        
        // 使用列主元QR求解：作为常见、稳妥的默认选择
        // 如果问题可能秩亏，或你需要更稳妥地处理欠定/最小范数解，
        // 可以考虑 completeOrthogonalDecomposition()
        coefficients = X_with_bias.colPivHouseholderQr().solve(y);
        
        intercept = coefficients(0);
        coefficients = coefficients.tail(X.cols());
    }
    
    Eigen::VectorXd predict(const Eigen::MatrixXd& X) const {
        return X * coefficients + Eigen::VectorXd::Constant(X.rows(), intercept);
    }
    
    double score(const Eigen::MatrixXd& X, const Eigen::VectorXd& y) const {
        Eigen::VectorXd y_pred = predict(X);
        double ss_res = (y - y_pred).squaredNorm();
        double ss_tot = (y - y.mean() * Eigen::VectorXd::Ones(y.size())).squaredNorm();
        return 1.0 - ss_res / ss_tot;  // R^2 score
    }
};

/**
 * 多项式拟合
 * 拟合: y = a0 + a1*x + a2*x^2 + ... + an*x^n
 */
Eigen::VectorXd polynomial_fit(const Eigen::VectorXd& x, 
                               const Eigen::VectorXd& y, 
                               int degree) {
    // 构建范德蒙德矩阵
    Eigen::MatrixXd V(x.size(), degree + 1);
    for (int i = 0; i <= degree; ++i) {
        V.col(i) = x.array().pow(i);
    }
    
    // 求解最小二乘问题
    // 这里继续使用列主元QR；若更关注秩亏情形的稳健性，
    // 可改为 V.completeOrthogonalDecomposition().solve(y)
    return V.colPivHouseholderQr().solve(y);
}

// 使用示例
int main() {
    // 线性回归示例
    std::cout << "=== 线性回归示例 ===\n";
    
    // 生成数据: y = 2*x1 + 3*x2 + 1 + noise
    Eigen::MatrixXd X(100, 2);
    X.setRandom();
    Eigen::VectorXd y = 2 * X.col(0) + 3 * X.col(1) + 
                        Eigen::VectorXd::Ones(100) + 
                        0.1 * Eigen::VectorXd::Random(100);
    
    LinearRegression model;
    model.fit(X, y);
    
    std::cout << "系数: " << model.coefficients.transpose() << "\n";
    std::cout << "截距: " << model.intercept << "\n";
    std::cout << "R^2 score: " << model.score(X, y) << "\n";
    
    // 多项式拟合示例
    std::cout << "\n=== 多项式拟合示例 ===\n";
    
    Eigen::VectorXd x_vals(10);
    x_vals << 0, 1, 2, 3, 4, 5, 6, 7, 8, 9;
    Eigen::VectorXd y_vals = x_vals.array().square() + 2 * x_vals.array() + 1;
    
    Eigen::VectorXd coeffs = polynomial_fit(x_vals, y_vals, 2);
    std::cout << "多项式系数 (从低次到高次): " << coeffs.transpose() << "\n";
    
    return 0;
}
```

## 10.3 三维刚体变换与ICP算法

**前置知识**：
- 刚体变换、旋转矩阵与四元数
- 奇异值分解（SVD）
- 点云配准的基本概念

**数据约定**：
- 本节点云采用“**每列一个点**”的组织方式，即矩阵形状为 `3 x N`

**实现说明**：
- 本节中的 ICP 为**教学简化版**
- 最近邻搜索使用暴力法，仅用于展示 Eigen 在线性代数部分的用法
- 生产环境通常需要 KD-tree 等更高效的数据结构
- 本节重点在于说明：如何用 Eigen 表达**刚体变换、去中心化、协方差矩阵与 SVD 求解**

**编译建议**：
- 推荐使用 **C++17**
- 本节代码为**独立示例程序**

```cpp
#include <Eigen/Dense>
#include <Eigen/Geometry>
#include <iostream>
#include <vector>

using namespace Eigen;

/**
 * 计算两组点之间的刚性变换（SVD方法）
 * 求解 R, t 使得: target = R * source + t
 */
std::pair<Matrix3d, Vector3d> rigid_transform_3D(
    const MatrixXd& source, 
    const MatrixXd& target) {
    
    // 计算质心
    Vector3d centroid_src = source.rowwise().mean();
    Vector3d centroid_tgt = target.rowwise().mean();
    
    // 去质心
    MatrixXd src_demean = source.colwise() - centroid_src;
    MatrixXd tgt_demean = target.colwise() - centroid_tgt;
    
    // 计算协方差矩阵
    Matrix3d H = src_demean * tgt_demean.transpose();
    
    // SVD分解（这里直接使用编译时模板参数写法）
    JacobiSVD<Matrix3d, ComputeFullU | ComputeFullV> svd(H);
    Matrix3d U = svd.matrixU();
    Matrix3d V = svd.matrixV();
    
    // 计算旋转矩阵
    Matrix3d R = V * U.transpose();
    
    // 处理反射情况（确保R是旋转矩阵而非反射矩阵）
    if (R.determinant() < 0) {
        V.col(2) *= -1;
        R = V * U.transpose();
    }
    
    // 计算平移
    Vector3d t = centroid_tgt - R * centroid_src;
    
    return {R, t};
}

/**
 * 简单ICP（迭代最近点）算法实现
 */
class SimpleICP {
public:
    struct Result {
        Matrix3d R;
        Vector3d t;
        double rmse;
        int iterations;
    };
    
    Result align(const MatrixXd& source, const MatrixXd& target, 
                 int max_iterations = 50, double tolerance = 1e-6) {
        MatrixXd src = source;
        Matrix3d R_total = Matrix3d::Identity();
        Vector3d t_total = Vector3d::Zero();
        
        double prev_error = std::numeric_limits<double>::max();
        
        for (int iter = 0; iter < max_iterations; ++iter) {
            // 步骤1: 找到最近点对（简化版本：假设已知对应关系）
            // 实际应用中需要KD树等数据结构
            MatrixXd matched_target = findNearestNeighbors(src, target);
            
            // 步骤2: 计算刚性变换
            auto [R, t] = rigid_transform_3D(src, matched_target);
            
            // 步骤3: 应用变换
            src = (R * src).colwise() + t;
            
            // 累计变换
            R_total = R * R_total;
            t_total = R * t_total + t;
            
            // 计算误差
            double error = (src - matched_target).colwise().norm().mean();
            
            if (std::abs(prev_error - error) < tolerance) {
                return {R_total, t_total, error, iter + 1};
            }
            prev_error = error;
        }
        
        return {R_total, t_total, prev_error, max_iterations};
    }
    
private:
    MatrixXd findNearestNeighbors(const MatrixXd& src, const MatrixXd& tgt) {
        // 暴力搜索最近邻点（仅用于演示）
        // 生产环境建议使用nanoflann或FLANN库加速
        MatrixXd result(3, src.cols());
        for (int i = 0; i < src.cols(); ++i) {
            double min_dist = std::numeric_limits<double>::max();
            int min_idx = 0;
            for (int j = 0; j < tgt.cols(); ++j) {
                double dist = (src.col(i) - tgt.col(j)).norm();
                if (dist < min_dist) {
                    min_dist = dist;
                    min_idx = j;
                }
            }
            result.col(i) = tgt.col(min_idx);
        }
        return result;
    }
};

// 使用示例
int main() {
    // 创建源点云
    MatrixXd source(3, 100);
    source.setRandom();
    
    // 应用已知变换生成目标点云
    AngleAxisd rotation(M_PI / 6, Vector3d::UnitZ());
    Vector3d translation(1, 2, 3);
    
    MatrixXd target = (rotation.toRotationMatrix() * source).colwise() + translation;
    
    // 添加噪声
    target += 0.01 * MatrixXd::Random(3, 100);
    
    // 使用ICP恢复变换
    SimpleICP icp;
    auto result = icp.align(source, target);
    
    std::cout << "=== ICP配准结果 ===\n";
    std::cout << "迭代次数: " << result.iterations << "\n";
    std::cout << "最终RMSE: " << result.rmse << "\n\n";
    
    std::cout << "估计的旋转矩阵:\n" << result.R << "\n\n";
    std::cout << "估计的平移向量: " << result.t.transpose() << "\n\n";
    
    // 验证
    std::cout << "真实旋转:\n" << rotation.toRotationMatrix() << "\n";
    std::cout << "真实平移: " << translation.transpose() << "\n";
    
    return 0;
}
```

## 10.4 卡尔曼滤波器

**前置知识**：
- 矩阵乘法、转置与逆
- 线性状态空间模型
- 协方差与高斯噪声的基本概念

**实现说明**：
- 本节使用的是**线性卡尔曼滤波**教学示例
- 更新步骤中直接使用了矩阵求逆，这样写便于理解公式，但在更严肃的数值计算场景中，通常更推荐改写为线性方程求解形式
- 本节重点不是构建工业级滤波器，而是展示如何用 Eigen 组织状态向量、系统矩阵、协方差矩阵与更新公式

**编译建议**：
- 推荐使用 **C++17**
- 本节代码为**独立示例程序**

```cpp
#include <Eigen/Dense>
#include <iostream>

using namespace Eigen;

/**
 * 线性卡尔曼滤波器实现
 * 
 * 状态方程: x_k = F * x_{k-1} + B * u_k + w_k
 * 观测方程: z_k = H * x_k + v_k
 * 
 * w_k ~ N(0, Q)  过程噪声
 * v_k ~ N(0, R)  观测噪声
 */
class KalmanFilter {
public:
    KalmanFilter(int state_dim, int measurement_dim) :
        n(state_dim), m(measurement_dim) {
        
        x = VectorXd::Zero(n);      // 状态估计
        P = MatrixXd::Identity(n, n); // 协方差矩阵
        F = MatrixXd::Identity(n, n); // 状态转移矩阵
        B = MatrixXd::Zero(n, n);   // 控制矩阵
        H = MatrixXd::Zero(m, n);   // 观测矩阵
        Q = MatrixXd::Identity(n, n) * 0.01; // 过程噪声
        R = MatrixXd::Identity(m, m) * 0.1;  // 观测噪声
    }
    
    // 预测步骤
    void predict(const VectorXd& u = VectorXd()) {
        // 状态预测
        x = F * x;
        if (u.size() > 0) {
            x += B * u;
        }
        
        // 协方差预测
        P = F * P * F.transpose() + Q;
    }
    
    // 更新步骤
    void update(const VectorXd& z) {
        // 计算卡尔曼增益
        MatrixXd S = H * P * H.transpose() + R;
        MatrixXd K = P * H.transpose() * S.inverse();
        
        // 状态更新
        VectorXd y = z - H * x;  // 观测残差
        x = x + K * y;
        
        // 协方差更新（Joseph形式，数值更稳定）
        MatrixXd I_KH = MatrixXd::Identity(n, n) - K * H;
        P = I_KH * P * I_KH.transpose() + K * R * K.transpose();
    }
    
    VectorXd getState() const { return x; }
    MatrixXd getCovariance() const { return P; }
    
public:
    MatrixXd F, B, H, Q, R;
    
private:
    int n, m;
    VectorXd x;
    MatrixXd P;
};

// 使用示例：一维运动跟踪
int main() {
    // 状态: [位置, 速度]
    // 观测: [位置]
    KalmanFilter kf(2, 1);
    
    // 设置系统矩阵
    double dt = 0.1;  // 时间步长
    kf.F << 1, dt,
            0, 1;
    kf.H << 1, 0;
    kf.Q << 0.01, 0,
            0, 0.001;
    kf.R << 0.1;
    
    // 模拟真实轨迹和观测
    double true_pos = 0;
    double true_vel = 2;
    
    std::cout << "=== 卡尔曼滤波跟踪 ===\n";
    std::cout << "时间\t观测位置\t估计位置\t估计速度\n";
    
    for (int i = 0; i < 20; ++i) {
        // 真实状态更新
        true_pos += true_vel * dt;
        
        // 生成带噪声的观测
        double measurement = true_pos + 0.1 * ((double)rand() / RAND_MAX - 0.5);
        
        // 卡尔曼滤波
        kf.predict();
        kf.update(VectorXd::Constant(1, measurement));
        
        VectorXd state = kf.getState();
        std::cout << i * dt << "\t" << measurement << "\t\t" 
                  << state(0) << "\t\t" << state(1) << "\n";
    }
    
    return 0;
}
```

## 10.5 图像处理 - 高斯滤波与边缘检测

**前置知识**：
- 矩阵块操作
- Array / Matrix 的逐元素运算
- 二维卷积与 Sobel 算子的基本概念

**数据约定**：
- 本节直接使用 `MatrixXd` 表示灰度图像
- 图像矩阵的每个元素代表一个像素值

**实现说明**：
- 本节实现为**简化教学版本**
- 未包含 padding、边界处理、SIMD 优化和实际图像 I/O
- 重点在于展示如何使用 Eigen 进行卷积和梯度计算
- 本节尤其适合和第四章一起阅读，以体会块操作与逐元素运算在二维数据上的配合方式

**编译建议**：
- 推荐使用 **C++17**
- 本节代码为**独立示例程序**

```cpp
#include <Eigen/Dense>
#include <iostream>

using namespace Eigen;

/**
 * 创建高斯核
 */
MatrixXd gaussian_kernel(int size, double sigma) {
    MatrixXd kernel(size, size);
    int center = size / 2;
    double sum = 0;
    
    for (int i = 0; i < size; ++i) {
        for (int j = 0; j < size; ++j) {
            int x = i - center;
            int y = j - center;
            kernel(i, j) = std::exp(-(x*x + y*y) / (2 * sigma * sigma));
            sum += kernel(i, j);
        }
    }
    
    return kernel / sum;  // 归一化
}

/**
 * 2D卷积（简化实现，无填充）
 */
MatrixXd convolve2d(const MatrixXd& input, const MatrixXd& kernel) {
    int output_rows = input.rows() - kernel.rows() + 1;
    int output_cols = input.cols() - kernel.cols() + 1;
    
    MatrixXd output(output_rows, output_cols);
    
    for (int i = 0; i < output_rows; ++i) {
        for (int j = 0; j < output_cols; ++j) {
            output(i, j) = (input.block(i, j, kernel.rows(), kernel.cols()).array() 
                           * kernel.array()).sum();
        }
    }
    
    return output;
}

/**
 * Sobel边缘检测
 */
std::pair<MatrixXd, MatrixXd> sobel_edge_detection(const MatrixXd& image) {
    // Sobel算子
    MatrixXd sobel_x(3, 3);
    sobel_x << -1, 0, 1,
               -2, 0, 2,
               -1, 0, 1;
    
    MatrixXd sobel_y(3, 3);
    sobel_y << -1, -2, -1,
                0,  0,  0,
                1,  2,  1;
    
    MatrixXd grad_x = convolve2d(image, sobel_x);
    MatrixXd grad_y = convolve2d(image, sobel_y);
    
    return {grad_x, grad_y};
}

// 使用示例
int main() {
    // 创建测试图像（简单的边缘）
    MatrixXd image(10, 10);
    image.setZero();
    image.rightCols(5).setOnes();  // 右半边为1
    
    std::cout << "=== 原始图像 ===\n" << image << "\n\n";
    
    // 高斯滤波
    MatrixXd gaussian = gaussian_kernel(3, 1.0);
    std::cout << "=== 高斯核 ===\n" << gaussian << "\n\n";
    
    MatrixXd blurred = convolve2d(image, gaussian);
    std::cout << "=== 高斯滤波后 ===\n" << blurred << "\n\n";
    
    // Sobel边缘检测
    auto [grad_x, grad_y] = sobel_edge_detection(image);
    MatrixXd magnitude = (grad_x.array().square() + grad_y.array().square()).sqrt();
    
    std::cout << "=== 边缘强度 ===\n" << magnitude << "\n";
    
    return 0;
}
```

---
