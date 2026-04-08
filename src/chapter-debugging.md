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

### 错误3：未定义引用

```
undefined reference to `pthread_create'
```

**解决方案**：
```bash
g++ ... -lpthread  # 链接pthread库
# 或使用CMake
target_link_libraries(myapp pthread)
```

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

// 或使用条件数 (Eigen 5.0+: 使用编译时模板参数)
Eigen::JacobiSVD<Eigen::Matrix2d, Eigen::ComputeFullU | Eigen::ComputeFullV> svd(A);
double cond = svd.singularValues()(0) / svd.singularValues()(svd.singularValues().size() - 1);
if (cond > 1e10) {
    std::cerr << "警告：矩阵病态，条件数 = " << cond << "\n";
}
```

### 错误3：内存对齐错误

**问题根源**：

Eigen使用SIMD指令（SSE/AVX）优化性能，要求数据按16字节或32字节对齐。当使用`new`、`std::vector`或STL容器时，可能因对齐问题导致崩溃。

```cpp
struct MyClass {
    int x;
    Eigen::Vector4d v;  // 需要16字节对齐
};

MyClass* obj = new MyClass[10];  // 可能崩溃
std::vector<MyClass> vec;        // 可能崩溃
```

**解决方案**：

**方案1：使用EIGEN_MAKE_ALIGNED_OPERATOR_NEW**

```cpp
struct MyClass {
    EIGEN_MAKE_ALIGNED_OPERATOR_NEW  // 重载new/delete操作符
    int x;
    Eigen::Vector4d v;
};

MyClass* obj = new MyClass;  // 现在可以正常工作
std::vector<MyClass> vec;    // 也可以正常使用
```

**方案2：使用Eigen提供的内存分配器**

```cpp
#include <Eigen/StdVector>

// 使用Eigen的对齐分配器
std::vector<Eigen::Vector4d, Eigen::aligned_allocator<Eigen::Vector4d>> vec;

// 或使用宏简化（推荐）
EIGEN_DEFINE_STL_VECTOR_SPECIALIZATION(Eigen::Vector4d)
std::vector<Eigen::Vector4d> vec;  // 现在可以正常使用
```

**方案3：禁用对齐（不推荐，性能损失）**

```cpp
Eigen::Matrix<double, 4, 1, Eigen::DontAlign> v;  // 禁用对齐
```

**最佳实践**：
- 固定大小类型（`Vector4d`, `Matrix4f`等）需要注意对齐
- 动态大小类型（`VectorXd`, `MatrixXd`）自动处理对齐
- 在结构体/类中使用固定大小Eigen类型时，添加`EIGEN_MAKE_ALIGNED_OPERATOR_NEW`
- 使用`Eigen::aligned_allocator`处理STL容器

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
double cond = svd.singularValues()(0) / svd.singularValues().last();
if (cond > 1e10) {
    std::cerr << "矩阵病态，条件数 = " << cond << "\n";
}
```

### 错误处理最佳实践

| 场景         | 检查方法                   | 处理方式           |
| ------------ | -------------------------- | ------------------ |
| 矩阵求逆     | 检查条件数                 | 使用伪逆或正则化   |
| 线性求解     | `info() != Success`        | 使用最小二乘解     |
| Cholesky分解 | `info() == NumericalIssue` | 矩阵非正定，改用LU |
| 数值溢出     | `hasNaN()`, `allFinite()`  | 检查输入数据       |

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

// 简单的性能分析器
class Profiler {
    static std::map<std::string, std::pair<int, double>> stats;
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
    
    // SVD分解（Eigen 5.0推荐使用BDCSVD替代JacobiSVD）
    // Eigen 5.0+: 使用编译时模板参数指定计算选项
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
    
    // Eigen 5.0+: 使用编译时模板参数，不计算U和V矩阵以提高性能
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

---
