# 四、高级操作篇：块操作与内存映射

本篇介绍块操作、Array、Map 和 Ref，关注表达式语义、内存布局、视图与拷贝。

> **重点**：
> 1. 区分取数据（视图）和复制数据（拷贝）
> 2. 行向量与列向量的类型差异
> 3. `Map` 与 `Ref` 的适用场景
> 4. 零拷贝写法与产生临时对象的写法

## 4.1 块操作与子矩阵

### 4.1.1 先理解：视图(view) 与拷贝(copy)

很多“取子矩阵”的操作默认返回的是**表达式视图**，而不是立即复制一份新矩阵。  
这里要把“表达式”和“视图”区分开来看：

- `block()`、`row()`、`col()`、`head()`、`segment()` 这类 API，通常返回的是**引用原对象数据的视图表达式**
- 赋值给 `MatrixXd`、`VectorXd` 等具体类型时发生**拷贝**
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

规则：

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

(`Matrix` 与 `Array` 的核心差别见 [基础篇](chapter-basics.md) 和 [矩阵基础篇](chapter-matrix-basics.md)。)

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

`Map` 将现有内存缓冲区直接映射为 Eigen 对象，**无需数据拷贝**。适合以下场景：

- 已有外部内存（C 数组、`std::vector`、设备/共享内存缓冲区）
- 将其“看成”一个 Eigen 矩阵或向量
- 希望避免额外复制

可以把 `Map` 理解为：  
**“不是自己的矩阵数据，只是临时包装成 Eigen 类型来用。”**

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
- 需要按行解释时显式使用 `RowMajor`
- 如果数据未正确对齐，可能影响 SIMD 优化，某些场景下还可能触发对齐相关问题（详见 `第九章：调试与排错篇` 的 `9.2 错误3：内存对齐错误`）
- 只读映射使用 `Map<const MatrixXd>`
- `Map` 更强调“把外部内存包装成 Eigen 对象”，而不是“通用函数参数适配”

## 4.4 Ref 类：函数参数传递

给函数传递矩阵/向量参数时有四种可选写法。先给出决策框架，再解释 `Ref` 的原理。

### 4.4.1 四种参数写法的对比

| 写法                         | 能接受什么                                       | 不能接受什么                                            | 适用场景                                                   |
| ---------------------------- | ------------------------------------------------ | ------------------------------------------------------- | ---------------------------------------------------------- |
| `MatrixXd& mat`              | 仅 `MatrixXd` 左值                               | block、Map、固定大小矩阵、表达式                        | 接口极窄、确定只会传 `MatrixXd`                            |
| `const MatrixXd& mat`        | 仅 `MatrixXd` 左值                               | 同上                                                    | 同上（只读版）                                             |
| `Ref<MatrixXd> mat`          | `MatrixXd`、block、Map、固定大小矩阵（布局兼容） | 布局不兼容的表达式（如 `RowMajor` 传入 `ColMajor` Ref） | **需要修改传入数据**且希望兼容多种矩阵来源                 |
| `Ref<const MatrixXd> mat`    | 同上，且额外接受更多只读表达式                   | 仍有布局兼容性限制                                      | **只读访问**且希望兼容多种矩阵来源（最推荐的只读参数写法） |
| `template<typename Derived>` | 任意 Eigen 表达式                                | 无（编译期多态）                                        | 需要最大灵活性，或表达式类型不确定                         |

### 4.4.2 `MatrixXd&` 为什么不够用？

直接用 `MatrixXd&` 作为参数有一个容易被忽略的限制：**它只能绑定到类型完全匹配的左值**。以下调用全部编译失败：

```cpp
void foo(const Eigen::MatrixXd& mat);

Eigen::MatrixXd A(4, 4);
Eigen::Matrix4d B;               // 固定大小矩阵——NOT MatrixXd
Eigen::Map<Eigen::MatrixXd> M;   // Map 类型——NOT MatrixXd

foo(A);                          // OK
foo(A.block(1, 1, 2, 2));       // 编译错误：block 返回的是 Block<...> 表达式，不是 MatrixXd
foo(B);                          // 编译错误：Matrix4d ≠ MatrixXd
foo(M);                          // 编译错误：Map<MatrixXd> ≠ MatrixXd
```

这就是为什么 Eigen 提供了 `Ref`——它通过内部的类型擦除机制，可以绑定到任何布局兼容的矩阵表达式上。

### 4.4.3 `Ref` 的工作原理与限制

`Ref<MatrixXd>` 在构造时做两件事：

1. **检查**传入表达式的内存布局是否兼容（步长、连续性、对齐），不兼容则编译报错
2. **包装**——如果兼容，`Ref` 内部持有指向原数据的指针，零拷贝

```cpp
// Ref 可以接受多种来源
void process(Eigen::Ref<Eigen::MatrixXd> mat) {
    mat(0, 0) = 999;  // 直接修改原始数据（零拷贝视图）
}

Eigen::MatrixXd A(4, 4);
process(A);                              // OK：完整矩阵
process(A.block(1, 1, 3, 3));           // OK：块表达式（布局兼容时）
process(A.topLeftCorner(3, 3));         // OK：角块表达式

Eigen::Matrix4d B;
process(B);                              // OK：固定大小矩阵（布局兼容时）
```

**`Ref` 的局限**：它不是万能适配器。以下情况会出问题：

```cpp
// 布局不兼容
Eigen::Matrix<double, Eigen::Dynamic, Eigen::Dynamic, Eigen::RowMajor> A_row(4, 4);
Eigen::Matrix<double, Eigen::Dynamic, Eigen::Dynamic, Eigen::ColMajor> A_col(4, 4);
A_row.setRandom();
A_col.setRandom();

// 以下可能在编译期或运行时报错（RowMajor vs ColMajor 的内步长可能不兼容）
// process(A_row.block(0, 0, 3, 3));
```

### 4.4.4 关键区分：可写 `Ref` vs 只读 `Ref<const>`

一个重要细节：**`Ref<const MatrixXd>` 比 `Ref<MatrixXd>` 更灵活**。

原因是 const 版本不要求可写语义，对传入表达式的内存布局约束更宽松。大多数只需读数据的函数用 `Ref<const MatrixXd>`。

```cpp
// 只读函数：推荐 Ref<const MatrixXd>
double compute_norm(Eigen::Ref<const Eigen::MatrixXd> mat) {
    return mat.norm();  // 不修改 mat，只读取
}

// 可写函数：用 Ref<MatrixXd>
void zero_first_row(Eigen::Ref<Eigen::MatrixXd> mat) {
    mat.row(0).setZero();  // 需要写入
}
```

### 4.4.5 决策树：到底用哪个？

```
需要修改传入矩阵？
├── 是 → 调用方可能传 block/Map/固定大小矩阵？
│   ├── 是 → 用 Ref<MatrixXd>
│   └── 否 → 用 MatrixXd&（最简单，零开销）
└── 否（只读）→ 调用方可能传 block/Map/固定大小矩阵？
    ├── 是 → 用 Ref<const MatrixXd>（最推荐）
    ├── 否 → 用 const MatrixXd&（最简单，零开销）
    └── 表达式类型极度不确定 → 用 template<typename Derived>
```

**实际中的推荐默认值**：

- 只读参数：**优先 `Ref<const MatrixXd>`**。开销极小，兼容性远超 `const MatrixXd&`
- 可写参数：优先 `Ref<MatrixXd>`，除非调用方永远只传 `MatrixXd` 本身
- 泛型代码：用模板参数 `template<typename Derived>` + `const MatrixBase<Derived>&`

### 4.4.6 完整示例

```cpp
#include <Eigen/Dense>
#include <iostream>

// 只读参数：接受 block、Map、固定大小等各种来源
double frobenius_squared(Eigen::Ref<const Eigen::MatrixXd> mat) {
    return mat.squaredNorm();
}

// 可写参数：接受 block、Map、固定大小等各种来源
void add_identity(Eigen::Ref<Eigen::MatrixXd> mat) {
    mat.diagonal().array() += 1.0;
}

// 泛型版本：接受任意表达式（包括 A + B 这类中间表达式）
template<typename Derived>
double trace_abs(const Eigen::MatrixBase<Derived>& mat) {
    return mat.template cast<double>().cwiseAbs().sum();
}

int main() {
    Eigen::MatrixXd A = Eigen::MatrixXd::Random(3, 3);

    // 直接传 MatrixXd —— 所有写法都 OK
    frobenius_squared(A);
    add_identity(A);

    // 传 block —— MatrixXd& 会失败，Ref 能接住
    frobenius_squared(A.block(0, 0, 2, 2));
    add_identity(A.block(0, 0, 2, 2));  // 直接修改 A 的子块

    // 传固定大小矩阵
    Eigen::Matrix3d B = Eigen::Matrix3d::Random();
    frobenius_squared(B);
    add_identity(B);

    // 传表达式 —— 只有模板能接住
    // frobenius_squared(A * 2.0);  // 编译错误：A * 2.0 是临时表达式，无法绑定到 Ref
    double t = trace_abs(A * 2.0);  // 模板 OK

    return 0;
}
```

> **注意**：`A * 2.0` 这类表达式是纯代数表达式，不存储连续内存，因此无法绑定到 `Ref`（即使是 `Ref<const>`）。如果需要接收这种中间表达式，只能用模板参数。

**Map vs Ref 选择指南**：

| 维度                   | 更适合 `Map`                       | 更适合 `Ref`               |
| ---------------------- | ---------------------------------- | -------------------------- |
| 主要目标               | 把一块现有内存映射成 Eigen 对象    | 为函数参数提供受控适配     |
| 是否拥有数据           | 不拥有                             | 不拥有                     |
| 典型使用位置           | 调用点、数据接入层、外部缓冲区包装 | 函数形参                   |
| 对外部内存建模         | 很适合                             | 不是核心用途               |
| 需要广泛接受复杂表达式 | 不适合                             | 有限制；很多时候模板更适合 |

可以记一个简单规则：

- **已有一块内存，要把它"看成矩阵"**：优先想 `Map`
- **在写函数参数，希望兼容常见 Eigen 对象**：优先想 `Ref`（只读用 `Ref<const>`，可写用 `Ref`）
- **想接受非常多种表达式类型（包括中间表达式）**：优先考虑模板

## 4.5 Broadcasting（广播）

Broadcasting 是 Eigen 中一种将向量自动"扩展"以匹配矩阵维度的机制，用于逐元素运算。它类似于 NumPy 的 `broadcasting` 概念，但 Eigen 的机制更显式。

### 基本概念

对矩阵每行加不同值或每列乘不同系数时，broadcasting 提供简洁写法：

```cpp
#include <Eigen/Dense>
#include <iostream>

int main() {
    Eigen::MatrixXd mat(3, 4);
    mat << 1, 2, 3, 4,
           5, 6, 7, 8,
           9, 10, 11, 12;
    
    Eigen::VectorXd v(3);
    v << 1, 2, 3;
    
    // 方式1：使用 colwise/rowwise + 显式广播
    Eigen::MatrixXd result = mat.colwise() + v;  // v 被"广播"到每一列
    // 等价于：result.col(j) = mat.col(j) + v
    std::cout << "mat.colwise() + v:\n" << result << "\n\n";
    
    // 方式2：逐行运算
    Eigen::RowVectorXd r(4);
    r << 10, 20, 30, 40;
    result = mat.rowwise() + r;  // r 被"广播"到每一行
    std::cout << "mat.rowwise() + r:\n" << result << "\n\n";
    
    // 方式3：结合 Array 使用（更多元素级操作）
    result = (mat.array().colwise() * v.array()).matrix();
    std::cout << "mat.colwise() * v:\n" << result << "\n\n";
    
    return 0;
}
```

**输出**：
```
mat.colwise() + v:
 2  3  4  5
 7  8  9 10
12 13 14 15

mat.rowwise() + r:
11 22 33 44
15 26 37 48
19 30 41 52

mat.colwise() * v:
 1  2  3  4
10 12 14 16
27 30 33 36
```

**常用 Broadcasting 操作**：

| 操作                                  | 效果                               |
| ------------------------------------- | ---------------------------------- |
| `mat.colwise() + v`                   | 每列加上向量 v（v 作为列向量）     |
| `mat.rowwise() + r`                   | 每行加上行向量 r（r 作为行向量）   |
| `mat.colwise().maxCoeff()`            | 每列的最大值                       |
| `mat.rowwise().sum()`                 | 每行的和                           |
| `mat.colwise().squaredNorm()`         | 每列的平方范数                     |
| `mat.array().rowwise() *= v.array()`  | 每列乘以不同的系数（原地操作）     |

**注意事项**：
- `colwise()` 搭配**列向量**使用，`rowwise()` 搭配**行向量**使用
- Broadcasting 操作返回的是**表达式**而非副本，可高效链式调用
- 维度必须兼容：行数匹配 colwise，列数匹配 rowwise

## 4.6 Reshape（重塑）

`reshaped()` 以不同的行/列布局重新解释同一个矩阵，不拷贝数据。

```cpp
#include <Eigen/Dense>
#include <iostream>

int main() {
    // 创建一个 2x6 矩阵
    Eigen::MatrixXd A(2, 6);
    A << 1, 2, 3, 4, 5, 6,
         7, 8, 9, 10, 11, 12;
    
    std::cout << "原始矩阵 A (2x6):\n" << A << "\n\n";
    
    // 重塑为 4x3（仍然是同一块内存）
    Eigen::MatrixXd B = A.reshaped(4, 3);
    std::cout << "重塑后 B (4x3):\n" << B << "\n\n";
    
    // 重塑为列向量（固定形状）
    Eigen::VectorXd v = A.reshaped();
    std::cout << "重塑为向量:\n" << v.transpose() << "\n\n";
    
    // 原地修改重塑后的视图会影响原矩阵
    A.reshaped(3, 4).row(0).setConstant(99);
    std::cout << "修改重塑视图后的 A:\n" << A << "\n\n";
    
    // 固定大小重塑
    Eigen::Matrix<double, 3, 4> C = A.reshaped<Eigen::FixedRows>(3, 4);
    std::cout << "固定大小重塑:\n" << C << "\n";
    
    return 0;
}
```

**关键点**：
- `reshaped()` 返回的依然是**视图**，不拷贝数据
- 如要独立副本，将结果赋值给具体类型（如上面的 `Eigen::MatrixXd B`）
- 元素总数必须不变：`rows * cols == original.rows() * original.cols()`
- 数据按列优先顺序重新排列

**reshaped() vs resize()**：

| 特性     | `reshaped(rows, cols)`       | `resize(rows, cols)`            |
| -------- | ---------------------------- | ------------------------------- |
| 拷贝数据 | 否（视图）                   | 可能重新分配内存，原数据可能丢失 |
| 元素总数 | 不变                         | 可改变，数据不保留或部分保留     |
| 用途     | 改变数据的行列解释方式       | 改变矩阵的实际尺寸               |

## 4.7 STL 互操作

Eigen 提供了与 C++ 标准库的兼容接口，便于在 STL 算法和容器中使用 Eigen 对象。

### 迭代器

```cpp
#include <Eigen/Dense>
#include <algorithm>
#include <iostream>

int main() {
    Eigen::VectorXd v(5);
    v << 5, 3, 1, 4, 2;
    
    // 使用 STL 算法操作 Eigen 向量
    std::sort(v.begin(), v.end());
    std::cout << "排序后: " << v.transpose() << "\n";  // 1 2 3 4 5
    
    // 查找元素
    auto it = std::find(v.begin(), v.end(), 3);
    if (it != v.end())
        std::cout << "找到 3，索引: " << (it - v.begin()) << "\n";
    
    // 逐行排序矩阵
    Eigen::MatrixXd M(3, 4);
    M.setRandom();
    for (int i = 0; i < M.rows(); ++i)
        std::sort(M.row(i).begin(), M.row(i).end());
    std::cout << "逐行排序后的 M:\n" << M << "\n";
    
    return 0;
}
```

### Range-for 遍历

```cpp
#include <Eigen/Dense>
#include <iostream>

int main() {
    Eigen::MatrixXd M(3, 4);
    M.setRandom();
    
    // 遍历所有元素（列优先顺序）
    std::cout << "所有元素（列优先）: ";
    for (auto val : M.reshaped()) {
        std::cout << val << " ";
    }
    std::cout << "\n";
    
    return 0;
}
```

### STL 容器中的 Eigen 类型

```cpp
#include <Eigen/Dense>
#include <vector>
#include <map>

// 动态大小类型可直接放入 STL 容器（无对齐问题）
std::vector<Eigen::MatrixXd> matrices;
std::map<int, Eigen::VectorXd> vectors;

// 固定大小可向量化类型需要 aligned_allocator
// 详见第 9 章：调试与排错篇
std::vector<Eigen::Vector4d, Eigen::aligned_allocator<Eigen::Vector4d>> fixed_vectors;
```

> **对应官方文档**：[Broadcasting](https://eigen.tuxfamily.org/dox/group__TutorialReductionsVisitorsBroadcasting.html) | [Reshape and Slicing](https://eigen.tuxfamily.org/dox/group__TutorialReshape.html) | [STL iterators and algorithms](https://eigen.tuxfamily.org/dox/group__TutorialSTL.html) | [Block operations](https://eigen.tuxfamily.org/dox/group__TutorialBlockOperations.html) | [The Array class and coefficient-wise operations](https://eigen.tuxfamily.org/dox/group__TutorialArrayClass.html) | [Map class](https://eigen.tuxfamily.org/dox/group__TutorialMapClass.html)

---
