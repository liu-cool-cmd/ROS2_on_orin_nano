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

#### 2. 编译 YDLIDAR 4ROS 驱动 (ROS 2)
不要用卖家给的树莓派包，直接去官方拉最新的 ROS 2 驱动。

```bash
# 1. 创建工作空间
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src

# 2. 拉取 YDLIDAR SDK (驱动底层)
git clone https://github.com/YDLIDAR/YDLidar-SDK.git
cd YDLidar-SDK
mkdir build && cd build
cmake ..
make
sudo make install
cd ../..

# 3. 拉取 ROS 2 驱动包
git clone https://github.com/YDLIDAR/ydlidar_ros2_driver.git
# 切回工作空间根目录
cd ~/ros2_ws

# 4. 安装依赖
rosdepc install --from-paths src --ignore-src -r -y #如果找不到rosdepc请运行sudo pip3 install rosdepc安装，并运行sudo rosdepc init&&rosdepc update来启动

# 5. 编译
colcon build --symlink-install --packages-select ydlidar_ros2_driver
```

**测试雷达：**
修改 `launch` 文件中的端口。
```bash
# 假设雷达是 /dev/ttyUSB0
sudo chmod 777 /dev/ttyUSB0
source install/setup.bash
ros2 launch ydlidar_ros2_driver ydlidar_launch.py
```
*如果转起来了且终端有数据刷屏，雷达 pass。*

#### 3. 编译奥比中光 Astra Pro 驱动
Astra Pro 是老款，OpenNI2 协议。新的 Orbbec SDK v2 可能不完全支持它，我们要用 `ros2_astra_camera` (基于 OpenNI2)。

**依赖坑预警：** Ubuntu 22.04 需要 `libgflags-dev` 和 `libgoogle-glog-dev`。

```bash
sudo apt install libopenni2-dev libgflags-dev libgoogle-glog-dev -y

cd ~/ros2_ws/src
# 克隆这一版，专门针对 ROS 2 Humble 和旧款 Astra
git clone https://github.com/orbbec/ros2_astra_camera.git

cd ~/ros2_ws
# 这一步可能会报错，如果报错，通常是 libuvc 问题，先试着编译
sudo apt install python3-colcon-common-extensions -y #安装colcon用于编译
colcon build --packages-select astra_camera
```

**测试相机：**
```bash
source install/setup.bash
ros2 launch astra_camera astra_mini.launch.py 
# 注意：Pro版可能需要修改 launch 文件中的 device_type 或 vendor_id
```

---
