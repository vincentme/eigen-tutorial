## C. API 快速索引

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
| 通用块   | `A.block(i, j, rows, cols)`       | 4.1      |
| 行       | `A.row(i)`                        | 4.1      |
| 列       | `A.col(j)`                        | 4.1      |
| 左上角   | `A.topLeftCorner(rows, cols)`     | 4.1      |
| 右下角   | `A.bottomRightCorner(rows, cols)` | 4.1      |
| 向量头部 | `v.head(n)`                       | 4.1      |
| 向量尾部 | `v.tail(n)`                       | 4.1      |
| 向量段   | `v.segment(start, n)`             | 4.1      |

### 矩阵分解与求解

| 功能         | API                                                         | 所在章节 |
| ------------ | ----------------------------------------------------------- | -------- |
| LU分解       | `PartialPivLU`, `FullPivLU`                                 | 5.1, 5.4 |
| QR分解       | `HouseholderQR`, `ColPivHouseholderQR`                      | 5.1, 5.4 |
| 完全正交分解 | `CompleteOrthogonalDecomposition`                           | 5.1      |
| Cholesky分解 | `LLT`, `LDLT`                                               | 5.1, 5.4 |
| SVD分解      | `JacobiSVD`, `BDCSVD`                                       | 5.1, 5.3 |
| 特征值分解   | `EigenSolver`, `SelfAdjointEigenSolver`                     | 5.2      |
| 求解线性方程 | `solver.solve(b)`, `completeOrthogonalDecomposition().solve(b)` | 5.1      |

### 几何变换

| 功能      | API                                                         | 所在章节 |
| --------- | ----------------------------------------------------------- | -------- |
| 旋转矩阵  | `Rotation2D`, `AngleAxis`, `toRotationMatrix()`             | 6.1, 6.2 |
| 四元数    | `Quaterniond`                                               | 6.2      |
| 欧拉角    | `MatrixBase::eulerAngles(a0, a1, a2)`                       | 6.3      |
| 变换矩阵  | `Transform`, `Affine3d`, `Isometry3d`                       | 6.4      |
| 旋转+平移 | `Translation3d * AngleAxisd`, `Translation3d * Quaterniond` | 6.4      |

### Array 类（逐元素运算）

| 功能       | API                         | 所在章节 |
| ---------- | --------------------------- | -------- |
| 逐元素乘法 | `A.array() * B.array()`     | 4.2      |
| 逐元素除法 | `A.array() / B.array()`     | 4.2      |
| 平方根     | `A.array().sqrt()`          | 4.2      |
| 指数       | `A.array().exp()`           | 4.2      |
| 对数       | `A.array().log()`           | 4.2      |
| 幂运算     | `A.array().pow(n)`          | 4.2      |
| 三角函数   | `A.array().sin()`, `.cos()` | 4.2      |
| 绝对值     | `A.array().abs()`           | 4.2      |

## D. 常见错误速查表 

| 错误信息                                           | 可能原因                 | 解决方案                                  | 参考章节 |
| -------------------------------------------------- | ------------------------ | ----------------------------------------- | -------- |
| `Eigen/Core: No such file`                         | 头文件路径错误           | 检查`-I`路径配置                          | 1.1, 1.2 |
| `requires at least c++14`                          | C++标准过低              | 添加`-std=c++14` 或更高标准               | 9.1      |
| `JacobiSVD is not a member of Eigen`               | 模块头文件缺失           | 补充 `<Eigen/Dense>` 或按需包含模块头文件 | 9.1      |
| `incomplete type ... used in nested name specifier`| 包含层级不完整           | 检查是否缺少 `SVD` / `Eigenvalues` 等头文件 | 9.1    |
| `Assertion failed`                                 | 维度不匹配               | 验证运算兼容性                            | 9.2, 9.3 |
| 矩阵包含NaN/Inf                                    | 数值溢出或奇异矩阵       | 检查条件数或最小奇异值，必要时使用伪逆    | 9.3, 9.6 |
| 内存对齐错误                                       | 固定大小矩阵对齐敏感场景 | 使用`EIGEN_MAKE_ALIGNED_OPERATOR_NEW`等方案 | 9.2    |
| 编译缓慢                                           | 模板展开开销             | 使用预编译头、减少不必要头文件包含        | 8.3      |
| 运行时断言失败                                     | 调试模式检查             | 检查输入维度和别名问题                    | 9.2, 9.3 |

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
# 说明：Eigen 5.0.x 的最低要求是 C++14，本教程中的示例默认按 C++17 组织

# 基础编译
g++ -std=c++17 -I/path/to/eigen -O2 program.cpp

# 通用高性能编译
g++ -std=c++17 -I/path/to/eigen -O3 -march=native \
    -DEIGEN_NO_DEBUG -DNDEBUG program.cpp

# 如项目明确使用 OpenMP，再按需启用
# g++ -std=c++17 -I/path/to/eigen -O3 -march=native -fopenmp \
#     -DEIGEN_NO_DEBUG -DNDEBUG program.cpp

# 调试编译
g++ -std=c++17 -I/path/to/eigen -O0 -g program.cpp
```

## G. 推荐学习资源 

1. **官方文档**: https://eigen.tuxfamily.org/
2. **官方教程**: https://eigen.tuxfamily.org/dox/GettingStarted.html
3. **API参考**: https://eigen.tuxfamily.org/dox/modules.html
4. **GitLab仓库**: https://gitlab.com/libeigen/eigen

---

# 总结

本教程系统介绍了Eigen库的各个方面：

- **基础篇**: 了解Eigen的核心特性和设计理念
- **安装篇**: 掌握多平台安装和 CMake 集成方法
- **矩阵基础篇**: 熟练进行矩阵与向量操作
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