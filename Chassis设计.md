## 底盘组件

FineMote 的底盘组件位于 [Components/Chassis](../Components/Chassis) 目录下，定义了里程计类、底盘基类、轮组结构体等。

___

### 空实现里程计 `WithoutOdom`

在 [ChassisBase.hpp](../Components/Chassis/ChassisBase.hpp#9) 中定义了**空实现里程计类** `WithoutOdom`，用于兼容无里程计的组件，模板参数 `DOFs` 为自由度数目，支持任意维度的里程计，其公有函数接口均是**无效果**的。

<details><summary>查看WithoutOdom定义</summary>

```cpp
template<int DOFs>
class WithoutOdom {
public:
    void SetOdom(const std::array<float, DOFs>& x);

    const std::array<float, DOFs>& GetOdom();

    template<typename T>
    void UpdateOdom(T&& v, uint32_t dt);
};
```
</details>

___
    
### 平面里程计 `PlanarOdom`

在 [ChassisBase.hpp](../Components/Chassis/ChassisBase.hpp#22) 中定义了**平面里程计类** `PlanarOdom`，根据**当前位置与角度**结合**实时速度与角速度**估算新位姿，其位姿列表采用格式 `x, y, theta`，`theta` 为弧度制。

公有函数接口：  
- `void SetOdom(const std::array<float, 3>& x)` 用于设置里程计位姿；  
- `const std::array<float, 3>& GetOdom()` 返回里程计位姿；  
- `void UpdateOdom(const std::array<float, 3>& v, uint32_t dt)` 根据传入速度和时间增量，**离散积分**更新位姿，注意 `v` 遵从 `vx, vy, w` 格式，`dt` 是时间增量，单位为毫秒。

私有成员：  
- `std::array<float, 3> estimatedX` 存储估算位姿，

<details><summary>查看PlanarOdom定义</summary>

```cpp
class PlanarOdom {
public:
    void SetOdom(const std::array<float, 3>& x);

    const std::array<float, 3>& GetOdom();

    void UpdateOdom(const std::array<float, 3>& v, uint32_t dt);

private:
    std::array<float, 3> estimatedX = { 0 };
};
```
</details>

___

### 抽象底盘基类 `ChassisBase`

在 [ChassisBase.hpp](../Components/Chassis/ChassisBase.hpp#42) 中定义了**抽象底盘基类** `ChassisBase`，继承自 `DeviceBase`，使用**里程计类** `OdomPolicy` 作为模板参数，其成员速度均遵从 `vx, vy, w` 的格式。

公有函数接口：  
- `void InverseKinematics(std::array<float, 3>&)` 用于逆运动学计算，将**底盘速度**转换为**轮组角度与速度**；  
- `void ForwardKinematics()` 用于正运动学计算，根据**轮组反馈**估算**实际速度**；  
- `template<typename T> void SetVelocity(T&& v)` 用于设置目标速度，***用户应使用该接口***。

保护成员：  
- `OdomPolicy odom` 存储里程计对象；  
- `std::array<float, 3> estimatedV` 存储估算速度。
- `std::array<float, 3> targetV` 存储目标速度；  

*其中，`InverseKinematics()` 和 `ForwardKinematics()` 为强制重实现的纯虚函数。*

<details><summary>查看ChassisBase定义</summary>

```cpp
template<typename OdomPolicy>
class ChassisBase: public DeviceBase {
public:
    virtual void InverseKinematics(std::array<float, 3>&) = 0;
    virtual void ForwardKinematics() = 0;

    template<typename T>
    void SetVelocity(T&& v);

protected:
    OdomPolicy odom;

    std::array<float, 3> targetV = { 0 };
    std::array<float, 3> estimatedV = { 0 };
};
```
</details>

___

## POV_Chassis

`POV_Chassis` 是 FineMote 提供的POV底盘实现，包含若干舵轮与一里程计选项。

### 舵轮结构体 `Swerve_t`

在 [POV_Chassis.hpp](../Components/Chassis/POV_Chassis.hpp#15) 中定义了**舵轮结构体** `Swerve_t`，包含转向电机与驱动电机，记录了安装信息。

电机成员：  
- `MotorBase* steerMotor` 指向转向电机对象；  
- `MotorBase* driveMotor` 指向驱动电机对象。

信息成员：  
- `float lx` 表示舵轮沿底盘 x 轴的安装位置；  
- `float ly` 表示舵轮沿底盘 y 轴的安装位置；  
- `float zeroPosition` 表示舵轮的零位角度偏移。

<details><summary>查看Swerve_t定义</summary>

```cpp
using Swerve_t = struct Swerve_t {
    MotorBase* steerMotor;
    MotorBase* driveMotor;
    float lx;
    float ly;
    float zeroPosition;
};
```
</details>

### 底盘类 `POV_Chassis`

在 [POV_Chassis.hpp](../Components/Chassis/POV_Chassis.hpp#23) 中定义了**舵轮底盘类** `POV_Chassis`，继承自 `ChassisBase<OdomPolicy>`，使用舵轮个数 `N` 与里程计类 `OdomPolicy` 作为模板参数，默认不使用里程计。

构造函数：  
- `POV_Chassis(const float _wheelDiameter, std::array<Swerve_t, N>&& configs)` 构造函数，传入轮子直径与舵轮列表，

<details><summary>查看POV_Chassis定义</summary>

```cpp
template<size_t N, typename OdomPolicy = WithoutOdom<3>>
class POV_Chassis: public ChassisBase<OdomPolicy> {
public:
    POV_Chassis(const float _wheelDiameter, std::array<Swerve_t, N>&& configs);

    void InverseKinematics(std::array<float, 3>& v) final;

    void ForwardKinematics() final;

    void Handle() final;

private:
    std::array<Swerve_t, N> modules;
    std::array<Matrixf<3, 3>, N> Jn;
    std::array<Matrixf<2, 3>, N> Hn;
    std::array<Matrixf<3, 1>, N> Xn;
    Matrixf<3, 3> Q;
    Matrixf<2, 2> B;
    const float alpha = 0.5;
    const float wheelDiameter;
};
```
</details>