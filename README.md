## 1. Môi trường và thư viện

- Ubuntu 22.04
- ROS 2 Humble Desktop
- Gazebo Ignition (Fortress)
- TurtleBot 4 Simulator
- SLAM Toolbox
- Navigation2 (Nav2)
- RViz2

## 2. Cài đặt phụ thuộc

Cập nhật hệ thống và cài đặt các package cần thiết:

```bash
sudo apt update

sudo apt install -y \
  ros-humble-turtlebot4-simulator \
  ros-humble-turtlebot4-navigation \
  ros-humble-slam-toolbox \
  ros-humble-navigation2 \
  ros-humble-nav2-bringup \
  ros-humble-teleop-twist-keyboard \
  python3-colcon-common-extensions
```

## 3. Build workspace

Đặt repository tại thư mục `~/Slam_toolbox-main`, sau đó chạy:

```bash
source /opt/ros/humble/setup.bash
cd ~/Slam_toolbox-main
colcon build --packages-select tb4_sim --symlink-install
source install/setup.bash
```

Nếu mở Terminal mới, nạp lại môi trường ROS 2 và workspace:

```bash
export ROS_DOMAIN_ID=24
source /opt/ros/humble/setup.bash
source ~/Slam_toolbox-main/install/setup.bash
```

Thực hiện bước trên trong mỗi Terminal mới trước khi chạy các lệnh ROS 2.

## 4. Hướng dẫn chạy chương trình

### 4.1. Simulation cơ bản

Khởi chạy TurtleBot 4 trong môi trường mê cung:

```bash
ros2 launch tb4_sim sim.launch.py world:=maze model:=standard rviz:=true
```

Kiểm tra các topic cơ bản trong một Terminal khác:

```bash
ros2 topic list
ros2 topic info /scan
ros2 topic info /odom
ros2 topic info /cmd_vel
```

Các topic chính:
- `/scan`: Dữ liệu LiDAR.
- `/odom`: Thông tin odometry của robot.
- `/cmd_vel`: Lệnh vận tốc điều khiển robot.

### 4.2. Xây dựng bản đồ bằng SLAM Toolbox

**Terminal 1:** Khởi chạy mô phỏng và SLAM Toolbox:

```bash
export ROS_DOMAIN_ID=24
source /opt/ros/humble/setup.bash
source ~/Slam_toolbox-main/install/setup.bash

ros2 launch tb4_sim mapping.launch.py world:=maze model:=standard
```

**Terminal 2:** Nạp môi trường và giải phóng robot khỏi dock:

```bash
export ROS_DOMAIN_ID=24
source /opt/ros/humble/setup.bash
source ~/Slam_toolbox-main/install/setup.bash

ros2 action send_goal /undock irobot_create_msgs/action/Undock "{}"
```

Mở khóa điều khiển và chạy teleoperation:

```bash
ros2 param set /motion_control safety_override full
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

Sử dụng bàn phím để điều khiển robot khảo sát môi trường. Nên di chuyển chậm, đi qua các hành lang và khu vực có thể tiếp cận để hạn chế sai số bản đồ.

**Lưu bản đồ:** Sau khi hoàn thành mapping, mở Terminal mới và chạy:

```bash
export ROS_DOMAIN_ID=24
source /opt/ros/humble/setup.bash
source ~/Slam_toolbox-main/install/setup.bash

ros2 run nav2_map_server map_saver_cli \
  -f ~/Slam_toolbox-main/tb4_sim/maps/map
```

Bản đồ được lưu thành hai tệp:

- `tb4_sim/maps/map.yaml`: Thông tin cấu hình bản đồ.
- `tb4_sim/maps/map.pgm`: Dữ liệu ảnh bản đồ lưới chiếm dụng.

### 4.3. Localization bằng AMCL

Dừng phiên mapping trước khi khởi chạy localization.

```bash
export ROS_DOMAIN_ID=24
source /opt/ros/humble/setup.bash
source ~/Slam_toolbox-main/install/setup.bash

ros2 launch tb4_sim localization.launch.py \
  world:=maze \
  model:=standard \
  map:=$HOME/Slam_toolbox-main/tb4_sim/maps/map.yaml
```

Quan sát trên RViz2 để kiểm tra vị trí và hướng của robot so với bản đồ đã lưu. Có thể sử dụng công cụ **2D Pose Estimate** để thiết lập vị trí ban đầu nếu cần.

### 4.4. Navigation với Nav2

Khởi chạy hệ thống navigation bằng bản đồ đã tạo:

```bash
export ROS_DOMAIN_ID=24
source /opt/ros/humble/setup.bash
source ~/Slam_toolbox-main/install/setup.bash

ros2 launch tb4_sim navigation.launch.py \
  world:=maze \
  model:=standard \
  map:=$HOME/Slam_toolbox-main/tb4_sim/maps/map.yaml
```

Nếu cần mở khóa điều khiển an toàn, chạy trong Terminal khác:

```bash
export ROS_DOMAIN_ID=24
source /opt/ros/humble/setup.bash
source ~/Slam_toolbox-main/install/setup.bash

ros2 param set /motion_control safety_override full
```

Trên RViz2:

1. Kiểm tra robot đã được định vị trên bản đồ.
2. Chọn công cụ **Nav2 Goal**.
3. Nhấn vào vị trí đích và kéo để thiết lập hướng mong muốn.
4. Quan sát robot tự lập kế hoạch và di chuyển đến đích.

Thực hiện với ít nhất ba đích đến khác nhau để kiểm tra khả năng điều hướng.

## 5. Một số lỗi thường gặp

**Không tìm thấy package `tb4_sim`:**

```bash
source /opt/ros/humble/setup.bash
cd ~/Slam_toolbox-main
colcon build --packages-select tb4_sim --symlink-install
source install/setup.bash
ros2 pkg list | grep tb4_sim
```

**Không nhận được dữ liệu LiDAR hoặc odometry:**

```bash
ros2 topic echo /scan
ros2 topic echo /odom
```

Kiểm tra Gazebo đã chạy, robot đã được spawn và các node liên quan đang hoạt động.

**Robot không di chuyển:**

Kiểm tra topic `/cmd_vel`, trạng thái dock và cơ chế an toàn của robot. Trong quá trình mapping hoặc navigation, chỉ gửi lệnh điều khiển sau khi đã xác nhận hệ thống sẵn sàng.

**Không tìm thấy bản đồ:**

```bash
ls -lh ~/Slam_toolbox-main/tb4_sim/maps/
```

Kiểm tra cả `map.yaml` và `map.pgm` đã tồn tại và đường dẫn bản đồ truyền vào launch file là chính xác.
