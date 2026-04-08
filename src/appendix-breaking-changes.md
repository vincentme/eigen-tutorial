## A. Eigen 5.0破坏性变更说明 

### A.1 版本号变更

Eigen 5.0 采用了**语义化版本控制（Semantic Versioning）**。

**重要说明**：Eigen的版本号格式比较特殊，WORLD版本号保持为3，不会改变。

| 版本      | 版本号格式 | 说明                      |
| --------- | ---------- | ------------------------- |
| Eigen 3.4 | 3.4.0      | 旧格式：WORLD.MAJOR.MINOR |
| Eigen 5.0 | 3.5.0      | 新格式：WORLD.MAJOR.MINOR |
|           |            | 但对外称为 Eigen 5.0      |

**版本号含义**：
- **WORLD（世界版本号）**：保持为3，表示Eigen 3.x系列
- **MAJOR（主版本号）**：包含破坏性API变更（从4跳到5）
- **MINOR（次版本号）**：新增功能，向后兼容
- **PATCH（补丁号）**：Bug修复，向后兼容

**代码中的版本号**：

```cpp
#include <Eigen/Core>

// Eigen 5.0.1
std::cout << EIGEN_WORLD_VERSION << "."    // 输出: 3
          << EIGEN_MAJOR_VERSION << "."     // 输出: 5
          << EIGEN_MINOR_VERSION << "\n";   // 输出: 1

// 注意：EIGEN_WORLD_VERSION是3，不是5
// 对外宣传的"Eigen 5.0"对应的是MAJOR版本号
```

### A.2 C++标准要求

| 版本            | 最低C++标准 | 推荐C++标准 |
| --------------- | ----------- | ----------- |
| Eigen 3.4.x     | C++11       | C++11/14    |
| **Eigen 5.0.x** | **C++14**   | **C++17**   |

### A.3 placeholders命名空间变更

`Eigen::last` 和 `Eigen::all` 被移动到 `Eigen::placeholders` 命名空间。

```cpp
// 旧代码 (Eigen 3.4) - 错误
Eigen::VectorXd v2 = v(Eigen::seq(0, Eigen::last, 2));

// 新代码 (Eigen 5.0) - 推荐
using namespace Eigen::placeholders;
Eigen::VectorXd v2 = v(seq(0, last, 2));
```

### A.4 SVD运行时选项改为编译时模板参数

`BDCSVD` 和 `JacobiSVD` 的计算选项从运行时构造函数参数改为编译时模板参数。

```cpp
// 旧代码 (Eigen 3.4) - 已弃用
Eigen::BDCSVD<Eigen::MatrixXd> svd(A, Eigen::ComputeThinU | Eigen::ComputeThinV);

// 新代码 (Eigen 5.0) - 推荐
Eigen::BDCSVD<Eigen::MatrixXd, Eigen::ComputeThinU | Eigen::ComputeThinV> svd(A);
```

### A.5 其他重要变更

- **aligned_allocator**：不再继承自 `std::allocator`
- **欧拉角**：返回更规范的形式，数值可能与旧版本不同
- **随机数生成器**：行为可能与旧版本不同
- **LGPL许可代码**：已被移除
- **BLAS返回类型**：从 `int` 改为 `void`

### A.6 已移除的功能

**1. `usingnamespace` 已被移除**

```cpp
// 旧代码 (Eigen 3.4) - 已移除
usingnamespace Eigen;

// 新代码 (Eigen 5.0) - 使用标准using
using namespace Eigen;
```

**2. `AlignedBox::sizes()` 已被移除**

```cpp
// 旧代码 (Eigen 3.4) - 已移除
Eigen::AlignedBox2d box;
Eigen::Vector2d sizes = box.sizes();

// 新代码 (Eigen 5.0) - 使用sizes()替代
Eigen::Vector2d sizes = box.sizes();
```

**3. `DenseBase::start()` 和 `DenseBase::end()` 已被移除**

```cpp
// 旧代码 (Eigen 3.4) - 已移除
Eigen::VectorXd v(10);
Eigen::VectorXd v2 = v.start(5);

// 新代码 (Eigen 5.0) - 使用head()
Eigen::VectorXd v2 = v.head(5);
```

### A.7 API行为变更

**1. 欧拉角规范化**

Eigen 5.0 对欧拉角进行了规范化处理，返回值可能与旧版本不同：

```cpp
Eigen::Vector3d ea = q.eulerAngles(0, 1, 2);
// Eigen 5.0 保证返回的欧拉角在规范范围内
// 例如：角度可能在 [-pi, pi] 而不是 [0, 2*pi]
```

**2. 随机数生成**

Eigen 5.0 的随机数生成器行为可能与旧版本不同：

```cpp
Eigen::MatrixXd A = Eigen::MatrixXd::Random(3, 3);
// Eigen 5.0 可能生成不同的随机数序列
// 如果需要确定性结果，建议使用固定种子
```

**3. 矩阵乘法优化**

Eigen 5.0 对矩阵乘法进行了更激进的优化，可能导致浮点数结果的微小差异：

```cpp
Eigen::MatrixXd C = A * B;
// Eigen 5.0 可能使用不同的算法
// 结果可能在数值精度上有微小差异
```
