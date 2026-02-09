# 🚀 VGGSfM + AprilTag 完整工作流 - 快速开始指南

## 📋 您当前的情况

✓ 已有4个ROS bag文件 (Intel D435相机数据)  
✓ VGGSfM环境已配置  
✓ 所有脚本已生成  
✓ 依赖已安装  

位置: `/root/vggsfm/data/`
```
20260205_163837.bag
20260205_163857.bag
20260205_164014.bag
20260205_164034.bag
```

## 🎯 目标

1. ✅ 从ROS bag提取RGB-D图像
2. ✅ 运行VGGSfM进行三维重建  
3. ✅ 检测AprilTag标记
4. ✅ 标定桌面物体的三维坐标

---

## 🚀 立即开始（3条命令）

### 方案1: 全自动化 (推荐)

```bash
bash /root/run_reconstruction_pipeline.sh
```

**耗时**: ~10-30分钟 (取决于数据量)  
**输出**: `/root/vggsfm/reconstruction_output/`

---

### 方案2: 手动逐步 (可控性更好)

#### 第1步：提取RGB-D数据 (5-10分钟)

```bash
python3 /root/extract_rgbd_from_bag.py \
  /root/vggsfm/data/20260205_163837.bag \
  /tmp/scene1
```

**会生成**:
```
/tmp/scene1/
├── rgb/              # RGB图像
├── depth/            # 深度图像  
└── info/metadata.json  # 相机参数
```

#### 第2步：三维重建 (执行一个即可)

```bash
cd /root/vggsfm

# 简单版 (推荐，快速)
python3 demo.py SCENE_DIR=/tmp/scene1 \
  query_frame_num=1 \
  use_poselib=false \
  fine_tracking=false \
  query_method=sp

# 或者高质量版 (更慢但质量好)
python3 demo.py SCENE_DIR=/tmp/scene1 \
  query_frame_num=3 \
  use_poselib=false \
  fine_tracking=false
```

**会生成** `/tmp/scene1/sparse/`:
```
├── cameras.bin      # 相机参数
├── images.bin       # 图像和特征
├── points3D.bin     # 三维点云
└── points3D.txt     # 点云文本版
```

#### 第3步：检测AprilTag (1-2分钟)

```bash
python3 /root/apriltag_calibration.py \
  /tmp/scene1/info/metadata.json \
  /tmp/scene1/rgb \
  /tmp/scene1/depth
```

**会生成**:
- `/tmp/apriltag_detections/` - 检测结果可视化
- `/tmp/calibration.json` - 标定结果

#### 第4步：交互式物体标注 (可选，更精确)

```bash
python3 /root/object_annotator.py \
  /tmp/scene1/sparse \
  /tmp/scene1/info/metadata.json \
  /tmp/scene1/depth
```

**操作说明**:
- **左键点击** 标注物体
- **Space** 保存当前帧
- **N** 下一帧
- **Q** 退出

**会生成** `/tmp/object_annotations.json` - 详细的物体坐标

---

## 📊 理解输出结果

### 1. 元数据 (`metadata.json`)

```json
{
  "camera_info": {
    "fx": 618.0,        // 焦距像素数
    "fy": 618.0,
    "cx": 320.0,        // 主点
    "cy": 240.0
  },
  "frames": [
    {
      "frame_id": 0,
      "rgb_file": "000000_rgb.png",
      "depth_file": "000000_depth.png",
      "timestamp": 1707135600.5
    }
  ]
}
```

### 2. 标定结果 (`calibration.json`)

```json
{
  "calibrations": {
    "frame_0": {
      "tag_id": 0,
      "objects": [
        {
          "object_id": 0,
          "image_pos": [250, 300],              // 图像坐标 (像素)
          "camera_pos": [0.1, 0.15, 0.5],      // 相机坐标 (米)
          "world_pos": [0, 0.1, 0.45]          // 世界坐标 (相对AprilTag)
        }
      ]
    }
  }
}
```

### 3. 点云 (`points3D.txt`)

```
# 3D point list
# X Y Z R G B ERROR TRACK
1.234 5.678 2.345 255 128 64 0.5 3 0 1 1 2
...
```

---

## 🔍 可视化结果

### 方案1: 用Python绘图

```python
import json
import numpy as np
import matplotlib.pyplot as plt
from mpl_toolkits.mplot3d import Axes3D

# 加载数据
with open('/tmp/calibration.json') as f:
    data = json.load(f)

# 提取所有物体坐标
fig = plt.figure(figsize=(10, 8))
ax = fig.add_subplot(111, projection='3d')

for frame_key, frame_data in data['calibrations'].items():
    for obj in frame_data['objects']:
        x, y, z = obj['world_pos']
        ax.scatter([x], [y], [z], c='red', s=200, marker='o')

# 标注AprilTag位置
for tag_id, pos in data['apriltags'].items():
    ax.scatter(*pos, c='blue', s=300, marker='^', label=f'Tag {tag_id}')

ax.set_xlabel('X (m)')
ax.set_ylabel('Y (m)')  
ax.set_zlabel('Z (m)')
ax.legend()
plt.title('物体三维坐标标定结果')
plt.show()
```

### 方案2: COLMAP可视化

```bash
pip install colmap  # 如果还没装

colmap gui --database_path=/tmp/scene1/sparse/database.db
```

---

## ⚡ 性能建议

| 场景 | 参数 | 耗时 |
|------|------|------|
| 快速预览 | `query_frame_num=1, fine_tracking=False` | 2-5分钟 |
| 标准质量 | `query_frame_num=2, fine_tracking=False` | 5-10分钟 |
| 高质量 | `query_frame_num=3, fine_tracking=True` | 15-30分钟 |
| **GPU显存** | 预计需要 | ~6-10 GB |

## 🛠️ 故障排除

### 问题: CUDA 显存不足
```bash
# 降低参数
python3 demo.py SCENE_DIR=/tmp/scene1 \
  query_frame_num=1 \
  robust_refine=1 \
  fine_tracking=false \
  img_size=512
```

### 问题: 无法提取bag文件
```bash
# 检查rosbag
python3 -c "import rosbag"

# 或者手动转换 (如果有ROS环境)
rosbag decompose /root/vggsfm/data/yourfile.bag
```

### 问题: 没有检测到AprilTag  
- 检查图像中是否有AprilTag
- 确保AprilTag大小合理（5-15cm）
- 调整光照

---

## 📈 坐标系说明

```
世界坐标系 (以第一个AprilTag为原点)
        
        Z ↑ (垂直表面)
        |
        +----→ X (向右)
       /
      Y (向下)
      
AprilTag中心 = (0, 0, 0)
```

-  **图像坐标**: 像素 (1024x768等)
-  **相机坐标**: 米，Z轴指向相机前方
-  **世界坐标**: 米，相对于AprilTag中心

---

## 📁 完整输出目录结构

```
/root/vggsfm/reconstruction_output/
├── 20260205_163837/          # 第一个bag的提取结果
│   ├── rgb/
│   │   ├── 000000_rgb.png
│   │   └── ...
│   ├── depth/
│   │   ├── 000000_depth.png
│   │   └── ...
│   └── info/
│       └── metadata.json
├── 20260205_163837_sfm/      # SfM重建结果
│   ├── images/
│   └── sparse/
│       ├── cameras.bin
│       ├── images.bin
│       └── points3D.bin
├── apriltag_detections/      # AprilTag检测可视化
│   ├── frame_000000_tag_0.png
│   └── ...
└── RECONSTRUCTION_REPORT.md  # 最终报告
```

---

## 🎓 下一步建议

1. **验证结果**
   ```bash
   ls -lh /root/vggsfm/reconstruction_output/
   ```

2. **查看可视化**
   ```bash
   # 打开检测图像
   file:///root/vggsfm/reconstruction_output/apriltag_detections/
   ```

3. **提取物体坐标**
   ```python
   import json
   with open('calibration.json') as f:
       coords = json.load(f)
   # 使用 coords['calibrations'] 中的数据
   ```

4. **导入到ROS或其他系统**
   ```python
   # 发布为ROS service/topic
   # 或导出为CSV/Excel进行后处理
   ```

---

## 💡 常见问题

**Q: 精度能达到多少？**  
A: 通常厘米级 (±1-3cm)，优化设置可达毫米级。

**Q: 需要多少张图片？**  
A: 最少10张，20-50张最佳。

**Q: 可以离线处理吗？**  
A: 可以，所有工具都不需要网络连接。

**Q: 结果可以导出到哪些格式？**  
A: JSON, CSV, PLY (点云), COLMAP格式等。

---

## 📞 获取帮助

- 查看详细文档: `/root/WORKFLOW_README.md`
- 检查环境: `python3 /root/check_environment.py`
- 查看演示数据: `/root/vggsfm/examples/`

---

**准备好了吗？** 现在就运行:
```bash
bash /root/run_reconstruction_pipeline.sh
```

祝您重建成功！ 🎉
