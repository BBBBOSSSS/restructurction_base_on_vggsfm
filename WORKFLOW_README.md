# VGGSfM三维重建 + AprilTag标定工作流

完整的从ROS bag文件进行三维重建和物体坐标标定的工作流。

## 快速开始

### 1. 数据准备

将您的bag文件放在 `/root/vggsfm/data/` 目录：
```bash
cp your_videos/*.bag /root/vggsfm/data/
```

### 2. 运行完整工作流

#### 方案A: 自动化工作流 (推荐)
```bash
bash /root/run_reconstruction_pipeline.sh
```

#### 方案B: 手动逐步执行

**步骤1: 提取RGB-D数据**
```bash
python3 /root/extract_rgbd_from_bag.py /root/vggsfm/data/your_video.bag /tmp/extracted_data
```

**步骤2: 运行SfM重建**
```bash
cd /root/vggsfm
python3 demo.py SCENE_DIR=/tmp/extracted_data \
  query_frame_num=1 \
  use_poselib=false \
  fine_tracking=false \
  query_method=sp
```

**步骤3: 检测AprilTag**
```bash
python3 /root/apriltag_calibration.py \
  /tmp/extracted_data/info/metadata.json \
  /tmp/extracted_data/rgb \
  /tmp/extracted_data/depth
```

**步骤4: 交互式标注物体 (用于精确标定)**
```bash
python3 /root/object_annotator.py \
  /tmp/extracted_data/sparse \
  /tmp/extracted_data/info/metadata.json \
  /tmp/extracted_data/depth
```

## 详细说明

### 工作流概览

```
ROS bag 文件
    ↓
[1] 提取RGB-D (extract_rgbd_from_bag.py)
    ├─ RGB图像
    ├─ 深度图像
    └─ 相机内参
    ↓
[2] 三维重建 (VGGSfM demo.py)
    └─ COLMAP格式点云 (sparse/)
    ↓
[3] AprilTag检测 (apriltag_calibration.py)
    ├─ Tag位置检测
    └─ 姿态估计
    ↓
[4] 物体坐标标定 (object_annotator.py)
    ├─ 交互式标注
    └─ 三维坐标计算
    ↓
输出结果 (JSON格式)
```

### 输出文件说明

#### 1. `extracted_data/`
- `rgb/` - 输入的RGB图像序列
- `depth/` - 对应的深度图像
- `info/metadata.json` - 相机参数和帧同步信息

#### 2. `extracted_data/sparse/` (COLMAP格式)
- `cameras.bin` - 相机的内外参
- `images.bin` - 图像和特征点信息
- `points3D.bin` - 稀疏点云
- `points3D.txt` - 点云文本格式（易读）

#### 3. `apriltag_detections/`
- 检测结果的可视化图像

#### 4. `calibration.json` (标定结果)
```json
{
  "calibrations": {
    "frame_0": {
      "tag_id": 0,
      "objects": [
        {
          "object_id": 0,
          "image_pos": [100, 150],           // 图像坐标 (像素)
          "camera_pos": [0.05, 0.1, 0.5],    // 相机坐标 (米)
          "world_pos": [-0.02, 0.08, 0.45]   // 世界坐标 (相对于AprilTag)
        }
      ]
    }
  },
  "apriltags": {
    "0": [0, 0, 0]  // AprilTag在世界坐标系的位置
  }
}
```

#### 5. `object_annotations.json` (手动标注)
```json
{
  "frame_0": [
    {
      "pixel": [100, 150],
      "camera_coords": [0.05, 0.1, 0.5],
      "object_id": 0,
      "description": "Object_0"
    }
  ]
}
```

## 坐标系统

### 相机坐标系 (OpenCV标准)
```
        Z↑ (向前)
         |
    -----|-----→ X (向右)
        /
       /
      Y (向下)
```

### AprilTag坐标系
- 原点在标签中心
- X轴向右，Y轴向下
- Z轴垂直标签表面向外

### 世界坐标系
- 由第一个AprilTag定义
- AprilTag中心为原点 (0, 0, 0)

## 关键参数

### SfM重建参数
```python
img_size: 1024           # 图像处理分辨率
query_frame_num: 1       # 查询帧数（降低可减少内存）
fine_tracking: False     # 精细跟踪（可选，消耗内存）
use_poselib: False       # 使用poselib（可避免pyceres崩溃）
query_method: "sp"       # 特征检测器，建议用"sp"(SuperPoint)
```

### AprilTag参数
```python
tag_size: 0.05           # 标签物理尺寸（米）
families: 'tag36h11'     # 标签族（推荐36h11）
```

### 深度标定
如果没有深度信息，可以设置固定的假设深度：
```bash
python3 /root/object_annotator.py ... --z_fixed 0.5  # 假设所有物体在0.5m处
```

## 故障排除

### 问题1: CUDA内存溢出
**症状**: `torch.OutOfMemoryError`

**解决方案**:
```python
# 降低参数
query_frame_num: 1       # 改为1
robust_refine: 1         # 改为1
img_size: 512            # 降低分辨率
fine_tracking: False     # 禁用
```

### 问题2: 无法检测到AprilTag
**症状**: 输出中没有AprilTag检测

**解决方案**:
- 检查AprilTag是否在图像中
- 调整光照条件
- 确保AprilTag大小合适（建议物理尺寸0.05-0.1米）

### 问题3: 深度信息不精确
**症状**: 深度图模糊或有缺失

**解决方案**:
- 使用相机附带的深度滤波工具
- 增加拍摄距离
- 确保充足光照

### 问题4: 坐标标定误差大
**症状**: 物体标定坐标不准

**解决方案**:
- 验证相机内参是否正确
- 检查AprilTag的物理尺寸设置
- 使用多帧标注求平均
- 检查AprilTag是否有形变

## 可视化结果

### 1. 查看点云 (需要COLMAP)
```bash
# 安装COLMAP
conda install -c conda-forge colmap

# 查看结果
colmap gui --database_path=/path/to/sparse/database.db
```

### 2. 用Python可视化
```python
import json
import numpy as np
import matplotlib.pyplot as plt
from mpl_toolkits.mplot3d import Axes3D

# 加载标定结果
with open('calibration.json') as f:
    calib = json.load(f)

# 绘制物体位置
fig = plt.figure()
ax = fig.add_subplot(111, projection='3d')

for frame_data in calib['calibrations'].values():
    for obj in frame_data['objects']:
        pos = obj['world_pos']
        ax.scatter(*pos, c='r', s=100)

# 绘制AprilTag位置
for tag_id, pos in calib['apriltags'].items():
    ax.scatter(*pos, c='b', s=200, marker='^')

ax.set_xlabel('X')
ax.set_ylabel('Y')
ax.set_zlabel('Z')
plt.show()
```

## 高级用法

### 多帧标注求平均
```python
# 在多个帧上标注同一物体，然后求平均位置以提高精度

import json
import numpy as np

with open('object_annotations.json') as f:
    annot = json.load(f)

object_positions = {}
for frame_data in annot.values():
    for obj in frame_data:
        obj_id = obj['object_id']
        if obj_id not in object_positions:
            object_positions[obj_id] = []
        object_positions[obj_id].append(obj['camera_coords'])

# 计算平均位置
for obj_id, positions in object_positions.items():
    avg_pos = np.mean(positions, axis=0)
    print(f"Object {obj_id}: {avg_pos}")
```

### 从COLMAP获取更多信息
```python
import subprocess
import sqlite3

# 导出点云为PLY格式
subprocess.run([
    'colmap', 'point_triangulator',
    '--database_path', './sparse/database.db',
    '--image_path', './images',
    '--input_path', './sparse',
    '--output_path', './sparse',
    '--Mapper.ba_refine_principal_point', 'true'
])

# 读取SQLite数据库获取特征点
conn = sqlite3.connect('./sparse/database.db')
cursor = conn.cursor()

# 查询所有特征点
cursor.execute('SELECT * FROM keypoints')
keypoints = cursor.fetchall()
```

## 常见问题 (FAQ)

**Q: 支持多个AprilTag吗?**
A: 支持！系统会自动检测所有AprilTag，并根据每个tag创建局部坐标系。

**Q: 可以用其他的标记方式吗?**
A: 可以修改`apriltag_calibration.py`中的检测部分，支持圆形标记、网格等。

**Q: 必须有深度相机吗?**
A: 不必须，系统会使用SfM的稀疏点云提供深度估计。精度会有降低但仍可用。

**Q: 输出坐标精度能达到多少?**
A: 一般能达到厘米级（±1-2cm），优化设置可达5mm级。

## 参考资源

- [VGGSfM GitHub](https://github.com/facebookresearch/vggsfm)
- [AprilTag官网](https://april.eecs.umich.edu/software/apriltag/)
- [COLMAP文档](https://colmap.github.io/)
- [ROS Bag Format](http://wiki.ros.org/Bags)

---

**最后更新**: 2026年2月  
**维护者**: VGGSfM工作流团队
