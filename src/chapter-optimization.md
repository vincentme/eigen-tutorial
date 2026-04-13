# 八、优化篇：性能调优技巧

## 8.1 编译优化选项

```bash
# 推荐的通用高性能编译配置
g++ -std=c++17 -O3 -march=native -DEIGEN_NO_DEBUG -DNDEBUG \
    -I/path/to/eigen -o optimized_app app.cpp

# 如项目明确使用 OpenMP，并且你已经评估过线程模型与构建环境，
# 再按需添加：
# g++ -std=c++17 -O3 -march=native -fopenmp -DEIGEN_NO_DEBUG -DNDEBUG \
#     -I/path/to/eigen -o optimized_app app.cpp

# 注意：-ffast-math 可能导致数值精度问题
# 仅在对精度要求不高的场景使用，例如图形渲染
# g++ ... -ffast-math ...  # 不推荐用于科学计算
```

**关于`-ffast-math`的警告**：

`-ffast-math`选项会启用激进的浮点优化，可能导致：
- 违反IEEE 754浮点标准
- NaN和Inf检查失效
- 数值精度降低
- 某些数学函数结果不准确

**建议**：
- 科学计算和数值分析：**不要使用**`-ffast-math`
- 图形渲染、游戏开发：可以谨慎使用
- 如需高性能，优先考虑`-O3 -march=native -ffp-contract=fast`
- `-fopenmp` 不是 Eigen 项目的默认必选项，应仅在确实需要并行化且构建环境支持时启用

### 向量化指令集

| 标志        | 指令集  | 要求                |
| ----------- | ------- | ------------------- |
| `-msse4.2`  | SSE4.2  | Intel Core 2及更新  |
| `-mavx`     | AVX     | Intel Sandy Bridge+ |
| `-mavx2`    | AVX2    | Intel Haswell+      |
| `-mavx512f` | AVX-512 | Intel Skylake-X+    |
| `-mfma`     | FMA     | 乘加融合指令        |

## 8.2 内存管理最佳实践

### 固定大小 vs 动态大小

```cpp
#include <Eigen/Dense>
#include <chrono>

void benchmark() {
    const int N = 1000000;
    
    // 固定大小（栈分配）- 在这类小矩阵场景下通常更快
    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) {
        Eigen::Matrix4d A = Eigen::Matrix4d::Random();
        Eigen::Matrix4d B = Eigen::Matrix4d::Random();
        Eigen::Matrix4d C = A * B;
    }
    auto fixed_time = std::chrono::high_resolution_clock::now() - start;
    
    // 动态大小（堆分配）- 在这类小矩阵场景下通常较慢
    start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) {
        Eigen::MatrixXd A = Eigen::MatrixXd::Random(4, 4);
        Eigen::MatrixXd B = Eigen::MatrixXd::Random(4, 4);
        Eigen::MatrixXd C = A * B;
    }
    auto dynamic_time = std::chrono::high_resolution_clock::now() - start;
    
    // 对这类小尺寸矩阵，固定大小常常会明显更快；
    // 具体收益依赖尺寸、编译器、目标平台和表达式形态
}
```

### 避免动态内存分配

```cpp
// 不推荐：运行时确定大小导致堆分配
template<typename T>
T compute(int n) {
    Eigen::VectorXd v(n);  // 堆分配
    // ...
}

// 推荐：使用模板参数在编译时确定大小
template<int N>
Eigen::Matrix<double, N, 1> compute() {
    Eigen::Matrix<double, N, 1> v;  // 栈分配
    // ...
}

// 或者使用“动态大小 + 编译期最大上限”
// 这种类型仍然是动态尺寸，只是给出一个编译期上限，便于优化小尺寸场景
template<int MaxN>
using VectorMax = Eigen::Matrix<double, Eigen::Dynamic, 1, 0, MaxN, 1>;

void compute(int n) {
    assert(n <= 100 && "n 超过编译期最大上限");
    VectorMax<100> v(n);  // 运行时长度为 n，最大不超过 100
}
```

### 预分配内存

```cpp
// 动态矩阵预分配
Eigen::MatrixXd A;
A.resize(1000, 1000);  // 一次性分配

// 多次操作时避免重复分配
Eigen::MatrixXd B(1000, 1000), C(1000, 1000);
B.setRandom();
C.setRandom();

for (int i = 0; i < 1000; ++i) {
    // A.resize(...)  // 不要这样做！
    A.noalias() = B * C;  // 当 A 与 B/C 不重叠时，可用 noalias() 避免不必要的临时对象
}
```

## 8.3 表达式模板与临时对象

Eigen的表达式模板是其高性能的核心机制，通过延迟求值避免创建临时对象。

### 什么是表达式模板？

Eigen使用表达式模板（Expression Templates）技术来优化运算。传统方式会为每个中间结果创建临时对象，而表达式模板通过延迟求值，将整个表达式编译成一个高效的循环。

**传统方式（无表达式模板）**：

```cpp
// 假设没有表达式模板
MatrixXd D = A + B + C;
// 实际执行：
// 1. temp1 = A + B;  // 创建临时矩阵
// 2. D = temp1 + C;  // 再创建临时矩阵
// 结果：多次内存分配、多次遍历数据
```

**Eigen的表达式模板**：

```cpp
MatrixXd D = A + B + C;
// 实际执行：
// 编译器生成类似以下的代码：
for (int i = 0; i < D.size(); ++i) {
    D(i) = A(i) + B(i) + C(i);
}
// 结果：零临时对象、单次遍历、编译时优化
```

### 表达式模板的工作原理

```cpp
// Eigen内部实现（简化）
template<typename Lhs, typename Rhs>
struct MatrixSum {
    const Lhs& lhs;
    const Rhs& rhs;
    
    double operator()(int i, int j) const {
        return lhs(i, j) + rhs(i, j);
    }
};

// A + B 返回的是 MatrixSum<MatrixXd, MatrixXd>，而不是新的矩阵
auto sum = A + B;  // sum是一个表达式对象，不计算结果

MatrixXd C = sum;  // 此时才真正计算（赋值时触发求值）
```

### 何时会创建临时对象？

**情况1：显式转换类型**

```cpp
MatrixXd A, B, C;
VectorXd v;

// 会创建临时对象
MatrixXd temp = A * B;        // 显式创建临时对象
v = (A * B).col(0);           // A*B的结果需要先计算出来

// 避免不必要的临时对象
C.noalias() = A * B;          // 当C与A/B不重叠时可使用noalias()
```

**情况2：表达式过于复杂**

```cpp
// 复杂表达式不一定更慢，但当存在子表达式复用、别名风险或可读性问题时，
// 显式拆分通常更容易分析和调优
MatrixXd result = A * B + C * D * E + F.transpose() * G;

// 如果需要复用中间结果，可以显式求值
MatrixXd temp1 = (C * D).eval();
MatrixXd temp2 = temp1 * E;
MatrixXd result2 = A * B + temp2 + F.transpose() * G;
```

**情况3：使用auto导致的陷阱**

```cpp
MatrixXd A, B;

// 错误：auto保存的是表达式对象，不是结果
auto sum = A + B;  // sum是表达式对象，不拥有数据

// 如果A或B在sum求值前被修改，结果可能与“保存当时结果”的直觉不一致；
// 如果它们已经销毁，则还可能出现悬空引用问题
A(0, 0) = 999;     // 修改A
MatrixXd C = sum;  // 这里得到的是“此刻再求值”的结果，而不是定义sum时就保存好的结果

// 正确做法
MatrixXd sum = A + B;  // 显式指定类型，立即求值
```

### noalias()优化

```cpp
MatrixXd A, B, C;

// 普通赋值（可能创建临时对象）
C = A * B;  // 编译器可能创建临时对象，因为C可能与A或B重叠

// 使用noalias()（承诺C不与A或B重叠）
C.noalias() = A * B;  // 零临时对象，性能最优

// 注意：如果C确实与A或B重叠，使用noalias()会导致错误结果
C = A * C;            // 正确：编译器会处理重叠
C.noalias() = A * C;  // 错误：结果不正确！
```

### 性能优化技巧

**技巧1：链式操作**

```cpp
// 推荐：链式操作，单次遍历
MatrixXd D = (A + B).cwiseProduct(C);

// 不推荐：分步操作，多次遍历
MatrixXd temp = A + B;
MatrixXd D = temp.cwiseProduct(C);
```

**技巧2：使用in-place操作**

```cpp
MatrixXd A, B;

// 推荐：原地操作
A += B;
A *= 2.0;

// 不推荐：创建新对象
A = A + B;
A = A * 2.0;
```

**技巧3：预分配结果矩阵**

```cpp
MatrixXd A(1000, 1000), B(1000, 1000);
A.setRandom();
B.setRandom();

// 推荐：预分配
MatrixXd C(1000, 1000);
C.noalias() = A * B;

// 不推荐：动态分配
MatrixXd C = A * B;  // 可能触发重新分配
```

### 常见陷阱与解决方案

| 问题         | 危险代码                           | 安全代码                                  |
| ------------ | ---------------------------------- | ----------------------------------------- |
| 别名问题     | `A = A * A`                        | `A = (A * A).eval()`                      |
| 临时对象引用 | `auto x = A + MatrixXd::Ones(n,n)` | `MatrixXd x = A + MatrixXd::Ones(n,n)`    |
| 重复计算     | `auto e = A+B; C=e*2; D=e*3`       | `MatrixXd e = (A+B).eval(); C=e*2; D=e*3` |

## 8.4 多线程并行计算

### 启用OpenMP支持

```cpp
// 编译时添加 -fopenmp
#include <Eigen/Dense>
#include <iostream>

int main() {
    // 查看Eigen使用的线程数
    std::cout << "Eigen使用的线程数: " << Eigen::nbThreads() << "\n";
    
    // 设置线程数
    Eigen::setNbThreads(4);
    
    // 或使用环境变量
    // export OMP_NUM_THREADS=4
    
    // 大型矩阵运算会自动并行化
    Eigen::MatrixXd A = Eigen::MatrixXd::Random(2000, 2000);
    Eigen::MatrixXd B = Eigen::MatrixXd::Random(2000, 2000);
    
    Eigen::MatrixXd C = A * B;  // 自动使用多线程
    
    return 0;
}
```

### 并行化策略

| 操作       | 并行化支持 | 阈值     |
| ---------- | ---------- | -------- |
| 矩阵乘法   | 是         | ~128×128 |
| 部分LU分解 | 是         | 较大矩阵 |
| QR分解     | 部分       | 较大矩阵 |
| SVD        | 否         | -        |
| 特征值分解 | 否         | -        |

## 8.5 向量化优化

```cpp
#include <Eigen/Dense>
#include <iostream>

int main() {
    // 检查向量化支持
    #ifdef __AVX2__
    std::cout << "编译时启用了AVX2\n";
    #endif
    
    #ifdef __SSE4_2__
    std::cout << "编译时启用了SSE4.2\n";
    #endif
    
    // 确保数据对齐以获得最佳向量化性能
    // Eigen的固定大小矩阵自动对齐
    Eigen::Matrix4d A;  // 自动16/32字节对齐
    
    // 动态矩阵也自动对齐
    Eigen::MatrixXd B(100, 100);
    
    // 使用Map时确保对齐
    alignas(32) double data[100];  // C++11对齐
    Eigen::Map<Eigen::VectorXd> v(data, 100);
    
    return 0;
}
```

## 8.6 性能分析技巧

```cpp
#include <Eigen/Dense>
#include <chrono>

class Timer {
    std::chrono::high_resolution_clock::time_point start;
    const char* name;
public:
    Timer(const char* n) : name(n) {
        start = std::chrono::high_resolution_clock::now();
    }
    ~Timer() {
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        std::cout << name << ": " << duration.count() << " μs\n";
    }
};

void benchmark_operations() {
    const int N = 1000;
    Eigen::MatrixXd A = Eigen::MatrixXd::Random(N, N);
    Eigen::MatrixXd B = Eigen::MatrixXd::Random(N, N);
    Eigen::VectorXd v = Eigen::VectorXd::Random(N);
    
    {
        Timer t("矩阵乘法");
        Eigen::MatrixXd C = A * B;
    }
    
    {
        Timer t("矩阵-向量乘法");
        Eigen::VectorXd w = A * v;
    }
    
    {
        Timer t("LU分解");
        Eigen::PartialPivLU<Eigen::MatrixXd> lu(A);
    }
    
    {
        Timer t("Cholesky分解");
        Eigen::LLT<Eigen::MatrixXd> llt(A.transpose() * A);
    }
}
```

## 常见问题 (FAQ)

**Q: 如何检查Eigen是否使用了向量化？**

A: 定义`EIGEN_VECTORIZE`宏查看：
```cpp
#include <Eigen/Core>
#ifdef EIGEN_VECTORIZE
std::cout << "向量化已启用\n";
#endif
```

**Q: 程序崩溃提示"unaligned memory access"**

A: 通常是结构体中包含Eigen类型导致：
```cpp
// 错误
struct MyData {
    Eigen::Vector4d v;  // 需要16字节对齐
    int x;
};

// 正确
struct MyData {
    EIGEN_MAKE_ALIGNED_OPERATOR_NEW  // 重载new以正确对齐
    Eigen::Vector4d v;
    int x;
};
```

**Q: 如何禁用多线程？**

A: 
```cpp
Eigen::setNbThreads(1);
// 或编译时定义
// -DEIGEN_DONT_PARALLELIZE
```

## 8.7 CUDA支持

Eigen 早已支持在 CUDA 内核中使用固定大小的矩阵和向量；这并不是 Eigen 5.0 才新增的能力。

**使用示例**：

```cpp
#include <Eigen/Core>

__global__ void matrix_multiply_kernel(float* result, const float* a, const float* b, int n) {
    // 在CUDA内核中使用Eigen固定大小矩阵
    Eigen::Map<const Eigen::Matrix3f> mat_a(a);
    Eigen::Map<const Eigen::Matrix3f> mat_b(b);
    Eigen::Map<Eigen::Matrix3f> mat_result(result);
    
    mat_result = mat_a * mat_b;
}

// 注意：
// 1. 适合固定大小矩阵（如 Matrix3f、Vector4d 等）
// 2. 动态大小对象通常不适合直接在 kernel 中这样使用
// 3. 需要使用 Eigen::Map 映射设备内存
// 4. 在 .cu 文件中通常需要关闭主机端 SIMD 向量化，并注意索引类型与编译器兼容性
```

**编译配置**：

```cmake
find_package(CUDA REQUIRED)
include_directories(${EIGEN_INCLUDE_DIR})

cuda_add_executable(my_cuda_app kernel.cu)
```

**限制**：
- 仅支持固定大小类型
- 不支持动态内存分配
- 部分高级功能不可用

## 练习题

1. **性能测试**: 比较固定大小和动态大小矩阵在不同尺寸下的性能差异。
2. **内存分析**: 使用`noalias()`优化矩阵链式乘法，测量性能提升。
3. **并行优化**: 测试不同线程数对大型矩阵乘法的影响，找到最优线程数。

---
