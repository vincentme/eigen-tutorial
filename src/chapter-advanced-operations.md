# 四、高级操作篇：块操作与内存映射

本篇介绍 Eigen 的高级操作技巧，包括块操作、Array、Map 和 Ref。和前几章相比，本章更关注**表达式语义、内存布局、视图(view) 与拷贝(copy)** 等问题。

> **建议先阅读**：基础篇、矩阵基础篇  
> **本章重点**：
> 1. 区分“取一块数据”和“复制一块数据”
> 2. 理解行向量与列向量的类型差异
> 3. 理解 `Map` 与 `Ref` 的适用边界
> 4. 知道哪些写法是零拷贝，哪些写法可能产生临时对象

## 4.1 块操作与子矩阵

### 4.1.1 先理解：视图(view) 与拷贝(copy)

在 Eigen 中，很多“取子矩阵”的操作默认返回的是**表达式视图**，而不是立即复制一份新矩阵。  
这里要把“表达式”和“视图”区分开来看：

- `block()`、`row()`、`col()`、`head()`、`segment()` 这类 API，通常返回的是**引用原对象数据的视图表达式**
- 如果你把结果赋值给 `MatrixXd`、`VectorXd` 这类具体对象，通常会发生**拷贝**
- 修改视图会影响原矩阵；修改拷贝不会影响原矩阵
- 但并不是所有 Eigen 表达式都等于“原地引用原数据”；有些表达式只是**延迟求值的组合表达式**

初学时可以先把这类块操作理解为“默认是视图，不是副本”，但不要把所有“表达式对象”都等同理解成“直接别名”。

```cpp
Eigen::MatrixXd A(6, 6);
A.setRandom();

// 方式1：复制出一个新的矩阵
Eigen::MatrixXd block_copy = A.block(1, 1, 3, 3);

// 方式2：直接操作原矩阵中的那一块
A.block(0, 0, 2, 2) = Eigen::Matrix2d::Identity();
```

对于初学者，可以先记住一个简单规则：

- `MatrixXd x = ...;` 往往表示“我要一个独立副本”
- `A.block(...) = ...;` 往往表示“我直接修改原矩阵中的局部区域”

### 4.1.2 常见块操作

```cpp
Eigen::MatrixXd A(6, 6);
A.setRandom();

// 复制出一个3×3子块
Eigen::MatrixXd block_copy = A.block(1, 1, 3, 3);  // 从(1,1)开始取3×3块

// 直接修改原矩阵中的块
A.block(0, 0, 2, 2) = Eigen::Matrix2d::Identity();

// 行和列操作：注意行向量和列向量的类型不同
Eigen::RowVectorXd row0 = A.row(0);
Eigen::VectorXd col1 = A.col(1);

// 特定块操作
Eigen::MatrixXd top_left = A.topLeftCorner(3, 3);
Eigen::MatrixXd bottom_right = A.bottomRightCorner(3, 3);
Eigen::MatrixXd top_rows = A.topRows(3);
Eigen::MatrixXd left_cols = A.leftCols(3);

// 向量特定操作
Eigen::VectorXd v(10);
Eigen::VectorXd head_copy = v.head(3);         // 前3个元素（复制）
Eigen::VectorXd tail_copy = v.tail(3);         // 后3个元素（复制）
Eigen::VectorXd segment_copy = v.segment(2, 4);// 从索引2开始的4个元素（复制）

// slicing / indexing API 在 Eigen 3.4 已引入；Eigen 5.0 中需要注意 last/all 的命名空间变化
Eigen::VectorXd even = v(Eigen::seq(0, Eigen::placeholders::last, 2));  // 每2个元素取一个
```

**注意事项**：
- `A.row(i)` 返回的是**行表达式**，语义上更接近行向量
- `A.col(j)` 返回的是**列表达式**，语义上更接近列向量
- 若只是“读一块数据”，可以直接使用表达式；若要长期保存或脱离原矩阵使用，再显式复制

## 4.2 Array 类：逐元素运算

当需要进行**逐元素运算**（而非线性代数意义上的矩阵运算）时，使用 `Array` 更自然。

一个非常重要的区分是：

- `Matrix` 主要用于：矩阵乘法、分解、线性方程求解
- `Array` 主要用于：逐元素乘法、逐元素除法、逐元素 `exp/log/sqrt`

如果你看到下面这种写法：

- `A * B`：若 `A` 和 `B` 是 `Matrix`，表示**矩阵乘法**
- `A.array() * B.array()`：表示**逐元素乘法**

这也是 Eigen 初学者最常混淆的地方之一。

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

## 4.3 Map 类：零拷贝内存映射

`Map` 类允许将现有的内存缓冲区直接映射为 Eigen 对象，**无需数据拷贝**。  
它最适合以下场景：

- 你已经有一块外部内存（如 C 数组、`std::vector`、设备/共享内存缓冲区）
- 你想把它“看成”一个 Eigen 矩阵或向量
- 你希望避免额外复制

可以把 `Map` 理解为：  
**“这不是我自己的矩阵数据，但我临时把它包装成 Eigen 类型来用。”**

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
- `Map` 对象**不拥有数据**，必须确保原始内存在 `Map` 生命周期内有效
- `Map<Eigen::Matrix3d>` 默认按 **列主序** 解释内存；也就是说，上面的 `raw_data` 会按列填入矩阵，而不是按很多 C/C++ 初学者直觉中的“逐行填入”
- 如果你希望按行解释同一块内存，需要显式使用 `RowMajor`
- 如果数据未正确对齐，可能影响 SIMD 优化，某些场景下还可能触发对齐相关问题
- 只读映射使用 `Map<const MatrixXd>`
- `Map` 更强调“把外部内存包装成 Eigen 对象”，而不是“通用函数参数适配”

## 4.4 Ref 类：函数参数传递

`Ref` 类主要用于**函数参数适配**。  
和 `Map` 相比，它的重点不是“包装一块外部内存”，而是：

- 让函数能接受 Eigen 矩阵对象
- 也尽可能接受某些兼容表达式
- 在满足布局、步长和可写性要求时避免不必要拷贝

但这里要特别注意：  
`Ref` 并不意味着“任何表达式都一定零拷贝传入”，它也不是“万能 Eigen 参数类型”。  
当表达式的布局、步长、连续性或可写性不满足要求时，编译可能失败；对于只读参数，某些表达式也可能需要借助临时对象后才能传入。

因此更稳妥的理解是：

- `Ref` 适合**受控接口**：你大致知道调用方会传什么类型
- 可写 `Ref` 通常比只读 `Ref` 更严格
- 如果你希望函数接收非常广泛的表达式类型，模板参数往往比强行使用 `Ref` 更自然

```cpp
// 可写Ref：更适合接收布局兼容、可修改的矩阵对象
void process_matrix(Eigen::Ref<Eigen::MatrixXd> mat) {
    mat(0, 0) = 999;  // 可以修改原始数据
}

// 只读Ref：通常比可写Ref更灵活，适合只读访问
void process_matrix_readonly(Eigen::Ref<const Eigen::MatrixXd> mat) {
    std::cout << "范数: " << mat.norm() << "\n";
}

// 使用示例
Eigen::MatrixXd A(4, 4);
A.setRandom();

process_matrix(A);                          // 传入完整矩阵
process_matrix_readonly(A.block(1, 1, 2, 2));  // 只读Ref接收块表达式更常见

double data[4] = {1, 2, 3, 4};
Eigen::Map<Eigen::Matrix<double, 2, 2>> mapped(data);
process_matrix_readonly(mapped);            // 传入Map
```

**实用经验**：
- 需要修改传入矩阵时，优先考虑 `Ref<MatrixXd>`，但要意识到它对参数布局和可写性的要求更严格
- 只读函数参数时，`Ref<const MatrixXd>` 往往更灵活，但也不是“任意表达式都无条件零拷贝”
- 如果参数非常泛化、表达式种类很多，可以考虑模板参数而不是强行用 `Ref`
- 如果你本来就有一块外部内存，优先先用 `Map` 包装，再传给函数

**Map vs Ref 选择指南**：

| 维度 | 更适合 `Map` | 更适合 `Ref` |
| --- | --- | --- |
| 主要目标 | 把一块现有内存映射成 Eigen 对象 | 为函数参数提供受控适配 |
| 是否拥有数据 | 不拥有 | 不拥有 |
| 典型使用位置 | 调用点、数据接入层、外部缓冲区包装 | 函数形参 |
| 对外部内存建模 | 很适合 | 不是核心用途 |
| 需要广泛接受复杂表达式 | 不适合 | 有限制；很多时候模板更适合 |

可以记一个简单规则：

- **已有一块内存，要把它“看成矩阵”**：优先想 `Map`
- **在写函数参数，希望兼容常见 Eigen 对象**：优先想 `Ref`
- **想接受非常多种表达式类型**：优先考虑模板

---
