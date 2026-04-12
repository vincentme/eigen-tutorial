## A. Eigen 5.0 破坏性变更说明

本附录只收录**有明确版本依据**、且对 Eigen 3.4 用户迁移到 Eigen 5.0/5.0.1 有实际影响的变更。这里重点关注：
- 版本号语义变化
- 最低 C++ 标准变化
- API 行为变化
- 明确的迁移注意事项

### A.1 版本号语义变更：从 `WORLD.MAJOR.MINOR` 到 `MAJOR.MINOR.PATCH`

Eigen 5.0 开始采用**语义化版本控制（Semantic Versioning）**。  
这意味着：

- **对外发布版本号**使用 `MAJOR.MINOR.PATCH`
- **历史兼容宏**中的 `EIGEN_WORLD_VERSION` 仍然保留为 `3`

因此：

| 发布版本 | `EIGEN_WORLD_VERSION` | `EIGEN_MAJOR_VERSION` | `EIGEN_MINOR_VERSION` | `EIGEN_PATCH_VERSION` |
| -------- | --------------------- | --------------------- | --------------------- | --------------------- |
| Eigen 3.4.0 | 3 | 4 | 0 | - |
| Eigen 5.0.0 | 3 | 5 | 0 | 0 |
| Eigen 5.0.1 | 3 | 5 | 0 | 1 |

**关键点**：
- 对外应写 **Eigen 5.0.1**
- 不应把 Eigen 5.0.1 理解成 `3.5.1`
- `WORLD=3` 只是历史兼容信息，不再表示对外主版本号

**推荐的版本输出方式**：

```cpp
#include <Eigen/Core>
#include <iostream>

int main() {
    std::cout << "Eigen version: "
              << EIGEN_MAJOR_VERSION << "."
              << EIGEN_MINOR_VERSION << "."
              << EIGEN_PATCH_VERSION << "\n";

    std::cout << "Legacy world version: "
              << EIGEN_WORLD_VERSION << "\n";
}
```

### A.2 C++ 标准要求提升

Eigen 5.x 要求至少使用 **C++14**。

| 版本 | 最低 C++ 标准 |
| --- | --- |
| Eigen 3.4.x | C++11 |
| Eigen 5.0.x | C++14 |

**迁移建议**：
- 使用 GNU/Clang 时至少加上 `-std=c++14`
- 如果教程或项目中使用了结构化绑定等特性，则应使用 `-std=c++17`

### A.3 `Eigen::all` / `Eigen::last` 命名空间变更

由于与其他项目发生命名冲突，Eigen 5.0 将：

- `Eigen::all`
- `Eigen::last`

移动到了：

- `Eigen::placeholders::all`
- `Eigen::placeholders::last`

**旧写法（3.4 常见写法）**：

```cpp
using namespace Eigen;
Eigen::VectorXd v2 = v(seq(0, last, 2));
```

**Eigen 5.0 中更稳妥的写法**：

```cpp
using namespace Eigen;
using namespace Eigen::placeholders;

Eigen::VectorXd v2 = v(seq(0, last, 2));
```

如果你不想导入整个 `Eigen` 命名空间，也可以显式写完整限定名。

### A.4 SVD 运行时选项已弃用，推荐改用编译时选项

Eigen 5.0 的一个重要变化是：

- **运行时 SVD 选项（thin/full U/V）已弃用**
- **推荐改用编译时模板参数**

这意味着旧写法并不是“突然不存在”，而是**不再推荐**。

**旧写法**：

```cpp
Eigen::BDCSVD<Eigen::MatrixXd> svd(A, Eigen::ComputeThinU | Eigen::ComputeThinV);
```

**推荐写法**：

```cpp
Eigen::BDCSVD<Eigen::MatrixXd, Eigen::ComputeThinU | Eigen::ComputeThinV> svd(A);
```

`JacobiSVD` 同理：

```cpp
Eigen::JacobiSVD<Eigen::MatrixXd, Eigen::ComputeThinU | Eigen::ComputeThinV> svd(A);
```

**迁移建议**：
- 旧代码短期内可能还能工作，但应尽早迁移
- 新代码直接使用编译时模板参数

### A.5 其他明确的破坏性变更

以下变更在 Eigen 5.0 的发布说明中有明确记录：

1. **`Eigen::aligned_allocator` 不再继承 `std::allocator`**
   - 原因与标准库分配器模型变化有关
   - 依赖旧继承关系的代码可能需要调整

2. **欧拉角返回值更 canonical**
   - 与旧版本相比，返回的欧拉角数值表示可能变化
   - 这属于行为变化，不代表轴顺序 API 本身变化

3. **随机数生成行为发生变化**
   - `Random()` 的具体数值序列不应被视为跨版本稳定
   - 不应依赖 Eigen 随机数的固定输出序列做版本无关测试

4. **所有 LGPL 许可代码已移除**
   - 发布说明中明确提到被移除的典型内容是 `ConstrainedConjugateGradient`

5. **Eigen BLAS 的返回类型从 `int` 改为 `void`**
   - 这是为兼容其他 BLAS 实现而做的调整

6. **直接包含内部头文件将报错**
   - 例如 `../src/...` 这类内部路径不再允许直接包含

### A.6 欧拉角行为变化的正确理解

Eigen 5.0 发布说明指出：欧拉角现在以**更规范（canonical）**的形式返回。  
这意味着：

- 等价旋转的**数值表示可能与旧版本不同**
- 角度范围可能变化
- 但 `eulerAngles(a0, a1, a2)` 请求的**轴顺序语义并没有改变**

更稳妥的写法是基于旋转矩阵来理解结果：

```cpp
Eigen::Matrix3d R = ...;
Eigen::Vector3d ea = R.eulerAngles(2, 1, 0);  // ZYX 顺序
```

**迁移建议**：
- 如果你的程序依赖某个固定欧拉角表示区间，升级后必须重新验证
- 如果你的目标是做旋转计算而不是展示角度，优先在内部使用四元数或旋转矩阵

### A.7 随机数行为变化的正确理解

Eigen 5.0 改变了随机数生成行为，因此以下代码：

```cpp
Eigen::MatrixXd A = Eigen::MatrixXd::Random(3, 3);
```

在不同 Eigen 版本中可能得到不同结果。

**建议**：
- 不要把 `Random()` 的具体输出当作回归测试基准
- 若需要可复现实验，请使用你自己的随机数引擎与固定种子，再把数据写入 Eigen 容器

### A.8 迁移检查清单

从 Eigen 3.4 迁移到 Eigen 5.0/5.0.1 时，建议至少检查以下内容：

- [ ] 编译选项是否已升级到 `C++14` 或更高
- [ ] 是否仍在使用 `Eigen::all` / `Eigen::last` 的旧写法
- [ ] 是否仍在使用 SVD 的运行时 thin/full 选项
- [ ] 是否依赖 `Random()` 的固定输出序列
- [ ] 是否依赖某种固定形式的欧拉角返回值
- [ ] 是否直接包含了 Eigen 内部头文件
- [ ] 是否有依赖 `aligned_allocator` 旧继承关系的代码

### A.9 本附录刻意不收录的内容

以下内容**不应**作为 Eigen 5.0 的正式破坏性变更条目写入迁移文档：

- 非法 C++ 语法（例如 `usingnamespace Eigen;`）
- 无法验证的伪迁移项
- “旧代码”和“新代码”完全相同的条目
- 没有明确 release note 或文档依据的 API 移除声明

这样可以避免把读者带入错误迁移路径。
