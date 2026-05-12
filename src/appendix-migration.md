# 附录：迁移变更汇总（Eigen 3.4 → 5.0/5.0.1）

本附录汇总了从 Eigen 3.4 迁移到 Eigen 5.0/5.0.1 时需要注意的所有变化，按主题组织，标注每个变更的类型（破坏性 / 行为语义变化 / 推荐用法变化）。

---

## 版本号语义变更

Eigen 5.0 开始采用**语义化版本控制（Semantic Versioning）**。

- 旧习惯：`WORLD.MAJOR.MINOR`
- 新习惯：`MAJOR.MINOR.PATCH`
- 历史兼容宏 `EIGEN_WORLD_VERSION` 仍保持为 `3`

| 发布版本     | `EIGEN_WORLD_VERSION` | `EIGEN_MAJOR_VERSION` | `EIGEN_MINOR_VERSION` | `EIGEN_PATCH_VERSION` |
| ------------ | --------------------- | --------------------- | --------------------- | --------------------- |
| Eigen 3.4.0  | 3                     | 4                     | 0                     | -                     |
| Eigen 5.0.0  | 3                     | 5                     | 0                     | 0                     |
| Eigen 5.0.1  | 3                     | 5                     | 0                     | 1                     |

**原则**：对外写 **Eigen 5.0.1**，不把 5.0.1 理解成 `3.5.1`。

**输出当前版本**：

```cpp
#include <Eigen/Core>
#include <iostream>

int main() {
    std::cout << "Eigen version: "
              << EIGEN_MAJOR_VERSION << "."
              << EIGEN_MINOR_VERSION << "."
              << EIGEN_PATCH_VERSION << "\n";
}
```

> **类型**：信息性变更

---

## C++ 标准要求

| 版本         | 最低 C++ 标准 |
| ------------ | ------------- |
| Eigen 3.4.x  | C++11         |
| Eigen 5.0.x  | C++14         |

**建议**：编译时至少使用 `-std=c++14`，教程与工程实践中更推荐 `-std=c++17`。

```cmake
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
```

> **类型**：破坏性变更

---

## `Eigen::all` / `Eigen::last` 命名空间迁移

出于与其他项目的命名冲突考虑，Eigen 5.0 将 `all` 和 `last` 从 `Eigen::` 移至 `Eigen::placeholders::`。

**旧写法（3.4）**：
```cpp
using namespace Eigen;
Eigen::VectorXd v2 = v(seq(0, last, 2));
```

**Eigen 5.0 推荐写法**：
```cpp
using namespace Eigen;
using namespace Eigen::placeholders;

Eigen::VectorXd v2 = v(seq(0, last, 2));
```

完整的 `placeholders` 命名空间包含：`all`、`last`、`lastp1`、`end`、`lastN`。

此外，Eigen 还提供了 `Eigen::indexing` 子命名空间作为快捷方式：
```cpp
using namespace Eigen::indexing;  // 导入所有 slicing 相关符号
```

> **类型**：破坏性变更

---

## SVD 计算选项：运行时 → 编译时

Eigen 5.0 中，SVD 的 thin/full U/V 运行时选项被**弃用**，推荐改用编译时模板参数。

**旧写法**：
```cpp
Eigen::BDCSVD<Eigen::MatrixXd> svd(A, Eigen::ComputeThinU | Eigen::ComputeThinV);
```

**推荐写法**：
```cpp
Eigen::BDCSVD<Eigen::MatrixXd, Eigen::ComputeThinU | Eigen::ComputeThinV> svd(A);
// JacobiSVD 同理：
Eigen::JacobiSVD<Eigen::MatrixXd, Eigen::ComputeThinU | Eigen::ComputeThinV> svd2(A);
```

**说明**：旧代码短期内可能还能工作，但应尽早迁移；新代码直接使用编译时模板参数。

> **类型**：推荐用法变化（旧写法已弃用）

---

## 欧拉角返回值更 canonical

Eigen 5.0 调整了欧拉角的返回形式，使其更接近**规范表示**。

- 等价旋转的数值表示可能与旧版本不同
- 角度范围可能变化
- `eulerAngles(a0, a1, a2)` 的**轴顺序语义没有改变**

**建议**：
- 不要依赖某个固定角度区间作为协议
- 若用于内部旋转计算，优先使用四元数或旋转矩阵

```cpp
Eigen::Matrix3d R = ...;
Eigen::Vector3d ea = R.eulerAngles(2, 1, 0);  // ZYX 顺序
// 结果应视为"当前版本下的一种等价表示"
```

> **类型**：行为语义变化

---

## 随机数生成行为变化

`MatrixXd::Random()` 在不同 Eigen 版本中可能得到不同序列。

**建议**：
- 不要把 `Random()` 的具体输出当作回归测试基准
- 若需要可复现实验，使用自己的随机数引擎与固定种子，再写入 Eigen 容器

> **类型**：行为语义变化

---

## `Eigen::aligned_allocator` 不再继承 `std::allocator`

原因与标准库分配器模型变化有关。依赖旧继承关系的代码（如自定义容器封装、STL 容器适配）可能需要调整。

对于普通用户，对齐用法本身不变：
```cpp
// 这仍然是正确的写法
std::vector<Eigen::Vector4d, Eigen::aligned_allocator<Eigen::Vector4d>> v;
```

> **类型**：破坏性变更

---

## Eigen BLAS 返回类型变更

Eigen BLAS 的返回类型从 `int` 改为 `void`，以提高与其他 BLAS 实现的兼容性。

影响范围：BLAS/LAPACK 接口兼容层、与外部线性代数库的集成代码。

> **类型**：破坏性变更

---

## LGPL 代码移除

所有 LGPL 许可代码已移除（如 `ConstrainedConjugateGradient`）。这些本属于"unsupported"模块，使用不广泛。

> **类型**：破坏性变更（移除功能）

---

## 直接包含内部头文件将被阻止

例如 `../src/...` 这类内部路径不再允许直接包含。

> **类型**：破坏性变更

---

## 以下内容**不是** Eigen 5.0 新增

为避免误解，下面这些能力在 Eigen 3.4 中已经存在：

- slicing / indexing API
- C++ 模板别名（如 `Vector<float, 3>` 写法）
- `bfloat16`
- 在 CUDA kernel 中使用固定大小 Eigen 类型
- `SVDBase::info()`
- 若干稀疏矩阵与子块/视图相关接口

---

## 迁移检查清单

从 Eigen 3.4 迁移到 5.0/5.0.1 时，逐项检查：

- [ ] 编译选项已升级到 C++14 或更高
- [ ] 不再使用 `Eigen::all` / `Eigen::last`（改用 `Eigen::placeholders::`）
- [ ] 不再使用 SVD 的运行时 thin/full 选项（改用编译时模板参数）
- [ ] 不依赖 `Random()` 的固定输出序列
- [ ] 不依赖某种固定形式的欧拉角返回值
- [ ] 不直接包含 Eigen 内部头文件
- [ ] 不依赖 `aligned_allocator` 的旧继承关系

## 总结

升级到 Eigen 5.0 时，最值得关注的不是"多了哪些新函数"，而是：

1. 代码是否满足 C++14+
2. 是否依赖旧版 `all`/`last` 命名空间
3. 是否仍在使用旧式 SVD 运行时选项
4. 是否依赖旧的欧拉角或随机数行为
5. 是否有 STL / allocator / BLAS 兼容层代码需要适配

---

> **参考**：[Eigen 5.0 CHANGELOG](https://gitlab.com/libeigen/eigen/-/blob/master/CHANGELOG.md) | [开始学习](intro.md)
