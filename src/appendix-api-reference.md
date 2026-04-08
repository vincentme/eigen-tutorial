## C. API快速索引 

### 矩阵创建与初始化

| 功能         | API                                   | 所在章节 |
| ------------ | ------------------------------------- | -------- |
| 固定大小矩阵 | `Matrix3d`, `Matrix4f`                | 3.1      |
| 动态大小矩阵 | `MatrixXd`, `MatrixXf`                | 3.1      |
| 向量类型     | `Vector3d`, `VectorXd`                | 3.1      |
| 零矩阵       | `Matrix::Zero(rows, cols)`            | 3.2      |
| 单位矩阵     | `Matrix::Identity(rows, cols)`        | 3.2      |
| 随机矩阵     | `Matrix::Random(rows, cols)`          | 3.2      |
| 常量矩阵     | `Matrix::Constant(rows, cols, value)` | 3.2      |

### 算术运算

| 功能     | API                         | 所在章节 |
| -------- | --------------------------- | -------- |
| 矩阵加法 | `A + B`                     | 3.3      |
| 矩阵减法 | `A - B`                     | 3.3      |
| 矩阵乘法 | `A * B`                     | 3.3      |
| 数乘     | `A * scalar`                | 3.3      |
| 转置     | `A.transpose()`             | 3.3      |
| 行列式   | `A.determinant()`           | 3.3      |
| 逆矩阵   | `A.inverse()`               | 3.3      |
| 点积     | `v1.dot(v2)`                | 3.4      |
| 叉积     | `v1.cross(v2)`              | 3.4      |
| 范数     | `v.norm()`, `v.lpNorm<1>()` | 3.4      |

### 块操作

| 功能     | API                               | 所在章节 |
| -------- | --------------------------------- | -------- |
| 通用块   | `A.block(i, j, rows, cols)`       | 3.5      |
| 行       | `A.row(i)`                        | 3.5      |
| 列       | `A.col(j)`                        | 3.5      |
| 左上角   | `A.topLeftCorner(rows, cols)`     | 3.5      |
| 右下角   | `A.bottomRightCorner(rows, cols)` | 3.5      |
| 向量头部 | `v.head(n)`                       | 3.5      |
| 向量尾部 | `v.tail(n)`                       | 3.5      |
| 向量段   | `v.segment(start, n)`             | 3.5      |

### 矩阵分解与求解

| 功能         | API                                     | 所在章节 |
| ------------ | --------------------------------------- | -------- |
| LU分解       | `PartialPivLU`, `FullPivLU`             | 4.1      |
| QR分解       | `HouseholderQR`, `ColPivHouseholderQR`  | 4.2      |
| Cholesky分解 | `LLT`, `LDLT`                           | 4.3      |
| SVD分解      | `JacobiSVD`, `BDCSVD`                   | 4.3      |
| 特征值分解   | `EigenSolver`, `SelfAdjointEigenSolver` | 4.4      |
| 求解线性方程 | `solver.solve(b)`                       | 4.1-4.3  |

### 几何变换

| 功能      | API                        | 所在章节 |
| --------- | -------------------------- | -------- |
| 旋转矩阵  | `Rotation2D`, `AngleAxis`  | 4.5      |
| 四元数    | `Quaterniond`              | 4.5      |
| 欧拉角    | `Quaternion::fromAngles()` | 4.5      |
| 变换矩阵  | `Transform`, `Isometry3d`  | 4.5      |
| 旋转+平移 | `Translation + Rotation`   | 4.5      |

### Array类（逐元素运算）

| 功能       | API                         | 所在章节 |
| ---------- | --------------------------- | -------- |
| 逐元素乘法 | `A.array() * B.array()`     | 3.6      |
| 逐元素除法 | `A.array() / B.array()`     | 3.6      |
| 平方根     | `A.array().sqrt()`          | 3.6      |
| 指数       | `A.array().exp()`           | 3.6      |
| 对数       | `A.array().log()`           | 3.6      |
| 幂运算     | `A.array().pow(n)`          | 3.6      |
| 三角函数   | `A.array().sin()`, `.cos()` | 3.6      |
| 绝对值     | `A.array().abs()`           | 3.6      |

## D. 常见错误速查表 

| 错误信息                         | 可能原因           | 解决方案                              | 参考章节 |
| -------------------------------- | ------------------ | ------------------------------------- | -------- |
| `Eigen/Core: No such file`       | 头文件路径错误     | 检查`-I`路径配置                      | 2.2      |
| `requires at least c++14`        | C++标准过低        | 添加`-std=c++14`                      | 2.4      |
| `Index out of range`             | 矩阵访问越界       | 检查索引范围                          | 6.2      |
| `Assertion failed`               | 维度不匹配         | 验证运算兼容性                        | 6.2      |
| `undefined reference to pthread` | 未链接线程库       | 添加`-lpthread`                       | 6.1      |
| 矩阵包含NaN/Inf                  | 数值溢出或奇异矩阵 | 检查条件数，使用伪逆                  | 6.5      |
| 内存对齐错误                     | 固定大小矩阵对齐   | 使用`EIGEN_MAKE_ALIGNED_OPERATOR_NEW` | 6.2      |
| 编译缓慢                         | 模板展开开销       | 使用显式实例化或预编译头              | 5.3      |
| 运行时断言失败                   | 调试模式检查       | 发布模式添加`-DNDEBUG`                | 6.3      |

## E. 快速参考表 

### 常用类型别名

| 别名          | 等价定义                           | 说明          |
| ------------- | ---------------------------------- | ------------- |
| `Matrix3d`    | `Matrix<double, 3, 3>`             | 3×3双精度矩阵 |
| `Vector3d`    | `Matrix<double, 3, 1>`             | 3维列向量     |
| `RowVector3d` | `Matrix<double, 1, 3>`             | 3维行向量     |
| `MatrixXd`    | `Matrix<double, Dynamic, Dynamic>` | 动态矩阵      |
| `VectorXd`    | `Matrix<double, Dynamic, 1>`       | 动态列向量    |
| `ArrayXXd`    | `Array<double, Dynamic, Dynamic>`  | 动态数组      |

### 常用函数

| 函数            | 说明               |
| --------------- | ------------------ |
| `setRandom()`   | 随机初始化 [-1, 1] |
| `setZero()`     | 置零               |
| `setOnes()`     | 置一               |
| `setIdentity()` | 单位矩阵           |
| `norm()`        | L2范数             |
| `squaredNorm()` | L2范数平方         |
| `dot()`         | 点积               |
| `cross()`       | 叉积（3D）         |
| `transpose()`   | 转置               |
| `inverse()`     | 逆矩阵             |
| `determinant()` | 行列式             |
| `trace()`       | 迹                 |

## F. 编译选项速查 

```bash
# 基础编译
g++ -std=c++14 -I/path/to/eigen -O2 program.cpp

# 高性能编译
g++ -std=c++17 -I/path/to/eigen -O3 -march=native -fopenmp \
    -DEIGEN_NO_DEBUG -DNDEBUG program.cpp

# 调试编译
g++ -std=c++14 -I/path/to/eigen -O0 -g program.cpp
```

## G. 推荐学习资源 

1. **官方文档**: https://eigen.tuxfamily.org/
2. **官方教程**: https://eigen.tuxfamily.org/dox/GettingStarted.html
3. **API参考**: https://eigen.tuxfamily.org/dox/modules.html
4. **GitHub仓库**: https://gitlab.com/libeigen/eigen

---

# 总结

本教程系统介绍了Eigen库的各个方面：

- **基础篇**: 了解Eigen的核心特性和设计理念
- **安装篇**: 掌握多平台安装和CMake集成方法
- **基础使用篇**: 熟练进行矩阵向量操作
- **进阶应用篇**: 掌握线性代数算法实现
- **优化篇**: 学会性能调优技巧
- **调试篇**: 具备问题排查能力
- **实战篇**: 能够应用于实际项目

通过系统学习和实践，您应该能够：
1. 熟练运用Eigen进行科学计算
2. 根据场景选择合适的算法和数据结构
3. 优化代码性能，发挥硬件潜力
4. 独立解决开发中遇到的问题

祝您在科学计算和数值分析的道路上越走越远！

---

*本教程基于Eigen 5.0.1版本编写，部分内容可能因版本更新而有所变化，请以官方文档为准。*