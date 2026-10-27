# Unitree 4D LiDAR L1 啟動紀錄（2026-10-27）

環境：Ubuntu、ROS 2 Jazzy；SDK 位置 `~/unilidar_sdk`
結果：已在 rviz2 中成功看到即時點雲（`/unilidar/cloud`）

![rviz2 即時點雲](images/rviz2_pointcloud.png)

## 檔案說明
- `README.md`：設定步驟與問題排解（本文件）
- `conversation.md`：完整的除錯對話紀錄
- `config/view.rviz`：rviz2 設定檔，取自 [unitreerobotics/unilidar_sdk](https://github.com/unitreerobotics/unilidar_sdk)
- `images/`：截圖

## 硬體確認
- L1 需另接 12V 電源，USB 只傳資料
- 接上後 `lsusb` 應出現 `10c4:ea60 Silicon Labs CP210x UART Bridge`（實際晶片 CP2104）
- 裝置節點：`/dev/ttyUSB0`

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
- 點雲 topic：`/unilidar/cloud`（frame `unilidar_lidar`）
- IMU topic：`/unilidar/imu`（frame `unilidar_imu`）
- 檢查資料：`ros2 topic hz /unilidar/cloud`
- rviz2 的 Fixed Frame 設為 `unilidar_lidar`；Decay Time 調到 3～5 秒點雲會更密

## 遇到的問題
**`SerialException device reports readiness to read but returned no data (device disconnected?)`**
- 原因：`example_lidar` 還在背景執行，佔住了 `/dev/ttyUSB0`
- 解法：關掉它（`Ctrl+C` 或 `kill <PID>`）後再啟動 ROS 節點
- 查誰佔用串口：`fuser -v /dev/ttyUSB0`
- 若仍有問題，可能是 ModemManager 或 brltty 干擾：
  `sudo systemctl stop ModemManager`、`sudo apt remove brltty`
