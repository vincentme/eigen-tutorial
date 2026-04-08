# 三、矩阵基础篇：声明与基本运算

本篇介绍Eigen矩阵和向量的基本操作，是使用Eigen的基础。

## 3.1 矩阵和向量的声明

Eigen提供了丰富的类型别名，简化矩阵和向量的声明。

### 类型命名规则

```
Matrix[尺寸][数据类型]

尺寸：X = 动态大小, N = 固定大小 N (如 2, 3, 4)
数据类型：d = double, f = float, i = int, cd = complex<double>
```

### 常用类型速查表

| 类型       | 完整定义                           | 说明             |
| ---------- | ---------------------------------- | ---------------- |
| `Matrix3d` | `Matrix<double, 3, 3>`             | 3×3双精度矩阵    |
| `MatrixXd` | `Matrix<double, Dynamic, Dynamic>` | 动态双精度矩阵   |
| `Vector3f` | `Matrix<float, 3, 1>`              | 3维单精度向量    |
| `VectorXd` | `Matrix<double, Dynamic, 1>`       | 动态双精度列向量 |

### 声明示例

```cpp
// 固定大小矩阵（栈分配，性能最优）
Eigen::Matrix3d A;           // 3×3双精度矩阵
Eigen::Vector3d v1;          // 3维列向量

// 动态大小矩阵（堆分配）
Eigen::MatrixXd D(10, 10);   // 10×10双精度矩阵
Eigen::VectorXd v2(100);     // 100维动态列向量

// 特殊矩阵
Eigen::Matrix3d I = Eigen::Matrix3d::Identity();  // 单位矩阵
Eigen::MatrixXd Z = Eigen::MatrixXd::Zero(5, 5);  // 零矩阵
Eigen::MatrixXd R = Eigen::MatrixXd::Random(3, 3);// 随机矩阵
```

## 3.2 矩阵初始化方法

```cpp
// 方法1：逗号初始化
Eigen::Matrix3d A;
A << 1, 2, 3,
     4, 5, 6,
     7, 8, 9;

// 方法2：逐个元素赋值
Eigen::MatrixXd B(3, 3);
for (int i = 0; i < 3; ++i)
    for (int j = 0; j < 3; ++j)
        B(i, j) = i * 3 + j + 1;

// 方法3：预定义函数
Eigen::Matrix3d C = Eigen::Matrix3d::Zero();      // 全零
Eigen::Matrix3d D = Eigen::Matrix3d::Identity();  // 单位矩阵
Eigen::Matrix3d E = Eigen::Matrix3d::Constant(5); // 全为5
Eigen::Matrix3d F = Eigen::Matrix3d::Random();    // 随机值
```

## 3.3 基础矩阵运算

```cpp
Eigen::Matrix3d A, B;
A << 1, 2, 3, 4, 5, 6, 7, 8, 9;
B << 9, 8, 7, 6, 5, 4, 3, 2, 1;

// 算术运算
Eigen::Matrix3d C = A + B;        // 矩阵加法
Eigen::Matrix3d E = A * B;        // 矩阵乘法
Eigen::Matrix3d F = A * 2.0;      // 数乘

// 转置
Eigen::Matrix3d At = A.transpose();

// 矩阵属性
double det = A.determinant();     // 行列式
double tr = A.trace();            // 迹
double norm = A.norm();           // Frobenius范数
Eigen::Matrix3d Ainv = A.inverse(); // 逆矩阵

// 逐元素运算（需转换为Array）
Eigen::Matrix3d D = A.array() * B.array();  // 逐元素乘法
Eigen::Matrix3d E = A.array().sqrt();       // 逐元素平方根
```

## 3.4 向量运算

```cpp
Eigen::Vector3d v1(1, 2, 3);
Eigen::Vector3d v2(4, 5, 6);

// 基本运算
double dot = v1.dot(v2);          // 点积
Eigen::Vector3d cross = v1.cross(v2);  // 叉积（仅3D向量）
double norm = v1.norm();          // 欧几里得范数
Eigen::Vector3d normalized = v1.normalized();  // 归一化向量

// 原地归一化
v1.normalize();

// 向量运算
v1.sum();                         // 所有元素之和
v1.prod();                        // 所有元素之积
v1.mean();                        // 平均值
v1.minCoeff();                    // 最小值
v1.maxCoeff();                    // 最大值
```

## 3.5 数据类型与存储

### 固定大小 vs 动态大小

| 特性     | 固定大小 (`Matrix3d`) | 动态大小 (`MatrixXd`) |
| -------- | --------------------- | --------------------- |
| 内存分配 | 栈（编译时确定）      | 堆（运行时分配）      |
| 性能     | 更快（无动态分配）    | 稍慢                  |
| 适用场景 | 小矩阵（≤4×4）        | 大矩阵或尺寸未知      |

### 存储顺序

```cpp
// Eigen默认列优先存储（与MATLAB、Fortran一致）
Eigen::Matrix<double, 3, 4, Eigen::ColMajor> A;  // 列优先（默认）
Eigen::Matrix<double, 3, 4, Eigen::RowMajor> B;  // 行优先（与C一致）

// 访问元素
A(i, j);  // 第i行第j列（从0开始）
```

---
