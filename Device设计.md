## 单例类

FineMote 中多次使用了**单例类**，诸如 `CAN_Base<id>` 和 `DeviceScheduler` 等设备都用单例类实现，可以通过类作用域下的 `GetInstance()` 函数取得，不允许拷贝构造/赋值。

___

## 设备基础 `DeviceBase` 与 `DeviceScheduler`

FineMote 的核心设计思想是将所有**硬件资源、抽象组件、业务任务**都封装为**DeviceBase派生类对象**，通过**DeviceScheduler**周期性访问。

___

### 抽象设备基类 `DeviceBase`

在 [DeviceBase.hpp](../Devices/DeviceBase/DeviceBase.hpp#12) 中定义了**抽象设备基类** `DeviceBase`。

公有函数接口：  
- - `void Update()` 用于更新状态；  
- - `void Handle()` 用于下放指令。

保护成员：  
- - `const uint32_t divisionFactor` 表示分频因子。

私有成员：  
- - `bool updated` 表示更新状态。

*其中，`Handle()` 为强制重实现的纯虚函数。*

<details><summary>查看DeviceBase定义</summary>

```cpp
class DeviceBase {
public:
    virtual void Handle() = 0;

    virtual void Update();

    explicit DeviceBase(uint32_t divisionFactor = 1);

    virtual ~DeviceBase();

    friend class DeviceScheduler;

protected:
    const uint32_t divisionFactor;

private:
    bool updated = false;
};
```
</details>

___

### 设备调度器 `DeviceScheduler`

在 [DeviceScheduler.hpp](../Devices/DeviceBase/DeviceScheduler.hpp#108) 中定义了**设备调度器类** `DeviceScheduler`，为单例类。

公有函数接口：  
- `static DeviceScheduler& GetInstance()` 用于获取单例对象；  
- `void RegisterDevice(DeviceBase* device)` 用于注册设备对象，由 `DeviceBase` 派生类的构造函数调用，**无需手动注册**；  
- `void Start()` 用于启动调度器。

私有成员：  
- `etl::vector<Bucket, MAX_BUCKETS> buckets_` 用于存储设备对象；  
- `bool runnint_` 记录调度器运行状态。

<details><summary>查看DeviceScheduler定义</summary>

```cpp
class DeviceScheduler {
public:
    static DeviceScheduler& GetInstance();

    void RegisterDevice(DeviceBase* device);

    void Start();

private:
    [[noreturn]] static void* BucketThreadFunc(void* arg);

    etl::vector<Bucket, MAX_BUCKETS> buckets_ {};
    bool runnint_ { false };
};
```
</details>