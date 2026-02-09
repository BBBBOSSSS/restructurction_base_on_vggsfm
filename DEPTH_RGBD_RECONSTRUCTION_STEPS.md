# 基于深度相机（RGB-D）的三维重建步骤梳理（本仓库版本）

本文档按 `reconstruction_toolkit/` 里的脚本，梳理从深度相机数据到点云、桌面分割、物体定位，并转换到 AprilTag 坐标系的完整流程。

> 你当前已有一份示例数据：`reconstruction_toolkit/output/20260205_163837/`（已包含 `rgb/ depth/ info/metadata.json` 以及 `rgbd_reconstruction/` 输出）。

---

## 依赖（最小集合）

常用依赖（按脚本导入项）：

```bash
pip install -U numpy opencv-python open3d scipy rosbags
pip install -U dt-apriltags  # 或：pip install -U pupil-apriltags
```

## 0. 输入与输出约定

### 0.1 输入目录结构（RGB-D 数据目录）

后续脚本都假定 `DATA_DIR` 形如：

```
DATA_DIR/
├── rgb/                # RGB帧（000000.png, 000001.png, ...）
├── depth/              # 深度帧（000000.png, 000001.png, ...）
└── info/metadata.json  # 相机内参、分辨率、时间戳等
```

### 0.2 RGB-D 重建输出目录

`rgbd_reconstruction.py` 会在 `DATA_DIR/rgbd_reconstruction/` 产出：

```
rgbd_reconstruction/
├── combined_pointcloud.ply
├── table_plane.ply
├── objects_above_table.ply
├── all_objects_colored.ply
├── object_00.ply, object_01.ply, ...
├── objects_info.json
└── reconstruction_report.md
```

### 0.3 AprilTag 坐标系输出目录

`apriltag_coordinate_system.py` 会在 `DATA_DIR/rgbd_reconstruction/apriltag_frame/` 产出：

```
apriltag_frame/
├── apriltag_detection.jpg
├── apriltag_info.json
├── objects_info_tag.json
├── combined_pointcloud_tag.ply
├── table_plane_tag.ply
└── object_00_tag.ply, object_01_tag.ply, ...
```

---

## 1. （可选）从 RealSense ROS bag 提取 RGB-D

如果你的原始数据是 `.bag`，先用 `extract_rgbd_from_bag.py` 提取成上面的 `DATA_DIR/` 结构：

```bash
python3 reconstruction_toolkit/extract_rgbd_from_bag.py \
  /path/to/your.bag \
  reconstruction_toolkit/output/<scene_name>
```

要点：
- 该脚本使用 `rosbags` 读取 ROS1 bag，并假设 RealSense topic 名称为：
  - RGB：`/device_0/sensor_1/Color_0/image/data`
  - Depth：`/device_0/sensor_0/Depth_0/image/data`
  - RGB CameraInfo：`/device_0/sensor_1/Color_0/info/camera_info`
  - Depth CameraInfo：`/device_0/sensor_0/Depth_0/info/camera_info`
- 输出图像命名为 `000000.png` 这种 6 位序号；相机内参与帧时间戳写入 `info/metadata.json`。

---

## 2. 生成点云并做桌面/物体分割（核心：深度重建）

对某个 `DATA_DIR` 运行完整流程：

```bash
python3 reconstruction_toolkit/rgbd_reconstruction.py \
  reconstruction_toolkit/output/20260205_163837 \
  --mode full \
  --max-frames 50 \
  --stride 10 \
  --no-viz
```

脚本内部做的事（对应 `RGBDReconstructor.full_pipeline()`）：
1. **多帧融合点云**：从多帧 RGB+Depth 用 Open3D 反投影生成点云后直接累加融合（默认认为相机基本静止/帧已对齐）。
2. **桌面平面检测**：RANSAC `segment_plane()` 分离桌面点云与“桌面上方”的点云。
3. **物体分割**：对“桌面上方”点云做 DBSCAN 聚类，输出每个物体点云 `object_XX.ply` 与 `objects_info.json`（中心点、AABB、尺寸等）。
4. **生成报告**：写入 `rgbd_reconstruction/reconstruction_report.md`。

关键注意事项：
- 深度单位：`load_rgbd_frame()` 默认 `depth_scale=1000.0`（常见 RealSense 深度 PNG 为毫米；即 1000 表示 1000mm=1m）。
- 尺寸对齐：如果深度分辨率与 RGB 不同，会把深度 resize 到 RGB（最近邻）。
- 相机运动：当前实现**不使用相机位姿**，多帧直接叠加；如果相机移动，会出现“重影/模糊”的点云。

---

## 3. 建立 AprilTag 世界坐标系（把物体坐标转到 Tag 坐标系）

在完成第 2 步后（确保存在 `rgbd_reconstruction/objects_info.json` 与 `combined_pointcloud.ply`），运行：

```bash
python3 reconstruction_toolkit/apriltag_coordinate_system.py \
  --data-dir reconstruction_toolkit/output/20260205_163837 \
  --tag-size 0.161
```

说明：
- `--tag-size` 单位是**米**（例如 5cm => `0.05`；16.1cm => `0.161`）。
- 脚本会在前若干帧 RGB 里搜索 AprilTag（默认 family：`tag16h5`），挑 decision margin 最大的一帧作为坐标系参考。
- 坐标变换使用约定：`p_cam = R * p_tag + t`，因此 `p_tag = R^T * (p_cam - t)`。

输出位置：
- `DATA_DIR/rgbd_reconstruction/apriltag_frame/`（见本文档 0.3）

---

## 4. 可视化与核对

### 4.1 查看 RGB-D 重建结果（相机坐标系）

建议先打开报告：
- `reconstruction_toolkit/output/20260205_163837/rgbd_reconstruction/reconstruction_report.md`

点云文件可用 Open3D/CloudCompare/MeshLab 打开，例如：
- `combined_pointcloud.ply`
- `table_plane.ply`
- `all_objects_colored.ply`

### 4.2 查看 AprilTag 坐标系结果（Tag 为原点）

```bash
python3 reconstruction_toolkit/view_apriltag_frame.py
```

（该脚本当前默认读取：`/root/reconstruction_toolkit/output/20260205_163837/rgbd_reconstruction`）

---

## 5. 常见问题（最影响精度的 3 件事）

1. **Tag 尺寸必须正确**：`--tag-size` 填错会导致坐标系比例整体错误（单位仍是米，但尺度不对）。
2. **Tag family 要匹配**：`apriltag_coordinate_system.py` 默认 `tag16h5`；如果你打印的是 `tag36h11` 需要改脚本里的 `families=...`。
3. **相机是否移动**：`rgbd_reconstruction.py` 的多帧融合没有配准/里程计；移动相机会显著降低效果。
