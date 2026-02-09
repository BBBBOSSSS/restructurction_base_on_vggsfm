# Bag文件处理指南

## 问题

您的bag文件使用了lz4压缩，但当前的Python rosbag库不支持lz4解压。

## 解决方案

### 方案A: 使用完整的ROS环境（推荐）

如果您有ROS环境安装，可以用rosbag命令行工具解压：

```bash
# 解压bag文件
rosbag decompress /root/vggsfm/data/20260205_163837.bag

# 然后提取数据
python3 /root/reconstruction_toolkit/extract_rgbd_from_bag.py \
  /root/vggsfm/data/20260205_163837.bag \
  /root/vggsfm/reconstruction_output/20260205_163837
```

### 方案B: Docker/ROS容器

在Docker中运行：

```bash
docker run -it -v /root/vggsfm/data:/data ros:latest bash
rosbag decompress /data/20260205_163837.bag
```

### 方案C: 在线工具

访问ROS社区在线工具进行bag文件转换。

### 方案D: 手动提取（如果您知道bag文件内的topic）

```python
# 如果有topic信息，可以使用脚本中的extract_rgbd函数
python3 -c "
from extract_rgbd_from_bag import extract_rgbd
extract_rgbd('/data/20260205_163837_uncompressed.bag', '/output')
"
```

## 临时解决方案

为了能继续使用工作流，可以：

1. 使用示例数据进行演示
2. 在Linux系统上安装ROS Noetic或Melodic
3. 联系系统管理员配置ROS环境

## 相关文件

- `extract_rgbd_from_bag.py` - RGB-D 提取脚本
- `decompress_bag.py` - 尝试解压工具（需要完整ROS）
- `/root/vggsfm/reconstruction_output/` - 处理结果目录

## 技术细节

rosbag lz4支持需要：
- python-lz4
- 重新编译的rosbag C++扩展
- 或完整的ROS2关键系统组件

当前环境中这些组件要么缺失，要么版本相关，导致无法处理lz4压缩。
