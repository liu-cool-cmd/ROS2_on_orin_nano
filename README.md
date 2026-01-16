**本机配置：亚博x3麦轮拓展板，astra pro深度相机，4ROS TOF雷达，orin nano 8GB super开发板**  
**系统：Ubuntu22.04  ROS2（Humble）**  
大量ai生成的因为我不会  

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

#### 3. 安装rosdepc工具

```sudo pip3 install rosdepc
sudo rosdepc init
rosdepc update
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

### 4. 激光雷达配置 (YDLIDAR TG15)

#### 4.1 安装底层 SDK
```bash
mkdir -p ~/ros2_ws/src && cd ~/ros2_ws/src
git clone https://github.com/YDLIDAR/YDLidar-SDK.git
cd YDLidar-SDK && mkdir build && cd build
cmake .. && make && sudo make install
```

#### 4.2 下载 ROS 2 驱动 (使用 Humble 专属分支)
不要直接下载默认分支，一定要指定 `-b humble`，这个分支已经由官方修复了参数声明报错问题。
```bash
cd ~/ros2_ws/src
# 关键：使用 -b humble 参数
git clone -b humble https://github.com/YDLIDAR/ydlidar_ros2_driver.git
```

#### 4.3 修改 TG15 适配参数
修改 `~/ros2_ws/src/ydlidar_ros2_driver/params/ydlidar.yaml` 以适配你的 TG15 雷达：
*   `port`: `/dev/ydlidar`
*   `baudrate`: `512000`
*   `lidar_type`: `1` (1 代表 TOF 型雷达)
*   `sample_rate`: `20`

#### 4.4 编译与运行
>先安装colcon编译器
```
sudo apt update
sudo apt install python3-colcon-common-extensions -y
```
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
**不推荐使用wsl,因为网络连通有一些我无法解决的问题，后文使用ubuntu笔记本当地面站**

<del>
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
</del>

---

##  常用排坑命令
*   **查看 USB 详细 ID：** `lsusb`
*   **实时查看内核日志：** `sudo dmesg -w`
*   **自动补齐 ROS 依赖：** `rosdepc install --from-paths src --ignore-src -r -y`
*   **最大性能模式：** `sudo nvpmodel -m 0 && sudo jetson_clocks`

---

## 7. 雅博 X3 底盘与扩展板驱动配置 (Yahboom Chassis)

本节记录如何驱动雅博专用扩展板，实现底盘控制、IMU反馈及电压监控。

### 7.1 底层硬件授权 (udev)
雅博驱动源码中多处硬编码引用了 `/dev/myserial`，为兼容源码且保留规范命名，建议设置双别名。

**修改规则文件**：`sudo nano /etc/udev/rules.d/99-yahboom.rules`
**写入以下内容**：
```bash
# 雅博底盘扩展板 (CH340芯片)
KERNEL=="tty*", ATTRS{idVendor}=="1a86", ATTRS{idProduct}=="7523", MODE:="0777", SYMLINK+="yahboomcar myserial"
```
*注：SYMLINK 中增加 myserial 是为了适配雅博 Mcnamu_driver_X3.py 中的默认路径。*

**刷新规则**：
```bash
sudo udevadm control --reload-rules && sudo udevadm trigger
```

### 7.2 安装底层 Python SDK (Rosmaster_Lib)
底盘节点依赖雅博封装的串口协议库，必须先于 ROS 节点安装。

```bash
# 进入从 software 目录拷贝出的 Rosmaster_Lib 文件夹
cd ~/Rosmaster_Lib
sudo python3 setup.py install
```

### 7.3 补齐系统级依赖
雅博 X3 的启动脚本包含模型解析、IMU 滤波及 EKF 融合，需手动补齐以下 Humble 标准包：

```bash
sudo apt update
sudo apt install ros-humble-joint-state-publisher \
                 ros-humble-xacro \
                 ros-humble-robot-state-publisher \
                 ros-humble-imu-filter-madgwick \
                 ros-humble-teleop-twist-keyboard -y
```

### 7.4 源码合并与编译
将雅博提供的 `library_ws/src` (消息包) 和 `yahboomcar_ws/src` (驱动包) 全部拷贝至自己的工作空间。

```bash
cd ~/ros2_ws
# 自动安装 package.xml 中声明的其他依赖
rosdepc install --from-paths src --ignore-src -r -y
# 编译
colcon build --symlink-install
source install/setup.bash
```

### 7.5 运行与控制

1. **设置车型变量**（必须，否则无法加载对应的 X3 模型）：
   ```bash
   echo 'export ROBOT_TYPE=X3' >> ~/.bashrc
   source ~/.bashrc
   ```

2. **启动底盘驱动**：
   ```bash
   ros2 launch yahboomcar_bringup yahboomcar_bringup_X3_launch.py
   ```

3. **键盘控制测试**：
   ```bash
   # 开启新终端
   ros2 run yahboomcar_ctrl yahboom_keyboard
   ```
   *控制键：i(前)、k(停)、j(左转)、l(右转)、u(左横移)、o(右横移)。*

### 7.6 常用监控话题
- **电压监控**：`ros2 topic echo /voltage` (建议不低于 11.0V)
- **速度反馈**：`ros2 topic echo /vel_raw`
- **IMU 姿态**：`ros2 topic echo /imu/imu_data`

---

### 💡 调试秘籍
- **Source 机制**：每当修改了驱动源码或添加了新包，必须在 `colcon build` 后重新 `source install/setup.bash`，否则新包无法被 `ros2 launch` 识别。
- **串口占用**：如果报 `SerialException: [Errno 16] Device or resource busy`，通常是因为之前的驱动进程没杀掉，执行 `pkill -9 -f Mcnamu_driver` 即可。

**总结启动指令：
底盘：ros2 launch yahboomcar_bringup yahboomcar_bringup_X3_launch.py
雷达：ros2 launch ydlidar_ros2_driver ydlidar_launch.py
相机：ros2 launch astra_camera astra_pro.launch.xml
新一键启动：ros2 launch yahboomcar_bringup bringup_all.launch.py**


<del> 
    
## 8. Linux地面站配置

### 1. 身份与网络对齐 (在笔记本终端执行)

首先确保笔记本和小车连在同一个 WiFi 下。

**设置 Domain ID (必须和小车一致):**
```bash
# 将 ID=30 写入配置文件
echo 'export ROS_DOMAIN_ID=30' >> ~/.bashrc

# 立即生效
source ~/.bashrc
```

---

### 2. 搬运并编译自定义消息包 (最关键)

你需要把小车上的 `yahboomcar_msgs` 文件夹弄到笔记本上。这相当于给笔记本装了一本“字典”，否则 Rviz2 收到数据也会报错说“看不懂”。

#### 第一步：创建工作空间 (如果还没建)
在笔记本终端执行：
```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src
```

#### 第二步：获取源码 (两种方法选一)

*   **方法 A (最快 - 远程拷贝)：** 如果你知道小车的 IP (假设是 192.168.10.3) 和用户名 (假设是 liu)，直接在**笔记本终端**运行：
    ```bash
    # 格式: scp -r <小车用户名>@<小车IP>:~/ros2_ws/src/yahboomcar_msgs .
    scp -r liu@192.168.10.3:~/ros2_ws/src/yahboomcar_msgs .
    ```
    *(输入小车密码后，文件夹就会自动传过来)*

*   **方法 B (U盘/Git)：** 如果你之前备份过，直接把 `yahboomcar_msgs` 文件夹拷贝到笔记本的 `~/ros2_ws/src/` 目录下即可。

#### 第三步：编译 (让笔记本认识这些消息)
>第0步安装好要用的工具
```
sudo apt update
sudo apt install python3-colcon-common-extensions -y
sudo pip3 install rosdepc
sudo rosdepc init
rosdepc update
```

```bash
cd ~/ros2_ws

# 1. 自动安装编译依赖 (防止报错)
# 如果你还没装 rosdepc，先运行: sudo pip3 install rosdepc && sudo rosdepc init && rosdepc update
rosdepc install --from-paths src --ignore-src -r -y

# 2. 编译
colcon build --symlink-install

# 3. 刷新环境 (至关重要)
source install/setup.bash
```

#### 第四步：把环境加载写入 .bashrc
为了防止以后每次打开终端都要手动 source，直接写死它：
```bash
echo 'source ~/ros2_ws/install/setup.bash' >> ~/.bashrc
source ~/.bashrc
```

---

### 3. 终极验证 (见证合体)

现在，笔记本和小车应该彻底通了。

1.  **在小车上**：启动雷达、相机或底盘驱动 (比如 `ros2 launch yahboomcar_bringup yahboomcar_bringup_X3_launch.py`)。
2.  **在笔记本上**：
    *   输入 `ros2 topic list` —— **应该能看到 `/scan`, `/voltage` 等话题。**
    *   输入 `ros2 topic echo /voltage` —— **应该能看到跳动的电压数值。**
    *   输入 `rviz2` —— **打开 Rviz2，添加 LaserScan，Fixed Frame 选 `laser_frame`，红点应该瞬间出现！**
</del>
---

## 8. 进阶：一键总启动 (Total Launch)
为了避免每次打开 3 个终端，创建一个总启动文件来管理所有硬件。

**文件路径**：`~/ros2_ws/src/yahboomcar_bringup/launch/bringup_all.launch.py`
**代码内容**：
```python
import os
from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource, FrontendLaunchDescriptionSource

def generate_launch_description():
    # 1. 定位功能包
    chassis_pkg = get_package_share_directory('yahboomcar_bringup')
    lidar_pkg   = get_package_share_directory('ydlidar_ros2_driver')
    camera_pkg  = get_package_share_directory('astra_camera')

    # 2. 包含子启动文件
    # 底盘 (Python)
    chassis_launch = IncludeLaunchDescription(
        PythonLaunchDescriptionSource(os.path.join(chassis_pkg, 'launch', 'yahboomcar_bringup_X3_launch.py'))
    )
    # 雷达 (Python)
    lidar_launch = IncludeLaunchDescription(
        PythonLaunchDescriptionSource(os.path.join(lidar_pkg, 'launch', 'ydlidar_launch.py'))
    )
    # 相机 (XML) - 使用 FrontendLaunchDescriptionSource
    camera_launch = IncludeLaunchDescription(
        FrontendLaunchDescriptionSource(os.path.join(camera_pkg, 'launch', 'astra_pro.launch.xml'))
    )

    return LaunchDescription([chassis_launch, lidar_launch, camera_launch])
```
**使用方法**：
```bash
ros2 launch yahboomcar_bringup bringup_all.launch.py
```

---

## 9. 地面站搭建 (Linux 笔记本)
**环境要求**：Ubuntu 22.04 + ROS 2 Humble (与小车一致)。

### 9.1 基础配置
1. **安装 ROS 2**：使用鱼香 ROS 一键脚本安装 Humble Desktop 版。
2. **安装编译工具**：`sudo apt install python3-colcon-common-extensions git`
3. **网络对齐**：
   * 连接同一 WiFi。
   * 修改 `~/.bashrc`：`export ROS_DOMAIN_ID=30`。

### 9.2 依赖包同步 (关键)
为了让 Rviz2 正确显示小车状态，必须将小车的 **消息定义** 和 **模型文件** 同步到笔记本。

1. **拷贝源码**：
   将小车上的 `yahboomcar_msgs` 和 `yahboomcar_description` 文件夹拷贝到笔记本的 `~/ros2_ws/src/`。
2. **编译**：
>第0步安装好要用的工具
```
sudo apt update
sudo apt install python3-colcon-common-extensions -y
sudo pip3 install rosdepc
sudo rosdepc init
rosdepc update
```
   ```bash
   cd ~/ros2_ws
   rosdepc install --from-paths src --ignore-src -r -y
   colcon build --symlink-install
   source install/setup.bash
   ```
### 9.3. 验证

现在，笔记本和小车应该彻底通了。

1.  **在小车上**：启动雷达、相机或底盘驱动 (比如 `ros2 launch yahboomcar_bringup yahboomcar_bringup_X3_launch.py`)。
2.  **在笔记本上**：
    *   输入 `ros2 topic list` —— **应该能看到 `/scan`, `/voltage` 等话题。**
    *   输入 `ros2 topic echo /voltage` —— **应该能看到跳动的电压数值。**
    *   输入 `rviz2` —— **打开 Rviz2，添加 LaserScan，Fixed Frame 选 `laser_frame`，红点应该瞬间出现！**
---

## 10. Rviz2 可视化配置
**启动**：在笔记本终端输入 `rviz2`。

**关键配置**：
1. **Fixed Frame**：设为 `base_link`。
2. **RobotModel**：添加后，Description Topic 选 `/robot_description`。
3. **LaserScan**：
   * Topic: `/scan`
   * **QoS Reliability**: 改为 `Best Effort` (解决 0 points 问题)。
   * Size: 0.05
4. **Image**：Topic 选 `/camera/color/image_raw`。

**保存配置**：
File -> Save Config As -> `~/ros2_ws/src/yahboomcar_bringup/rviz/my_robot.rviz`。

---

**把这段加进去，你的文档就完美了！** 
准备好把车放到地板上，开始你的第一次“建图”之旅了吗？
