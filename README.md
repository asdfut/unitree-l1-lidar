**中文** | [English](README.en.md)

# Unitree 4D LiDAR L1 啟動與建圖紀錄（2026-10-27）

環境：Ubuntu 24.04、ROS 2 Jazzy；SDK 位置 `~/unilidar_sdk`

結果：
- 已在 rviz2 中成功看到即時點雲（`/unilidar/cloud`）
- 已將 [Point-LIO](https://github.com/unitreerobotics/point_lio_unilidar) 移植到 ROS 2 Jazzy，並用 L1 完成一次室內建圖（約 156 萬點）

![rviz2 即時點雲](images/rviz2_pointcloud.png)

![Point-LIO 建圖結果](images/pointlio_map.png)

## 檔案說明
- `README.md`：設定步驟與問題排解（本文件，中文）
- `README.en.md`：英文版
- `conversation.md`：對話紀錄 1，包括 SDK、rviz2 與 Point-LIO 移植
- `conversation_2.md`：對話紀錄 2，包括 USB 連線排錯與 Point-LIO 建圖測試
- `config/view.rviz`：rviz2 設定檔，取自 [unitreerobotics/unilidar_sdk](https://github.com/unitreerobotics/unilidar_sdk)
- `point_lio_ros2/point_lio_ros2_jazzy.patch`：Point-LIO 的 ROS 2 Jazzy 移植 patch
- `images/`：截圖與地圖預覽

## 下載 SDK
官方 SDK：<https://github.com/unitreerobotics/unilidar_sdk>（含 C++ SDK、ROS 1、ROS 2 套件）
```bash
cd ~
git clone https://github.com/unitreerobotics/unilidar_sdk.git
```
其他資源（說明書、Windows 工具等）：<https://www.unitree.com/download>

## 硬體確認
- L1 需另接 12V 電源，USB 只傳資料
- 接上後 `lsusb` 應出現 `10c4:ea60 Silicon Labs CP210x UART Bridge`（實際晶片 CP2104）
- 裝置節點：`/dev/ttyUSB0`
- 插拔時可用 `sudo dmesg -w` 即時觀察，正常會看到 `cp210x converter now attached to ttyUSB0`

## 串口權限
```bash
sudo chmod 666 /dev/ttyUSB0          # 暫時，拔插後失效
sudo usermod -aG dialout $USER       # 永久，需登出再登入
```

## SDK 測試程式
```bash
cd ~/unilidar_sdk/unitree_lidar_sdk
mkdir -p build && cd build && cmake .. && make -j4
../bin/example_lidar                 # 預設讀 /dev/ttyUSB0，Ctrl+C 結束
```

## ROS 2 + rviz2
終端機 1：
```bash
cd ~/unilidar_sdk/unitree_lidar_ros2
source /opt/ros/jazzy/setup.bash
colcon build                         # 只需第一次
source install/setup.bash
ros2 launch unitree_lidar_ros2 launch.py
```
終端機 2：
```bash
source /opt/ros/jazzy/setup.bash
rviz2 -d ~/unilidar_sdk/unitree_lidar_ros2/src/unitree_lidar_ros2/rviz/view.rviz
# 或使用本 repo 的 config/view.rviz
```
- 點雲 topic：`/unilidar/cloud`（frame `unilidar_lidar`），實測約 9.3 Hz
- IMU topic：`/unilidar/imu`（frame `unilidar_imu`），實測約 234 Hz
- 檢查資料：`ros2 topic hz /unilidar/cloud`
- rviz2 的 Fixed Frame 設為 `unilidar_lidar`；Decay Time 調到 3～5 秒點雲會更密

## Point-LIO 建圖（ROS 2 Jazzy）
Unitree 官方的 [point_lio_unilidar](https://github.com/unitreerobotics/point_lio_unilidar) 只支援 ROS 1（catkin、Noetic），沒辦法直接在 Ubuntu 24.04 + Jazzy 上編譯。本 repo 提供移植 patch，對應上游 commit `18ed597`（v2.0.2）。

主要修改：
- `catkin` 改成 `ament_cmake`，C++17，預設 Release 編譯
- `roscpp` API 改為 `rclcpp`（`laserMapping.cpp`、`parameters.cpp`、`preprocess.*`、`IMU_Processing.hpp`）
- `config/unilidar_l1.yaml` 改成 `ros__parameters` 格式，並把原本寫在 ROS 1 launch 檔裡的參數合併進來（浮點數要保留小數點，ROS 2 不會自動轉型）
- 新增 `launch/mapping_unilidar_l1.launch.py` 與 rviz2 設定檔 `rviz_cfg/unilidar_l1.rviz`

### 編譯
```bash
mkdir -p ~/point_lio_ws/src && cd ~/point_lio_ws/src
git clone https://github.com/unitreerobotics/point_lio_unilidar.git
cd point_lio_unilidar
git checkout 18ed597
git apply /path/to/unitree-l1-lidar/point_lio_ros2/point_lio_ros2_jazzy.patch
cd ~/point_lio_ws
source /opt/ros/jazzy/setup.bash
colcon build
```

### 執行
終端機 1：啟動 LiDAR 驅動（見上一節）

終端機 2：
```bash
cd ~/point_lio_ws
source /opt/ros/jazzy/setup.bash && source install/setup.bash
ros2 launch point_lio_unilidar mapping_unilidar_l1.launch.py   # 加 rviz:=false 可不開 rviz2
```
- 啟動後的前幾秒要讓 LiDAR **保持靜止**，讓 IMU 完成初始化（日誌出現 `IMU Initializing: 100.0 %`）
- 輸出 topic：`/pointlio/cloud_registered`、`/pointlio/odom`（實測約 9 Hz），以及 `/pointlio/laser_map`、`/pointlio/path`、`/tf`
- 按 `Ctrl+C` 結束時會自動存檔：`~/point_lio_ws/src/point_lio_unilidar/PCD/scans.pcd`
- **下次執行會覆蓋 `scans.pcd`**，要保留的話請先複製出來
- 查看地圖：`pcl_viewer scans.pcd`（需先 `sudo apt install pcl-tools`）

本次測試的地圖有 1,559,419 點，檔案約 50 MB，所以沒有放進 repo，只放了預覽圖 `images/pointlio_map.png`。

## 遇到的問題
**`SerialException device reports readiness to read but returned no data (device disconnected?)`**
- 原因：`example_lidar` 還在背景執行，佔住了 `/dev/ttyUSB0`
- 解法：關掉它（`Ctrl+C` 或 `kill <PID>`）後再啟動 ROS 節點
- 查誰佔用串口：`fuser -v /dev/ttyUSB0`
- 若仍有問題，可能是 ModemManager 或 brltty 干擾：
  `sudo systemctl stop ModemManager`、`sudo apt remove brltty`

**`Unilidar failed to initialize! Serial port "/dev/ttyUSB0" does not exist!`**
- 原因：系統根本沒偵測到 LiDAR 的 USB 轉串口板，`lsusb` 裡沒有 `10c4:ea60`
- 先看 `dmesg`：
  - `cp210x ttyUSB0: failed set request ... status: -19` 代表裝置在使用中被拔掉或斷電（-19 = ENODEV）
  - 插上時完全沒有任何新的 USB 紀錄：通常是 **USB 線只能充電、不能傳資料**，或線沒插在轉接板上，或經過了沒供電的 Hub
- 注意分辨裝置：`Logitech`、`046d` 是滑鼠接收器；LiDAR 是 `10c4`、`cp210x`、`CP2104`
- CP2104 靠 USB 供電，所以就算 LiDAR 沒接 12V，只要資料線正常，也一定會出現 `10c4:ea60`
- Ubuntu 24.04 的 brltty udev 規則裡，`10c4:ea60` 那一行預設是註解掉的，所以這次不是 brltty 搶走裝置
