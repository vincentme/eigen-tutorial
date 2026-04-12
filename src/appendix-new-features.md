## B. Eigen 5.0 新特性

本附录仅列出**可以明确归属于 Eigen 5.0 的变化**，避免把 Eigen 3.4 已有能力误写成 5.0 新增特性。

> **说明**：Eigen 5.0 是一次较大的版本更新，但官方发布说明的重点更多集中在**版本号语义调整、破坏性变更、兼容性调整、性能改进与长期演进方向**。因此，本附录更适合作为“5.0 值得关注的变化摘要”，而不是“所有新 API 的完整列表”。

### B.1 版本号切换到语义化版本（SemVer）

Eigen 5.0 开始采用 **Semantic Versioning**：

- 旧习惯：`WORLD.MAJOR.MINOR`
- 新习惯：`MAJOR.MINOR.PATCH`

例如：

- Eigen 3.4 的对外版本是 `3.4.0`
- Eigen 5.0.1 的对外版本是 `5.0.1`

但出于历史兼容考虑，内部宏中的 `EIGEN_WORLD_VERSION` 仍保持为 `3`。

**示例**：

```cpp
#include <Eigen/Core>
#include <iostream>

int main() {
    std::cout << "Eigen version: "
              << EIGEN_MAJOR_VERSION << "."
              << EIGEN_MINOR_VERSION << "."
              << EIGEN_PATCH_VERSION << "\n";
    return 0;
}
```

### B.2 Eigen 5.x 继续要求 C++14

Eigen 5.0.x 要求编译器至少支持 **C++14**。这不是“新增语法糖特性”，但它是 5.x 时代最重要的使用前提之一。

**建议**：

- 最低标准：`C++14`
- 教程与工程实践中更推荐：`C++17`

**示例**：

```cmake
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
```

### B.3 SVD 计算选项推荐改为编译时模板参数

Eigen 5.0 中，SVD 的 thin/full U/V 运行时选项被**弃用**，推荐改用编译时模板参数。

**推荐写法**：

```cpp
Eigen::BDCSVD<Eigen::MatrixXd, Eigen::ComputeThinU | Eigen::ComputeThinV> svd(A);
Eigen::JacobiSVD<Eigen::MatrixXd, Eigen::ComputeThinU | Eigen::ComputeThinV> svd2(A);
```

**说明**：

- 这项变化的重要性不在于“多了一个新类”，而在于**推荐用法发生了变化**
- 编译时选项更清晰，也更符合 Eigen 5.x 的接口方向

### B.4 `Eigen::all` 和 `Eigen::last` 迁移到 `Eigen::placeholders`

出于命名冲突考虑，Eigen 5.0 将：

- `Eigen::all`
- `Eigen::last`

迁移为：

- `Eigen::placeholders::all`
- `Eigen::placeholders::last`

这对使用 slicing/indexing API 的代码影响较大。

**示例**：

```cpp
#include <Eigen/Core>

using namespace Eigen;
using namespace Eigen::placeholders;

int main() {
    Eigen::VectorXd v(10);
    v.setLinSpaced(10, 0, 9);

    Eigen::VectorXd even = v(seq(0, last, 2));
    return 0;
}
```

### B.5 欧拉角返回值更加规范（canonical）

Eigen 5.0 调整了欧拉角的返回形式，使其更接近**规范表示**。这意味着：

- 与旧版本相比，返回的数值可能不同
- 但它们通常表示的是**同一个旋转**
- 变化主要体现在**角度范围和等价表示**上，而不是 API 参数顺序本身发生变化

**建议**：

- 如果项目依赖某一组固定欧拉角输出结果，升级到 5.0 后应重新验证
- 若用于内部旋转计算，优先使用四元数或旋转矩阵

### B.6 随机数生成行为变化

Eigen 5.0 调整了随机数生成行为，因此：

- `MatrixXd::Random()` 在不同版本之间可能得到不同序列
- 旧版本和新版本的结果不应做“逐值一致”的假设

**示例**：

```cpp
Eigen::MatrixXd A = Eigen::MatrixXd::Random(3, 3);
```

**建议**：

- 不要依赖 Eigen 默认随机数序列做回归测试
- 如果需要可复现的随机行为，使用你自己的随机数引擎生成数据，再填充到 Eigen 对象中

### B.7 `aligned_allocator` 的标准库兼容性变化

Eigen 5.0 中：

- `Eigen::aligned_allocator` 不再继承自 `std::allocator`

这是一个偏底层但很重要的兼容性变化，主要影响：

- 自定义容器封装
- STL 容器适配
- 依赖旧式分配器继承关系的代码

对大多数普通教程读者来说，这不是最常见的日常 API 变化，但对于泛型库作者和底层工程代码是需要关注的。

### B.8 BLAS 返回类型兼容性调整

Eigen 5.0 将 Eigen BLAS 的返回类型从 `int` 改为 `void`，以提高与其他 BLAS 实现的兼容性。

这类变更通常不会影响一般矩阵运算代码，但会影响：

- BLAS/LAPACK 接口兼容层
- 与外部线性代数实现的集成代码
- 某些依赖旧接口签名的适配代码

### B.9 需要注意：以下内容**不是** Eigen 5.0 新增

为了避免误解，下面这些能力虽然在 Eigen 5.0 中依然可用，但**不是 5.0 才首次引入**：

- slicing / indexing API
- C++ 模板别名（如 `Vector<float, 3>` 这类写法）
- `bfloat16`
- 在 CUDA kernel 中使用固定大小 Eigen 类型
- `SVDBase::info()`
- 若干稀疏矩阵与子块/视图相关接口

如果需要区分“5.0 新变化”和“旧版本已存在但 5.0 仍支持的能力”，应优先参考官方发布说明与版本文档。

### B.10 总结

相较于把 Eigen 5.0 理解为“增加了很多全新 API”，更准确的理解是：

- **版本号体系变了**
- **兼容性规则更清晰了**
- **一些旧接口被弃用或迁移了**
- **部分行为更规范但也可能导致升级差异**
- **底层实现、性能与生态兼容性继续演进**

因此，升级到 Eigen 5.0 时，最值得关注的不是“多了哪些炫目的新函数”，而是：

1. 你的代码是否满足 C++14+
2. 是否依赖旧版 `all/last` 命名空间
3. 是否仍在使用旧式 SVD 运行时选项
4. 是否依赖旧的欧拉角或随机数行为
5. 是否有 STL / allocator / BLAS 兼容层代码需要适配
