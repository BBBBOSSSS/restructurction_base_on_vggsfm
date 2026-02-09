# 3D重建与AprilTag标定工具包
本项目用于基于深度视觉相机的三维重构 本项目output目录中已经制作3个数据集可根据自己显存大小来选择合适的数据集，数据集中rgbd与深度信息均由Intel d435相机采集，环绕apriltag与物体半周模拟多视角机位，用于vggsfm点云重构
本项目旨在为制作vla模型数据集服务，旨在寻找单机位视频情况下，重构精确三维空间的方法，进而获取精准的action信息用于vla模型的后训练
此文件夹包含从ROS bag文件进行3D重建和AprilTag对象标定的完整工具链。

## 📁 文件说明

| 文件 | 描述 |
|------|------|
| `explain.py` | 环境检查工具 - 验证依赖和配置 |
| `extract_rgbd_from_bag.py` | ROS bag RGB-D提取器 - 从bag文件提取RGB和深度图像 |
| `apriltag_calibration.py` | AprilTag标定工具 - 检测标签并计算3D坐标 |
| `object_annotator.py` | 对象标注工具 - 交互式鼠标标注工具 |
| `run_reconstruction_pipeline.sh` | 完整自动化脚本 - 一键执行整个流程 |
| `WORKFLOW_README.md` | 详细文档 - 完整工作流说明和参数调整指南 |
| `QUICK_START.md` | 快速开始 - 新用户指引和常见问题 |

## 🚀 快速开始

### 1. 验证环境
```bash
cd /root/reconstruction_toolkit
python3 check_environment.py
```

### 2. 自动执行完整流程
```bash
bash run_reconstruction_pipeline.sh
```

### 3. 单独运行各工具

**提取RGB-D数据：**
```bash
python3 extract_rgbd_from_bag.py /path/to/file.bag /output/dir
```

**检测AprilTag：**
```bash
python3 apriltag_calibration.py /path/to/rgb/dir /path/to/metadata.json
```

**交互式标注对象：**
```bash
python3 object_annotator.py /path/to/sfm/sparse /path/to/metadata.json
```

## 📋 工作流步骤

```
ROS Bag文件
    ↓
1️⃣ extract_rgbd_from_bag.py (RGB-D提取)
    ↓
2️⃣ demo.py (VGGSfM 3D重建)
    ↓
3️⃣ apriltag_calibration.py (标签检测+标定)
    ↓
4️⃣ object_annotator.py (可选：手动标注)
    ↓
📊 最终输出报告
```

## ⚙️ 系统要求

- Python 3.8+
- PyTorch 2.0+
- OpenCV 4.0+
- ROS (可选，用于bag文件处理)

## 📚 更多帮助

- **详细工作流说明**: 查看 `WORKFLOW_README.md`
- **常见问题**: 查看 `QUICK_START.md` 的FAQ部分
- **参数调整**: 参考 `WORKFLOW_README.md` 的参数调整章节

## 💾 输出文件结构

```
reconstruction_output/
├── [bag_name]/
│   ├── rgb/                    # RGB图像
│   ├── depth/                  # 深度图像
│   └── info/metadata.json      # 相机内参
├── [bag_name]_sfm/
│   └── sparse/                 # COLMAP重建结果
├── apriltag_detections/        # 标签检测可视化
├── calibration.json            # 标定结果坐标
└── RECONSTRUCTION_REPORT.md    # 最终报告
```
