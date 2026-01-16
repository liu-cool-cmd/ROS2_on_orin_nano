环境极速搭建 (为移植做准备)

不要用官方源，速度慢到让你怀疑人生。也不要手动配 ROS 2 源，容易 GPG Key 报错。我们用“一行代码”工具。

#### 1. 换源与 ROS 2 Humble 安装
使用国内大神开发的 `fishros` 工具，它是小白和高手的共同选择。

```bash
wget http://fishros.com/install -O fishros && . fishros
```
**按提示操作：**
1.  选择 `1` (一键安装 ROS)。
2.  选择 `1` (更换系统源 -> 选清华源)。
3.  系统版本选 `humble` (Desktop 版)。
4.  **关键点：** 它会自动处理 `rosdep` (ROS 依赖管理)，选择 `rosdepc` (国内版)，这能解决 99% 的网络报错。

验证安装：
```bash
source /opt/ros/humble/setup.bash
ros2
# 出现帮助信息即成功
```
*建议：把 `source /opt/ros/humble/setup.bash` 加到 `~/.bashrc` 文件末尾，省得每次开终端都要输。*

#### 2. 配置 CUDA 环境变量 (JetPack 6 特性)
JetPack 6 基于 Ubuntu 22.04，CUDA 路径可能需要手动加一下，否则编译驱动会找不到 NVCC。

```bash
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
echo 'export ROS_DOMAIN_ID=30' >> ~/.bashrc
>设置 ROS 2 分布式通信频道 (防止干扰)
source ~/.bashrc
nvcc -V
# 应显示 CUDA 12.x 版本
```

---

### 第三阶段：驱动移植与排坑 (硬核实战)
固定usb id
先lsusb记下id

### 第一步：打开 udev 规则文件
假设你之前创建的文件名是 `99-yahboom.rules`：

```bash
sudo nano /etc/udev/rules.d/99-yahboom.rules
```
*(如果没有这个文件，可以检查一下你之前写到哪个文件里了，常见路径是 `/etc/udev/rules.d/` 目录下)*

### 第二步：修改文件内容
在编辑器中，把内容修改为只保留**底盘**和**雷达**。建议改成下面这样（加了注释更清晰）：

```bash
# 底盘 (CH340) - 对应你的 ID 1a86:7523
KERNEL=="ttyUSB*", ATTRS{idVendor}=="1a86", ATTRS{idProduct}=="7523", MODE:="0777", SYMLINK+="myserial"

# 激光雷达 (CP210x) - 对应你的 ID 10c4:ea60
KERNEL=="ttyUSB*", ATTRS{idVendor}=="10c4", ATTRS{idProduct}=="ea60", MODE:="0777", SYMLINK+="ydlidar"
```
> 注意 ttyUSB* 此处依据驱动创建的设备名更改
> ch340/341 要装驱动，源码在gayhub网盘
**操作提示：**
1.  **删除**掉那两行包含 `2bc5`（相机）的内容。
2.  按下 `Ctrl + O`，然后按 `Enter`（保存）。
3.  按下 `Ctrl + X`（退出编辑器）。

### 第三步：让规则立即生效
修改完文件后，系统不会自动加载，你需要手动刷新一下：

```bash
# 1. 重新加载规则文件
sudo udevadm control --reload-rules

# 2. 触发新规则生效
sudo udevadm trigger
```

### 第四步：检查结果（关键验证）
这是验证你刚才忙活半天是否成功的唯一标准：

1.  **确认底盘别名：**
    ```bash
    ls -l /dev/myserial
    ```
    *预期看到：* `... /dev/myserial -> ttyUSBX` (X可能是0或1)

2.  **确认雷达别名：**
    ```bash
    ls -l /dev/ydlidar
    ```
    *预期看到：* `... /dev/ydlidar -> ttyUSBX`
---

## 3. CH340 底盘驱动与系统排坑
Ubuntu 22.04 默认存在服务冲突且可能缺失 CH341 模块。

1. **卸载冲突服务 (重要)：**
   ```bash
   sudo apt remove brltty -y
   ```

2. **手动编译驱动 (若系统不识别 ttyCH341USB)：**
   下载官方 [CH341SER_LINUX](https://www.wch.cn/download/CH341SER_LINUX_ZIP.html) 源码并编译：
   ```bash
   cd CH341SER_LINUX/driver
   make
   sudo make load
   ```
   *注意：若安装此驱动，udev 规则中的 KERNEL 应设为 `tty*` 以匹配 `ttyCH341USB0`。*

---

## 4. 激光雷达配置 (YDLIDAR TG15)

### 4.1 安装 SDK 与驱动
```bash
mkdir -p ~/ros2_ws/src && cd ~/ros2_ws/src
git clone https://github.com/YDLIDAR/YDLidar-SDK.git
cd YDLidar-SDK && mkdir build && cd build && cmake .. && make && sudo make install

cd ~/ros2_ws/src
git clone -b humble https://github.com/YDLIDAR/ydlidar_ros2_driver.git
```

### 4.2 修复 Humble 编译报错
Humble 要求 `declare_parameter` 必须有默认值。执行以下“手术”脚本：
```bash
sed -i 's/node->declare_parameter("\([^"]*\)");/node->declare_parameter<std::string>("\1", "");/g' ~/ros2_ws/src/ydlidar_ros2_driver/src/ydlidar_ros2_driver_node.cpp
```

### 4.3 TG15 参数调优
修改 `~/ros2_ws/src/ydlidar_ros2_driver/params/ydlidar.yaml`：
*   `port`: `/dev/ydlidar`
*   `baudrate`: `512000`
*   `lidar_type`: `1` (TOF型)
*   `fixed_size`: `3000` (防止点数溢出警告)

**编译与运行：**
```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ydlidar_ros2_driver
source install/setup.bash
ros2 launch ydlidar_ros2_driver ydlidar_launch.py
```

---

## 5. 深度相机配置 (Orbbec Astra Pro)

### 5.1 安装依赖
```bash
sudo apt update
sudo apt install ros-humble-camera-info-manager libuvc-dev libgoogle-glog-dev nlohmann-json3-dev libgflags-dev libopenni2-dev -y
```

### 5.2 安装驱动
```bash
cd ~/ros2_ws/src
git clone https://github.com/orbbec/ros2_astra_camera.git

# 安装 udev 规则
cd ros2_astra_camera/astra_camera/scripts
sudo bash install.sh
```

### 5.3 编译与运行
```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-up-to astra_camera
source install/setup.bash
# 注意使用 XML 格式的 launch 文件
ros2 launch astra_camera astra_pro.launch.xml
```

---

## 6. 地面站 (Windows WSL2) 联机配置
为了在笔记本上预览画面（Rviz2），需打通网络。

1. **WSL2 网络模式：**
   在 `C:\Users\用户名\.wslconfig` 添加（仅限 Win11 或 Win10 预览版，旧版需改用 **Discovery Server** 或 **桥接模式**）：
   ```ini
   [wsl2]
   networkingMode=mirrored
   ```

2. **环境对齐：**
   WSL2 必须安装 **Ubuntu 22.04 + ROS 2 Humble**。
   ```bash
   echo 'export ROS_DOMAIN_ID=30' >> ~/.bashrc
   ```

3. **数据验证：**
   小车开启驱动后，WSL2 终端输入：
   ```bash
   ros2 topic list
   ros2 topic hz /scan
   ros2 topic hz /camera/color/image_raw
   ```

---

## 7. 常用排坑命令
*   **查看 USB 详细 ID：** `lsusb`
*   **实时查看内核日志：** `sudo dmesg -w`
*   **自动补齐 ROS 依赖：** `rosdepc install --from-paths src --ignore-src -r -y`
*   **最大性能模式：** `sudo nvpmodel -m 0 && sudo jetson_clocks`
