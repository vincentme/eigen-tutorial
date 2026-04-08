# 四、高级操作篇：块操作与内存映射

本篇介绍Eigen的高级操作技巧，包括块操作、Array类、Map类和Ref类。

## 4.1 块操作与子矩阵

```cpp
Eigen::MatrixXd A(6, 6);
A.setRandom();

// 块操作
Eigen::MatrixXd block1 = A.block(1, 1, 3, 3);  // 从(1,1)开始取3×3块
A.block(0, 0, 2, 2) = Eigen::Matrix2d::Identity();  // 修改块

// 行和列操作
Eigen::VectorXd row0 = A.row(0);
Eigen::VectorXd col1 = A.col(1);

// 特定块操作
Eigen::MatrixXd topLeft = A.topLeftCorner(3, 3);
Eigen::MatrixXd bottomRight = A.bottomRightCorner(3, 3);
Eigen::MatrixXd topRows = A.topRows(3);
Eigen::MatrixXd leftCols = A.leftCols(3);

// 向量特定操作
Eigen::VectorXd v(10);
Eigen::VectorXd head = v.head(3);           // 前3个元素
Eigen::VectorXd tail = v.tail(3);           // 后3个元素
Eigen::VectorXd segment = v.segment(2, 4);  // 从索引2开始的4个元素

// 步长访问（Eigen 5.0+）
using namespace Eigen::placeholders;
Eigen::VectorXd v2 = v(seq(0, last, 2));    // 每2个元素取一个
```

## 4.2 Array类：逐元素运算

当需要进行逐元素运算（而非线性代数运算）时，使用`Array`类。

```cpp
Eigen::ArrayXXd A(3, 3);
A << 1, 2, 3, 4, 5, 6, 7, 8, 9;

// 逐元素运算
Eigen::ArrayXXd B = A * 2;      // 逐元素乘法
Eigen::ArrayXXd C = A.sqrt();   // 平方根
Eigen::ArrayXXd D = A.exp();    // 指数
Eigen::ArrayXXd E = A.log();    // 对数
Eigen::ArrayXXd F = A.pow(2);   // 平方
Eigen::ArrayXXd G = A.abs();    // 绝对值

// 比较运算
Eigen::Array<bool, 3, 3> mask = A > 5;

// Matrix与Array互转
Eigen::MatrixXd M = A.matrix();     // Array -> Matrix
Eigen::ArrayXXd A2 = M.array();     // Matrix -> Array

// 链式操作
Eigen::MatrixXd result = (M.array() * 2).exp().matrix();
```

## 4.3 Map类：零拷贝内存映射

`Map`类允许将现有的内存缓冲区直接映射为Eigen矩阵，**无需数据拷贝**。

```cpp
// 从C数组映射
double raw_data[9] = {1, 2, 3, 4, 5, 6, 7, 8, 9};
Eigen::Map<Eigen::Matrix3d> mat(raw_data);  // 列优先映射

// 从std::vector映射
std::vector<double> vec = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12};
Eigen::Map<Eigen::MatrixXd> mat_from_vec(vec.data(), 3, 4);

// 修改原始数据
mat(0, 0) = 100;  // 直接修改raw_data[0]

// 行优先映射
Eigen::Map<Eigen::Matrix<double, 3, 3, Eigen::RowMajor>> mat_row(raw_data);

// 步长映射
double stride_data[10] = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
Eigen::Map<Eigen::VectorXd, 0, Eigen::InnerStride<2>> strided_vec(stride_data, 5);
```

**注意事项**：
- Map对象不拥有数据，确保原始数据在Map使用期间有效
- 如果数据未正确对齐，可能影响SIMD优化
- 只读映射使用`Map<const MatrixXd>`

## 4.4 Ref类：函数参数传递

`Ref`类专门用于函数参数，可以接受**任意兼容的Eigen表达式**，包括Matrix、Map、块表达式等。

```cpp
// 推荐：使用Ref（零拷贝）
void process_matrix(Eigen::Ref<Eigen::MatrixXd> mat) {
    mat(0, 0) = 999;  // 可以修改原始数据
}

// 只读Ref
void process_matrix_readonly(Eigen::Ref<const Eigen::MatrixXd> mat) {
    std::cout << "范数: " << mat.norm() << "\n";
}

// 使用示例
Eigen::MatrixXd A(4, 4);
A.setRandom();

process_matrix(A);                    // 传入完整矩阵
process_matrix(A.block(1, 1, 2, 2));  // 传入块表达式（零拷贝）

double data[4] = {1, 2, 3, 4};
Eigen::Map<Eigen::MatrixXd> mapped(data, 2, 2);
process_matrix(mapped);               // 传入Map
```

**Map vs Ref 选择指南**：

| 场景             | 推荐使用                                       |
| ---------------- | ---------------------------------------------- |
| 函数参数传递     | `Ref<MatrixXd>`                                |
| 映射外部内存     | `Map<MatrixXd>`                                |
| 需要存储映射对象 | `Map<MatrixXd>`                                |
| 只读访问外部数据 | `Map<const MatrixXd>` 或 `Ref<const MatrixXd>` |

---
