# 九、调试与排错篇

## 9.1 常见编译错误

### 错误1：头文件未找到

```
fatal error: Eigen/Core: No such file or directory
```

**解决方案**：
```bash
# 检查Eigen安装路径
g++ -I/usr/include/eigen3 ...  # 系统安装路径
# 或
g++ -I/path/to/eigen ...       # 手动安装路径
```

### 错误2：C++标准不满足

```
error: Eigen requires at least c++14 support
```

**解决方案**：
```bash
g++ -std=c++14 ...  # 或 -std=c++17
```

### 错误3：模块头文件缺失或包含不完整

```text
error: ‘JacobiSVD’ is not a member of ‘Eigen’
```

或：

```text
error: incomplete type ‘Eigen::SelfAdjointEigenSolver<...>’ used in nested name specifier
```

**常见原因**：
- 只包含了 `<Eigen/Core>`，但实际使用了分解、特征值或几何相关功能
- 头文件包含层级不完整

**解决方案**：
```bash
# 若使用大多数稠密矩阵功能，直接包含：
#include <Eigen/Dense>

# 若希望按模块最小化包含，可按需使用：
#include <Eigen/SVD>
#include <Eigen/Eigenvalues>
#include <Eigen/Geometry>
```

**说明**：
这类错误比线程链接问题更常见，也更符合 Eigen 初学者的实际踩坑路径。

## 9.2 运行时错误

### 错误1：矩阵尺寸不匹配

```cpp
Eigen::MatrixXd A(3, 3), B(4, 4);
Eigen::MatrixXd C = A * B;  // 运行时错误（调试模式）
```

**调试方法**：
```cpp
// 使用断言检查
assert(A.cols() == B.rows() && "矩阵维度不匹配");

// 或使用Eigen的调试模式（默认开启）
// 编译时添加 -DEIGEN_NO_DEBUG 禁用（发布模式）
```

### 错误2：奇异矩阵

```cpp
Eigen::Matrix2d A;
A << 1, 2,
     2, 4;  // 奇异矩阵（行列式为0）
Eigen::Matrix2d A_inv = A.inverse();  // 结果可能为Inf/NaN
```

**调试方法**：
```cpp
// 检查行列式
double det = A.determinant();
if (std::abs(det) < 1e-10) {
    std::cerr << "警告：矩阵接近奇异，行列式 = " << det << "\n";
    // 使用伪逆或其他方法
}

// 或使用条件数（这里直接使用编译时模板参数写法）
Eigen::JacobiSVD<Eigen::Matrix2d, Eigen::ComputeFullU | Eigen::ComputeFullV> svd(A);
const auto singular_values = svd.singularValues();
double sigma_max = singular_values(0);
double sigma_min = singular_values(singular_values.size() - 1);

if (sigma_min <= 1e-15) {
    std::cerr << "警告：矩阵奇异或接近奇异，最小奇异值 = " << sigma_min << "\n";
} else {
    double cond = sigma_max / sigma_min;
    if (cond > 1e10) {
        std::cerr << "警告：矩阵病态，条件数 = " << cond << "\n";
    }
}
```

### 错误3：内存对齐错误

**前置概念：什么是"固定大小可向量化"（fixed-size vectorizable）类型？**

Eigen 中 **"固定大小可向量化"（fixed-size vectorizable）** 类型的判断标准：

> 该类型在编译期大小固定，且大小是 16 字节的整数倍。

因为 SIMD 指令（SSE/AVX/AVX512）以 128 bit（16 字节）、256 bit（32 字节）或 512 bit（64 字节）的数据包为单位工作，这些数据包在**对齐到数据包大小**时读写效率最高。Eigen 会为满足条件的类型申请对应的对齐（16/32/64 字节），从而充分利用 SIMD 向量化。

**典型的"固定大小可向量化"类型包括**：

| 类型                 | 大小    | 对齐要求 |
| -------------------- | ------- | -------- |
| `Eigen::Vector2d`    | 16 字节 | 16 字节  |
| `Eigen::Vector4d`    | 32 字节 | 32 字节  |
| `Eigen::Vector4f`    | 16 字节 | 16 字节  |
| `Eigen::Matrix2d`    | 32 字节 | 32 字节  |
| `Eigen::Matrix2f`    | 16 字节 | 16 字节  |
| `Eigen::Matrix4d`    | 64 字节 | 64 字节  |
| `Eigen::Matrix4f`    | 64 字节 | 64 字节  |
| `Eigen::Affine3d`    | 64 字节 | 64 字节  |
| `Eigen::Affine3f`    | 32 字节 | 32 字节  |
| `Eigen::Quaterniond` | 32 字节 | 32 字节  |
| `Eigen::Quaternionf` | 16 字节 | 16 字节  |

> **注意**：动态大小类型（如 `VectorXd`、`MatrixXd`）在堆上分配自己的系数数组，会自行处理绝对对齐，因此不受下文讨论的问题影响。

**问题根源**：

Eigen 使用 SIMD 指令（SSE/AVX）来优化性能。固定大小可向量化类型必须在**正确对齐的内存地址**上创建，否则 SIMD 指令会触发段错误（segfault）。

Eigen 通常会自动处理这些对齐问题——它通过 `alignas` 关键字声明对齐要求，并重载 `operator new` 来返回对齐的指针。但在少数边界情况下，这些对齐设置会被绕过，从而导致崩溃。

**实际错误信息**：

程序因对齐问题崩溃时，通常出现以下断言失败信息：

```
my_program: path/to/eigen/Eigen/src/Core/DenseStorage.h:44:
Eigen::internal::matrix_array<T, Size, MatrixOptions, Align>::internal::matrix_array()
[with T = double, int Size = 2, int MatrixOptions = 2, bool Align = true]:
Assertion (reinterpret_cast<size_t>(array) & (sizemask)) == 0 && "this assertion
is explained here: ... **** READ THIS WEB PAGE !!! ****"' failed.
```

**定位问题根源**：

用调试器获取 backtrace 定位触发断言的代码位置：

```bash
$ gdb ./my_program          # 启动 GDB
> run                       # 运行程序
...                         # 复现崩溃
> bt                        # 获取 backtrace
```

---

**四大根源与对应解决方案**

对齐问题的根源可分为四类，下面逐一说明。

> **C++17 用户注意**：如果你使用 C++17 且编译器足够新（GCC >= 7, Clang >= 5, MSVC >= 19.12），编译器会自动处理大部分对齐问题。在这种情况下，`EIGEN_MAKE_ALIGNED_OPERATOR_NEW` 宏为空操作，STL 容器的对齐也由编译器保证。但如果你需要兼容旧编译器或旧标准，请继续阅读下文。

---

#### 根源 1：结构体/类中包含固定大小可向量化 Eigen 成员

类中包含固定大小可向量化 Eigen 类型成员，通过 `new` 动态分配时，返回的指针可能不满足对齐要求。

```cpp
// 问题代码
class Foo {
    Eigen::Vector4d v;  // 需要 32 字节对齐（AVX 下）
    int x;
};

Foo *foo = new Foo;  // foo 指针可能未对齐，导致 foo->v 也未对齐
```

**原因**：`Eigen::Vector4d` 本身有对齐的 `operator new`，但 `Foo` 没有。当 `new Foo` 时，用的是 `Foo` 的 `operator new`（默认无对齐），返回的 `foo` 指针可能不是 32 字节对齐的。成员 `v` 的对齐是相对于 `Foo` 起始地址的——如果 `foo` 不对齐，`foo->v` 也不会对齐。

**解决方案 A：使用 `EIGEN_MAKE_ALIGNED_OPERATOR_NEW`**（推荐）

```cpp
class Foo {
    Eigen::Vector4d v;
    int x;
public:
    EIGEN_MAKE_ALIGNED_OPERATOR_NEW  // 重载 operator new，保证返回对齐指针
};

Foo *foo = new Foo;  // 安全
```

**解决方案 B：条件性对齐（模板参数决定是否需对齐）**

```cpp
template<int N> class Foo {
    typedef Eigen::Matrix<float, N, 1> Vector;
    enum { NeedsToAlign = (sizeof(Vector) % 16) == 0 };
    Vector v;
public:
    EIGEN_MAKE_ALIGNED_OPERATOR_NEW_IF(NeedsToAlign)
};

Foo<4> *foo4 = new Foo<4>;  // 128bit 对齐
Foo<3> *foo3 = new Foo<3>;  // 默认系统对齐（Vector3f 为 12 字节，不需要特殊对齐）
```

**解决方案 C：禁用对齐（性能有损失，但不影响正确性）**

```cpp
class Foo {
    Eigen::Matrix<double, 4, 1, Eigen::DontAlign> v;
};
```

**关于成员排列顺序**：Eigen 成员不需要放在类的最前面——对齐是自动处理的。但从节省内存的角度，建议将对齐要求高的成员放在前面，避免编译器插入过多的填充字节。例如 AVX 下：

```cpp
class Foo {
    Eigen::Vector4d v;  // 32 字节对齐
    int x;              // 后面可能有填充
};
// 优于
class Foo {
    int x;              // 后面编译器会插入 24 字节填充
    Eigen::Vector4d v;
};
```

#### 根源 2：STL 容器或手动内存分配

将固定大小可向量化 Eigen 类型（或包含它们的类）放入 STL 容器时，默认的 `std::allocator` 不保证对齐。

```cpp
// 问题代码
std::vector<Eigen::Vector2d> my_vector;
std::map<int, Eigen::Matrix4f> my_map;
```

**解决方案 A：使用 `Eigen::aligned_allocator`**

```cpp
// vector
std::vector<Eigen::Vector2d, Eigen::aligned_allocator<Eigen::Vector2d>> my_vector;

// map（注意：第三个参数 std::less 是默认值，但必须显式写出才能指定第四个参数）
std::map<int, Eigen::Vector4d, std::less<int>,
         Eigen::aligned_allocator<std::pair<const int, Eigen::Vector4d>>> my_map;
```

**解决方案 B：特化 `std::vector`（C++98/03 环境）**

```cpp
#include <Eigen/StdVector>

// 针对特定类型特化 vector
EIGEN_DEFINE_STL_VECTOR_SPECIALIZATION(Eigen::Matrix2d)

// 之后可以直接使用
std::vector<Eigen::Matrix2d> vec;  // 自动使用对齐分配
```

> **注意**：特化必须在所有用到 `std::vector<该类型>` 的代码之前声明，否则编译器会用默认分配器编译那些实例。

**手动内存分配**：使用 `std::make_shared` 或 `std::allocate_shared` 时，同样需要传入对齐分配器。

#### 根源 3：按值传递 Eigen 对象

在 C++17 之前，将固定大小可向量化 Eigen 对象按值传递给函数不仅低效，还可能导致崩溃：

```cpp
// 问题代码（pre-C++17）
void my_function(Eigen::Vector2d v);  // 按值传递，可能违反对齐

struct Foo { Eigen::Vector2d v; };
void my_function(Foo v);              // 同样的问题
```

**解决方案**：改用 const 引用传递

```cpp
void my_function(const Eigen::Vector2d& v);
void my_function(const Foo& v);
```

> **注意**：函数**返回** Eigen 对象按值是没问题的，只有参数传递时存在此问题。C++17 及以上标准中，编译器已能正确处理此场景。

#### 根源 4：编译器对栈对齐的错误假设（GCC on Windows）

这是仅见于 Windows 上 GCC（MinGW、TDM-GCC）的历史问题（GCC 4.5 后已修复）。

**现象**：在一个普通函数中声明局部 Eigen 变量，也会触发对齐断言：

```cpp
void foo() {
    Eigen::Quaternionf q;  // 可能崩溃
}
```

**原因**：GCC 默认假设栈已 16 字节对齐，但在 Windows 上栈仅保证 4 字节对齐。当函数从其他线程或被其他编译器编译的二进制调用时，栈对齐可能被破坏。

**解决方案**（任选其一）：

```cpp
// 方案 A：单函数标记
__attribute__((force_align_arg_pointer)) void foo() {
    Eigen::Quaternionf q;
}
```

```bash
# 方案 B：全局编译选项（告知 GCC 栈只保证 4 字节对齐）
g++ -mincoming-stack-boundary=2 ...

# 方案 C：全局编译选项（等同于给所有函数加 force_align_arg_pointer）
g++ -mstackrealign ...
```

---

**如果不需要最优向量化，如何彻底禁用对齐？**

三种方式（按影响范围递增）：

1. **单个类型禁用**：使用 `Eigen::DontAlign` 模板参数
   ```cpp
   Eigen::Matrix<double, 4, 1, Eigen::DontAlign> v;
   ```

2. **降低静态对齐阈值**：定义 `EIGEN_MAX_STATIC_ALIGN_BYTES` 为 0（禁用所有 16 字节及以上静态对齐）或 16（仅禁用 32/64 字节对齐）。注意这会破坏 ABI 兼容性。

3. **完全禁用向量化**：同时定义 `EIGEN_DONT_VECTORIZE` 和 `EIGEN_DISABLE_UNALIGNED_ARRAY_ASSERT`。这保留了对齐机制（维持 ABI 兼容），但关闭了向量化。

**如何验证代码的对齐安全性？**

代码有单元测试时，可通过链接仅返回 8 字节对齐缓冲区的自定义 malloc 库暴露对齐缺陷。编译时定义 `EIGEN_MALLOC_ALREADY_ALIGNED=0`。

**总结：最佳实践**

| 场景                                        | 解决方案                                                               |
| ------------------------------------------- | ---------------------------------------------------------------------- |
| 类中有固定大小可向量化成员，且用 `new` 分配 | 添加 `EIGEN_MAKE_ALIGNED_OPERATOR_NEW`                                 |
| 模板类中条件性地需要对齐                    | 使用 `EIGEN_MAKE_ALIGNED_OPERATOR_NEW_IF(条件)`                        |
| STL 容器中存放固定大小可向量化类型          | 使用 `Eigen::aligned_allocator`                                        |
| 函数参数传递 Eigen 对象（pre-C++17）        | 改用 `const &` 传递                                                    |
| GCC on Windows 栈对齐问题                   | `-mstackrealign` 或 `force_align_arg_pointer` 属性                     |
| 动态大小类型（`VectorXd`, `MatrixXd` 等）   | 无需特殊处理                                                           |
| C++17 + 现代编译器                          | 编译器自动处理，上述宏和分配器大多变为空操作，但仍建议保留以兼容旧环境 |

## 9.3 错误处理机制

Eigen提供多层错误处理机制，从编译时检查到运行时验证。

### 编译时检查（静态断言）

```cpp
// 维度检查在编译时进行
Eigen::Matrix<double, 3, 4> A;
Eigen::Matrix<double, 4, 3> B;
auto C = A * B;  // 编译通过：3x4 * 4x3 = 3x3
// auto D = A + B;  // 编译错误：维度不匹配
```

### 运行时检查（调试模式）

```cpp
// 调试模式下，Eigen会检查运行时错误
Eigen::MatrixXd X(3, 3), Y(4, 4);
// auto Z = X + Y;  // 调试模式触发断言

// 禁用调试检查（发布模式）
// 编译选项：-DEIGEN_NO_DEBUG -DNDEBUG
```

### 分解运算状态检查

```cpp
// 所有分解类都提供info()方法
Eigen::LLT<Eigen::MatrixXd> llt(A);
if (llt.info() == Eigen::NumericalIssue) {
    // 矩阵非正定
    std::cerr << "Cholesky分解失败：矩阵非正定\n";
}

Eigen::BDCSVD<Eigen::MatrixXd, Eigen::ComputeThinU | Eigen::ComputeThinV> svd(A);
if (svd.info() != Eigen::Success) {
    std::cerr << "SVD分解失败\n";
}
```

### 数值检查方法

```cpp
Eigen::MatrixXd A = Eigen::MatrixXd::Random(100, 100);

// 检查NaN
if (A.hasNaN()) {
    std::cerr << "矩阵包含NaN\n";
}

// 检查Inf
if (!A.allFinite()) {
    std::cerr << "矩阵包含Inf\n";
}

// 检查条件数（判断是否病态）
Eigen::JacobiSVD<Eigen::MatrixXd> svd(A);
const auto singular_values = svd.singularValues();
double sigma_max = singular_values(0);
double sigma_min = singular_values(singular_values.size() - 1);

if (sigma_min <= 1e-15) {
    std::cerr << "矩阵奇异或接近奇异，最小奇异值 = " << sigma_min << "\n";
} else {
    double cond = sigma_max / sigma_min;
    if (cond > 1e10) {
        std::cerr << "矩阵病态，条件数 = " << cond << "\n";
    }
}
```

### 错误处理最佳实践

| 场景         | 检查方法                       | 处理方式                         |
| ------------ | ------------------------------ | -------------------------------- |
| 矩阵求逆     | 检查条件数或最小奇异值         | 使用伪逆、正则化或改写为 `solve` |
| 线性求解     | `info() != Success`            | 使用更稳健的分解或最小二乘解     |
| Cholesky分解 | `info() == NumericalIssue`     | 矩阵非正定，改用LU/QR            |
| 数值溢出     | `hasNaN()`, `allFinite()`      | 检查输入数据与数值范围           |
| 秩亏/奇异    | 最小奇异值接近 0，或条件数极大 | 降级为 SVD / 伪逆 / 正则化       |

## 9.4 调试技巧

### 打印矩阵信息

```cpp
#include <Eigen/Dense>
#include <iostream>

template<typename Derived>
void print_matrix_info(const Eigen::MatrixBase<Derived>& mat, const std::string& name) {
    std::cout << "=== " << name << " ===\n";
    std::cout << "尺寸: " << mat.rows() << " x " << mat.cols() << "\n";
    std::cout << "内容:\n" << mat << "\n";
    std::cout << "范数: " << mat.norm() << "\n";
    std::cout << "最小值: " << mat.minCoeff() << "\n";
    std::cout << "最大值: " << mat.maxCoeff() << "\n";
    std::cout << "==================\n\n";
}

// 使用
Eigen::MatrixXd A = Eigen::MatrixXd::Random(5, 5);
print_matrix_info(A, "矩阵A");
```

### 检查数值稳定性

```cpp
bool check_numerical_stability(const Eigen::MatrixXd& A) {
    // 检查NaN
    if (A.hasNaN()) {
        std::cerr << "错误：矩阵包含NaN\n";
        return false;
    }
    
    // 检查Inf
    if (!A.allFinite()) {
        std::cerr << "错误：矩阵包含Inf\n";
        return false;
    }
    
    // 检查数值范围
    double max_abs = A.cwiseAbs().maxCoeff();
    if (max_abs > 1e100) {
        std::cerr << "警告：矩阵元素过大 (" << max_abs << ")\n";
    }
    if (max_abs < 1e-100 && max_abs > 0) {
        std::cerr << "警告：矩阵元素过小 (" << max_abs << ")\n";
    }
    
    return true;
}
```

## 9.5 性能瓶颈定位

```cpp
#include <Eigen/Dense>
#include <chrono>
#include <map>
#include <string>

// 简单的性能分析器
class Profiler {
    static inline std::map<std::string, std::pair<int, double>> stats;
public:
    static void record(const std::string& name, double ms) {
        stats[name].first++;
        stats[name].second += ms;
    }
    
    static void report() {
        std::cout << "\n=== 性能报告 ===\n";
        for (const auto& [name, data] : stats) {
            auto [count, total] = data;
            std::cout << name << ": " << count << " 次, 平均 " 
                      << total / count << " ms, 总计 " << total << " ms\n";
        }
    }
};

#define PROFILE_SCOPE(name) \
    auto _start = std::chrono::high_resolution_clock::now(); \
    struct _profiler_##__LINE__ { \
        const char* _name; \
        ~_profiler_##__LINE__() { \
            auto _end = std::chrono::high_resolution_clock::now(); \
            auto _dur = std::chrono::duration_cast<std::chrono::microseconds>(_end - _start).count() / 1000.0; \
            Profiler::record(_name, _dur); \
        } \
    } _prof_##__LINE__{name};

// 使用示例
void my_function() {
    PROFILE_SCOPE("my_function");
    
    PROFILE_SCOPE("矩阵创建");
    Eigen::MatrixXd A = Eigen::MatrixXd::Random(1000, 1000);
    
    PROFILE_SCOPE("矩阵运算");
    Eigen::MatrixXd B = A * A.transpose();
}
```

## 9.6 错误处理最佳实践

### 分解运算的错误检查

Eigen的分解类提供`info()`方法检查计算是否成功：

```cpp
#include <Eigen/Dense>
#include <iostream>

int main() {
    Eigen::Matrix3d A;
    A << 1, 2, 3,
         4, 5, 6,
         7, 8, 9;
    
    // 特征值分解
    Eigen::SelfAdjointEigenSolver<Eigen::Matrix3d> eigensolver(A.transpose() * A);
    if (eigensolver.info() != Eigen::Success) {
        std::cerr << "特征值分解失败!\n";
        return -1;
    }
    std::cout << "特征值:\n" << eigensolver.eigenvalues() << "\n";
    
    // LU分解求解线性方程
    Eigen::MatrixXd B = Eigen::MatrixXd::Random(3, 1);
    Eigen::FullPivLU<Eigen::MatrixXd> lu(A);
    if (!lu.isInvertible()) {
        std::cerr << "矩阵不可逆!\n";
        return -1;
    }
    Eigen::VectorXd x = lu.solve(B);
    if (lu.info() != Eigen::Success) {
        std::cerr << "求解失败!\n";
        return -1;
    }
    
    // SVD分解：
    // - BDCSVD：通常更适合较大矩阵
    // - JacobiSVD：通常更适合较小矩阵或更强调精度的场景
    // Eigen 5.x 推荐使用编译时模板参数指定 thin/full U/V 选项
    Eigen::BDCSVD<Eigen::MatrixXd, Eigen::ComputeThinU | Eigen::ComputeThinV> svd(A);
    if (svd.info() != Eigen::Success) {
        std::cerr << "SVD分解失败!\n";
        return -1;
    }
    
    return 0;
}
```

### 安全的矩阵求逆

```cpp
#include <Eigen/Dense>
#include <stdexcept>

Eigen::MatrixXd safe_inverse(const Eigen::MatrixXd& A, double threshold = 1e-10) {
    if (A.rows() != A.cols()) {
        throw std::invalid_argument("矩阵必须是方阵");
    }
    
    // 这里计算 thin U/V，是因为后续需要调用 solve() 来构造稳定的逆或伪逆
    // Eigen 5.x 推荐使用编译时模板参数指定 thin/full U/V 选项
    Eigen::BDCSVD<Eigen::MatrixXd, Eigen::ComputeThinU | Eigen::ComputeThinV> svd(A);
    double cond = svd.singularValues()(0) / 
                  svd.singularValues()(svd.singularValues().size() - 1);
    
    if (cond > 1.0 / threshold) {
        std::cerr << "警告：矩阵病态，条件数 = " << cond << "\n";
        return svd.solve(Eigen::MatrixXd::Identity(A.rows(), A.cols()));
    }
    
    return A.inverse();
}
```

### 数值稳定性检查

```cpp
bool check_matrix_valid(const Eigen::MatrixXd& A, const std::string& name = "矩阵") {
    if (A.hasNaN()) {
        std::cerr << "错误：" << name << "包含NaN值\n";
        return false;
    }
    if (!A.allFinite()) {
        std::cerr << "错误：" << name << "包含Inf值\n";
        return false;
    }
    return true;
}

// 使用示例
Eigen::MatrixXd A = Eigen::MatrixXd::Random(3, 3);
Eigen::MatrixXd B = A.inverse();
if (!check_matrix_valid(B, "逆矩阵")) {
    // 处理错误
}
```

## 常见问题 (FAQ)

**Q: 调试模式下程序很慢，如何优化？**

A: 发布模式编译：
```bash
g++ -O3 -DNDEBUG -DEIGEN_NO_DEBUG ...
```

**Q: 如何启用Eigen的详细调试信息？**

A: 
```cpp
#define EIGEN_RUNTIME_NO_MALLOC  // 检测堆分配
#define EIGEN_INITIALIZE_MATRICES_BY_NAN  // 初始化矩阵为NaN
```

**Q: 程序在矩阵运算时崩溃，如何定位？**

A: 
1. 确保使用调试模式编译（不加`-DNDEBUG`）
2. 使用`Eigen::Matrix`的`.data()`检查原始内存
3. 使用Valgrind检测内存错误：
```bash
valgrind --tool=memcheck --leak-check=full ./myapp
```

## 练习题

1. **调试练习**: 编写一个包含常见错误的程序，练习使用调试技巧定位问题。
2. **性能分析**: 使用提供的Profiler类分析一个复杂算法的性能瓶颈。

> **对应官方文档**：[Unaligned array assertion](https://eigen.tuxfamily.org/dox/group__TopicUnalignedArrayAssert.html) | [Structures Having Eigen Members](https://eigen.tuxfamily.org/dox/group__TopicStructHavingEigenMembers.html) | [Using STL Containers with Eigen](https://eigen.tuxfamily.org/dox/group__TopicStlContainers.html)

---
