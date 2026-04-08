# 六、几何变换篇：旋转与变换

本篇介绍Eigen的几何变换功能，包括旋转矩阵、四元数、欧拉角和仿射变换。

## 6.1 旋转表示方式对比

三维空间中的旋转有多种数学表示方式，各有优劣：

| 表示方式     | Eigen类名     | 参数数量 | 优点               | 缺点                    | 适用场景           |
| ------------ | ------------- | -------- | ------------------ | ----------------------- | ------------------ |
| **旋转矩阵** | `Matrix3d`    | 9        | 直观、易于理解     | 冗余(6个约束)、需正交化 | 矩阵运算、变换复合 |
| **欧拉角**   | `EulerAngles` | 3        | 直观、参数少       | 万向节锁、顺序依赖      | 用户界面、角度输入 |
| **轴角**     | `AngleAxis`   | 4        | 无奇异性、直观     | 插值复杂                | 单次旋转定义       |
| **四元数**   | `Quaternion`  | 4        | 无奇异性、插值简单 | 不直观                  | 动画、姿态估计     |

## 6.2 四元数详解

**什么是四元数？**

四元数是一种扩展复数的数学概念，由爱尔兰数学家Hamilton于1843年发明。一个四元数表示为：

```
q = w + xi + yj + zk
```

其中 w 是实部（标量部分），(x, y, z) 是虚部（向量部分），i, j, k 满足：
- i² = j² = k² = ijk = -1

**为什么四元数适合表示旋转？**

1. **紧凑性**：仅需4个数（vs 旋转矩阵9个数）
2. **无奇异性**：不存在万向节锁问题
3. **插值友好**：球面线性插值(SLERP)可平滑过渡
4. **数值稳定**：归一化即可恢复正交性

**四元数与旋转的关系**：

绕单位向量 **n** 旋转角度 θ 的四元数为：
```
q = cos(θ/2) + sin(θ/2)(nₓi + nᵧj + n_zk)
```

```cpp
#include <Eigen/Geometry>
#include <iostream>
#include <cmath>

int main() {
    // ========== 四元数创建方式 ==========
    
    // 方式1：直接赋值
    // 注意：Eigen中四元数构造函数参数顺序为 (w, x, y, z)
    // 但 coeffs() 返回的顺序是 (x, y, z, w)
    Eigen::Quaterniond q1(1, 0, 0, 0);  // 单位四元数 (w=1, x=0, y=0, z=0)，表示无旋转
    std::cout << "单位四元数 coeffs(): " << q1.coeffs().transpose() << "\n";
    // 输出: 单位四元数 coeffs(): 0 0 0 1  (顺序为 x, y, z, w)
    std::cout << "四元数元素: w=" << q1.w() << ", x=" << q1.x() 
              << ", y=" << q1.y() << ", z=" << q1.z() << "\n";
    // 输出: 四元数元素: w=1, x=0, y=0, z=0
    
    // 方式2：从轴角构造（绕Z轴旋转45度）
    Eigen::AngleAxisd rotation_vector(M_PI / 4, Eigen::Vector3d(0, 0, 1));
    Eigen::Quaterniond q2(rotation_vector);
    std::cout << "绕Z轴45度: " << q2.coeffs().transpose() << "\n";
    // 输出: 绕Z轴45度: 0 0 0.382683 0.92388
    
    // 方式3：从旋转矩阵构造
    Eigen::Matrix3d R = rotation_vector.toRotationMatrix();
    Eigen::Quaterniond q3(R);
    
    // 方式4：从两个向量构造（从v1旋转到v2的最短旋转）
    Eigen::Vector3d v1(1, 0, 0), v2(0, 1, 0);
    Eigen::Quaterniond q4 = Eigen::Quaterniond::FromTwoVectors(v1, v2);
    std::cout << "X轴到Y轴的旋转: " << q4.coeffs().transpose() << "\n";
    // 输出: X轴到Y轴的旋转: 0 0 0.707107 0.707107
    
    // ========== 四元数基本操作 ==========
    
    // 归一化（确保表示有效旋转，非常重要！）
    q2.normalize();
    
    // 获取旋转矩阵
    Eigen::Matrix3d R_from_q = q2.toRotationMatrix();
    std::cout << "\n旋转矩阵:\n" << R_from_q << "\n";
    // 输出:
    // 旋转矩阵:
    //  0.707107 -0.707107         0
    //  0.707107  0.707107         0
    //         0         0         1
    
    // 旋转向量
    Eigen::Vector3d v(1, 0, 0);
    Eigen::Vector3d v_rotated = q2 * v;  // 等价于 q2.toRotationMatrix() * v
    std::cout << "旋转后的向量: " << v_rotated.transpose() << "\n";
    // 输出: 旋转后的向量: 0.707107 0.707107 0
    
    // ========== 四元数复合（旋转串联）==========
    
    // 先绕X轴90度，再绕Z轴90度
    Eigen::Quaterniond qx(Eigen::AngleAxisd(M_PI / 2, Eigen::Vector3d::UnitX()));
    Eigen::Quaterniond qz(Eigen::AngleAxisd(M_PI / 2, Eigen::Vector3d::UnitZ()));
    
    // 四元数乘法：q_combined = qz * qx 表示先qx后qz
    Eigen::Quaterniond q_combined = qz * qx;
    
    // 验证：旋转X轴
    Eigen::Vector3d x_axis(1, 0, 0);
    Eigen::Vector3d result = q_combined * x_axis;
    std::cout << "复合旋转后的X轴: " << result.transpose() << "\n";
    // 输出: 复合旋转后的X轴: 0 0 1
    
    // ========== 四元数插值（SLERP）==========
    
    // 球面线性插值：在两个旋转之间平滑过渡
    Eigen::Quaterniond q_start = Eigen::Quaterniond::Identity();
    Eigen::Quaterniond q_end(Eigen::AngleAxisd(M_PI / 2, Eigen::Vector3d::UnitZ()));
    
    // t=0时为q_start，t=1时为q_end，t=0.5为中间旋转
    for (double t : {0.0, 0.25, 0.5, 0.75, 1.0}) {
        Eigen::Quaterniond q_interp = q_start.slerp(t, q_end);
        Eigen::AngleAxisd aa(q_interp);
        std::cout << "t=" << t << ": 旋转角度 " << aa.angle() * 180 / M_PI << "度\n";
    }
    // 输出:
    // t=0: 旋转角度 0度
    // t=0.25: 旋转角度 22.5度
    // t=0.5: 旋转角度 45度
    // t=0.75: 旋转角度 67.5度
    // t=1: 旋转角度 90度
    
    return 0;
}
```

**四元数使用注意事项**：

1. **必须归一化**：数值计算误差可能导致四元数不再单位化，定期调用`normalize()`
2. **乘法顺序**：`q1 * q2` 表示先应用 q2，再应用 q1（与矩阵乘法一致）
3. **双倍覆盖**：q 和 -q 表示相同的旋转，比较时需注意

## 6.3 欧拉角

**什么是欧拉角？**

欧拉角使用三个角度描述三维旋转，由数学家欧拉于1775年提出。常见的约定包括：

| 约定名称          | 旋转顺序       | 应用领域           |
| ----------------- | -------------- | ------------------ |
| **ZYX（内旋）**   | 偏航→俯仰→滚转 | 航空、航海、机器人 |
| **XYZ（固定轴）** | X→Y→Z          | 计算机图形学       |
| **ZYZ**           | Z→Y→Z          | 机器人学、力学     |

**万向节锁问题**：

当中间轴旋转±90°时，第一轴和第三轴重合，丢失一个自由度。

```cpp
#include <Eigen/Geometry>
#include <iostream>
#include <cmath>

int main() {
    // ========== 欧拉角创建 ==========
    
    // 定义欧拉角（弧度）
    double yaw = M_PI / 6;    // 30度，绕Z轴
    double pitch = M_PI / 4;  // 45度，绕Y轴
    double roll = M_PI / 3;   // 60度，绕X轴
    
    // 从欧拉角创建旋转矩阵（ZYX顺序，内旋）
    // 内旋：依次绕物体自身的Z、Y、X轴旋转
    Eigen::Matrix3d R;
    R = Eigen::AngleAxisd(yaw, Eigen::Vector3d::UnitZ()) *
        Eigen::AngleAxisd(pitch, Eigen::Vector3d::UnitY()) *
        Eigen::AngleAxisd(roll, Eigen::Vector3d::UnitX());
    
    std::cout << "旋转矩阵（ZYX顺序）:\n" << R << "\n\n";
    
    // ========== 从旋转矩阵提取欧拉角 ==========
    
    // eulerAngles(2, 1, 0) 表示ZYX顺序
    // 返回值范围：[0, π] × [-π, π] × [-π, π]
    Eigen::Vector3d euler = R.eulerAngles(2, 1, 0);
    std::cout << "提取的欧拉角（弧度）: " << euler.transpose() << "\n";
    std::cout << "提取的欧拉角（度）: " 
              << (euler * 180 / M_PI).transpose() << "\n\n";
    // 输出:
    // 提取的欧拉角（弧度）: 0.523599 0.785398 1.0472
    // 提取的欧拉角（度）: 30 45 60
    
    // ========== 万向节锁演示 ==========
    
    std::cout << "===== 万向节锁演示 =====\n";
    
    // 当pitch = ±90°时，发生万向节锁
    double pitch_locked = M_PI / 2;
    Eigen::Matrix3d R_locked;
    R_locked = Eigen::AngleAxisd(yaw, Eigen::Vector3d::UnitZ()) *
               Eigen::AngleAxisd(pitch_locked, Eigen::Vector3d::UnitY()) *
               Eigen::AngleAxisd(roll, Eigen::Vector3d::UnitX());
    
    // 尝试提取欧拉角
    Eigen::Vector3d euler_locked = R_locked.eulerAngles(2, 1, 0);
    std::cout << "pitch=90°时提取的欧拉角: " << (euler_locked * 180 / M_PI).transpose() << "\n";
    // 输出可能不稳定，因为yaw和roll效果相同
    
    // ========== 避免万向节锁的建议 ==========
    
    std::cout << "\n避免万向节锁的方法:\n";
    std::cout << "1. 使用四元数代替欧拉角进行旋转计算\n";
    std::cout << "2. 限制pitch角度范围（如-89°到89°）\n";
    std::cout << "3. 在需要用户输入时才使用欧拉角，内部计算用四元数\n";
    
    return 0;
}
```

**Eigen 5.0兼容性说明**：Eigen 5.0中欧拉角的返回值形式更加规范，可能导致与旧版本不同的输出顺序或范围。如需跨版本兼容，建议显式处理返回值。

## 6.4 仿射变换

**什么是变换矩阵？**

变换矩阵是4×4的齐次变换矩阵，综合表示旋转和平移：

```
T = | R₃ₓ₃  t₃ₓ₁ |
    | 0₁ₓ₃   1    |

其中 R 是3×3旋转矩阵，t 是3×1平移向量
```

**为什么使用齐次坐标？**

- 统一表示旋转和平移
- 变换复合简化为矩阵乘法
- 便于处理刚体运动链

```cpp
#include <Eigen/Geometry>
#include <iostream>
#include <cmath>

int main() {
    // ========== 创建变换矩阵 ==========
    
    // 方式1：分别设置旋转和平移
    Eigen::Affine3d T1 = Eigen::Affine3d::Identity();
    T1.rotate(Eigen::AngleAxisd(M_PI / 4, Eigen::Vector3d::UnitZ()));
    T1.pretranslate(Eigen::Vector3d(1, 2, 3));
    
    std::cout << "变换矩阵T1:\n" << T1.matrix() << "\n\n";
    // 输出:
    // 变换矩阵T1:
    //  0.707107 -0.707107         0         1
    //  0.707107  0.707107         0         2
    //         0         0         1         3
    //         0         0         0         1
    
    // 方式2：链式构造（推荐）
    // 注意：乘法顺序从右到左执行
    Eigen::Affine3d T2 = 
        Eigen::Translation3d(1, 2, 3) *                    // 先平移
        Eigen::AngleAxisd(M_PI / 4, Eigen::Vector3d::UnitZ());  // 再旋转
    
    // ========== 应用变换 ==========
    
    Eigen::Vector3d point(1, 0, 0);
    Eigen::Vector3d transformed = T2 * point;
    
    std::cout << "原始点: " << point.transpose() << "\n";
    std::cout << "变换后: " << transformed.transpose() << "\n\n";
    // 输出:
    // 原始点: 1 0 0
    // 变换后: 1.70711 2.70711 3
    
    // ========== 变换复合 ==========
    
    Eigen::Affine3d T_a = Eigen::Translation3d(1, 0, 0) * Eigen::Affine3d::Identity();
    Eigen::Affine3d T_b = Eigen::Translation3d(0, 1, 0) * Eigen::Affine3d::Identity();
    
    // T_combined = T_b * T_a 表示先T_a后T_b
    Eigen::Affine3d T_combined = T_b * T_a;
    
    Eigen::Vector3d origin(0, 0, 0);
    std::cout << "原点经T_a后: " << (T_a * origin).transpose() << "\n";
    std::cout << "再经T_b后: " << (T_combined * origin).transpose() << "\n\n";
    // 输出:
    // 原点经T_a后: 1 0 0
    // 再经T_b后: 1 1 0
    
    // ========== 逆变换 ==========
    
    Eigen::Affine3d T_inv = T2.inverse();
    Eigen::Vector3d back = T_inv * transformed;
    
    std::cout << "逆变换还原: " << back.transpose() << "\n";
    std::cout << "还原误差: " << (back - point).norm() << "\n\n";
    // 输出:
    // 逆变换还原: 1 0 0
    // 还原误差: 0
    
    // ========== 提取旋转和平移 ==========
    
    Eigen::Matrix3d R = T2.rotation();
    Eigen::Vector3d t = T2.translation();
    
    std::cout << "旋转部分:\n" << R << "\n";
    std::cout << "平移部分: " << t.transpose() << "\n";
    
    return 0;
}
```

## 6.5 缩放变换

```cpp
#include <Eigen/Geometry>
#include <iostream>

int main() {
    // ========== 各向同性缩放 ==========
    Eigen::UniformScaling<double> uniform_scale(2.0);  // 各方向放大2倍
    
    // ========== 各向异性缩放 ==========
    Eigen::DiagonalMatrix<double, 3> aniso_scale(2, 3, 4);  // X放大2倍，Y放大3倍，Z放大4倍
    
    // ========== 完整变换链 ==========
    // 变换顺序：缩放 → 旋转 → 平移（从右到左）
    Eigen::Quaterniond q(Eigen::AngleAxisd(M_PI / 4, Eigen::Vector3d::UnitZ()));
    
    Eigen::Affine3d full_transform = 
        Eigen::Translation3d(1, 0, 0) *    // 最后平移
        q *                                 // 然后旋转
        Eigen::Scaling(2.0, 3.0, 4.0);      // 先缩放
    
    Eigen::Vector3d p(1, 1, 1);
    Eigen::Vector3d result = full_transform * p;
    
    std::cout << "完整变换结果: " << result.transpose() << "\n";
    // 输出: 完整变换结果: -0.828427 4.24264 4
    
    return 0;
}
```

## 6.6 实战：三维刚体变换

```cpp
#include <Eigen/Geometry>
#include <iostream>
#include <vector>
#include <cmath>

// 3自由度机械臂的正向运动学
class RobotArm3DOF {
private:
    std::vector<double> link_lengths_;  // 各连杆长度
    
public:
    RobotArm3DOF() : link_lengths_({1.0, 0.8, 0.5}) {}
    
    // 计算末端执行器位置
    Eigen::Vector3d forwardKinematics(const std::vector<double>& joint_angles) {
        if (joint_angles.size() != 3) {
            throw std::invalid_argument("需要3个关节角度");
        }
        
        // 从基坐标系开始
        Eigen::Affine3d T = Eigen::Affine3d::Identity();
        
        for (size_t i = 0; i < 3; ++i) {
            // 旋转关节（绕Z轴）
            T.rotate(Eigen::AngleAxisd(joint_angles[i], Eigen::Vector3d::UnitZ()));
            // 沿X轴平移到下一个关节
            T.translate(Eigen::Vector3d(link_lengths_[i], 0, 0));
        }
        
        return T.translation();
    }
    
    // 计算完整的末端位姿
    Eigen::Affine3d forwardKinematicsFull(const std::vector<double>& joint_angles) {
        Eigen::Affine3d T = Eigen::Affine3d::Identity();
        
        for (size_t i = 0; i < 3; ++i) {
            T.rotate(Eigen::AngleAxisd(joint_angles[i], Eigen::Vector3d::UnitZ()));
            T.translate(Eigen::Vector3d(link_lengths_[i], 0, 0));
        }
        
        return T;
    }
    
    // 打印工作空间边界（简化版）
    void printWorkspaceBoundary() {
        std::cout << "工作空间半径范围: [" 
                  << link_lengths_[2] << ", " 
                  << (link_lengths_[0] + link_lengths_[1] + link_lengths_[2]) 
                  << "]\n";
    }
};

int main() {
    RobotArm3DOF arm;
    arm.printWorkspaceBoundary();
    
    // 设置关节角度（弧度）
    std::vector<double> angles = {M_PI / 6, M_PI / 4, M_PI / 3};
    
    std::cout << "\n关节角度（度）:\n";
    for (size_t i = 0; i < angles.size(); ++i) {
        std::cout << "  关节" << i+1 << ": " << angles[i] * 180 / M_PI << "°\n";
    }
    
    // 计算末端位置
    Eigen::Vector3d end_pos = arm.forwardKinematics(angles);
    std::cout << "\n末端执行器位置: " << end_pos.transpose() << "\n";
    
    // 计算末端姿态
    Eigen::Affine3d end_pose = arm.forwardKinematicsFull(angles);
    Eigen::Vector3d end_orientation = end_pose.rotation() * Eigen::Vector3d::UnitX();
    std::cout << "末端执行器朝向: " << end_orientation.transpose() << "\n";
    
    // 示例输出:
    // 工作空间半径范围: [0.5, 2.3]
    // 
    // 关节角度（度）:
    //   关节1: 30°
    //   关节2: 45°
    //   关节3: 60°
    // 
    // 末端执行器位置: 1.0458 1.43934 0
    // 末端执行器朝向: -0.258819 0.965926 0
    
    return 0;
}
```

## 6.7 几何变换常见问题

**Q: 四元数和旋转矩阵如何选择？**

A: 
- **使用四元数**：姿态估计、动画插值、存储/传输
- **使用旋转矩阵**：矩阵运算密集、需要直接访问旋转后的坐标轴

**Q: 变换矩阵乘法顺序如何理解？**

A: 变换从右向左执行。`T = T1 * T2` 表示先应用 T2，再应用 T1。

**Q: 如何处理坐标系转换？**

A: 使用变换矩阵的逆。如果 T_A_B 表示从B到A的变换，则 T_B_A = T_A_B.inverse()。

---
