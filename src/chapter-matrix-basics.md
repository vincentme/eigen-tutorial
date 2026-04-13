# 三、矩阵基础篇：声明与基本运算

本篇是对 Eigen 矩阵与向量操作的**正式系统讲解**。如果您刚完成前一章“基础篇”，可以把本章理解为从“快速上手”进入“规范使用”的过渡：前一章帮助您建立直觉，本章则帮助您建立稳定、可迁移的使用习惯。

**建议先修**：
- 已完成安装与最小示例验证
- 了解 `Matrix` 与 `Array` 的基本区别
- 知道 Eigen 是纯头文件库，且示例默认按 C++17 组织

**本章学习目标**：
1. 学会声明固定大小与动态大小的矩阵/向量
2. 掌握常见初始化方法与基本运算
3. 理解矩阵运算与逐元素运算的区别
4. 建立对行/列、尺寸匹配、存储顺序的基本认识

## 3.1 矩阵和向量的声明

在 Eigen 中，最先要建立的概念不是“怎么写代码”，而是“对象代表什么”：
- 矩阵通常表示线性变换、系数表或二维数据
- 列向量通常表示坐标、状态、参数或观测值
- 行向量则更常用于一行数据、转置结果或某些统计场景

后续章节默认以**列向量**作为主要约定；如果没有特别说明，`VectorXd` 一般指动态大小的列向量。

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
// 固定大小矩阵（尺寸在编译期确定，通常更容易被优化）
Eigen::Matrix3d A;           // 3×3双精度矩阵
Eigen::Vector3d v1;          // 3维列向量

// 动态大小矩阵（尺寸在运行时确定）
Eigen::MatrixXd D(10, 10);   // 10×10双精度矩阵
Eigen::VectorXd v2(100);     // 100维动态列向量

// 特殊矩阵
Eigen::Matrix3d I = Eigen::Matrix3d::Identity();  // 单位矩阵
Eigen::MatrixXd Z = Eigen::MatrixXd::Zero(5, 5);  // 零矩阵
Eigen::MatrixXd R = Eigen::MatrixXd::Random(3, 3);// 随机矩阵
```

## 3.2 矩阵初始化方法

初始化方式不只是语法差异，也对应不同的使用场景：
- **逗号初始化**：适合小型固定大小对象，最直观
- **循环赋值**：适合按规则填充
- **预定义函数**：适合零矩阵、单位矩阵、常量矩阵、随机矩阵等常见情况

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

这一节请特别注意两类“乘法”不要混淆：

- `A * B`：**线性代数意义**上的矩阵乘法
- `A.array() * B.array()`：**逐元素**乘法

这是 Eigen 初学者最常见的混淆点之一。  
如果您发现自己在写 `sqrt`、`exp`、逐元素乘除法，一般就应该先想到 `Array` 视图。

```cpp
Eigen::Matrix3d A, B;
A << 2, -1, 0,
     -1, 2, -1,
     0, -1, 2;                    // 可逆矩阵
B << 9, 8, 7,
     6, 5, 4,
     3, 2, 1;

// 算术运算
Eigen::Matrix3d C = A + B;        // 矩阵加法
Eigen::Matrix3d AB = A * B;       // 矩阵乘法
Eigen::Matrix3d scaled = A * 2.0; // 数乘

// 转置
Eigen::Matrix3d At = A.transpose();

// 矩阵属性
double det = A.determinant();       // 行列式
double tr = A.trace();              // 迹
double norm = A.norm();             // Frobenius范数
Eigen::Matrix3d Ainv = A.inverse(); // 逆矩阵（仅对可逆方阵有意义）

// 逐元素运算（需转换为Array）
Eigen::Matrix3d coeff_product = A.array() * B.array(); // 逐元素乘法

// 逐元素平方根要求元素非负；这里单独构造一个满足前提的示例矩阵
Eigen::Matrix3d P;
P << 1, 4, 9,
     16, 25, 36,
     49, 64, 81;
Eigen::Matrix3d sqrtP = P.array().sqrt();
```

## 3.4 向量运算

向量部分建议优先记住三件事：

1. `dot()` 是点积，结果是标量
2. `cross()` 只适用于三维向量
3. `normalized()` 返回新向量，而 `normalize()` 会直接修改原对象

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

这一节是后续“高级操作篇”的前置基础。  
特别是 `block`、`Map`、`Ref`、切片访问等功能，都和“对象尺寸是否已知”“内存是否连续”“默认按列存储还是按行存储”密切相关。

如果您暂时只想先会用，可以先记住：
- 小而固定的矩阵，优先考虑固定大小类型
- 尺寸运行时才能确定时，使用动态大小类型
- Eigen 默认是**列优先（ColMajor）**

### 固定大小 vs 动态大小

| 特性     | 固定大小 (`Matrix3d`) | 动态大小 (`MatrixXd`) |
| -------- | --------------------- | --------------------- |
| 尺寸信息 | 编译期已知            | 运行时确定            |
| 性能     | 对小而固定的矩阵通常更容易优化 | 更灵活，但通常会有动态分配开销 |
| 适用场景 | 小矩阵、尺寸固定场景  | 大矩阵或尺寸未知      |

### 存储顺序

```cpp
// Eigen默认列优先存储（与MATLAB、Fortran一致）
Eigen::Matrix<double, 3, 4, Eigen::ColMajor> A;  // 列优先（默认）
Eigen::Matrix<double, 3, 4, Eigen::RowMajor> B;  // 行优先（与C一致）

// 访问元素
A(i, j);  // 第i行第j列（从0开始）
```

---
