# 3D Point Cloud Reconstruction (World Coordinate System) Based on VGGSfM and AprilTag with an Intel RealSense D435 Depth Camera

This project is used for 3D reconstruction based on a depth vision camera. In the `output` directory of this project, three datasets have already been prepared; you can choose an appropriate dataset according to your GPU memory size. In the datasets, both RGB-D and depth information are provided.

This project aims to serve the creation of datasets for VLA models. It focuses on finding a method to reconstruct accurate 3D space from single-camera video, so that precise action information can be obtained for post-training of VLA models.

This folder contains a complete toolchain for performing 3D reconstruction from ROS bag files and calibrating objects using AprilTags.

## 📁 File Descriptions

| File | Description |
|------|-------------|
| `explain.py` | Environment check tool — verifies dependencies and configuration |
| `extract_rgbd_from_bag.py` | ROS bag RGB-D extractor — extracts RGB and depth images from a bag file |
| `apriltag_calibration.py` | AprilTag calibration tool — detects tags and computes 3D coordinates |
| `object_annotator.py` | Object annotation tool — interactive mouse-based labeling tool |
| `run_reconstruction_pipeline.sh` | Full automation script — runs the entire pipeline with one command |
| `WORKFLOW_README.md` | Detailed documentation — full workflow explanation and parameter tuning guide |
| `QUICK_START.md` | Quick start — onboarding guide for new users and common Q&A |

## 🚀 Quick Start

### 1. Verify the environment
```bash
cd /root/reconstruction_toolkit
python3 check_environment.py
```

### 2. Run the full pipeline automatically
```bash
bash run_reconstruction_pipeline.sh
```

### 3. Run each tool individually

**Extract RGB-D data:**
```bash
python3 extract_rgbd_from_bag.py /path/to/file.bag /output/dir
```

**Detect AprilTags:**
```bash
python3 apriltag_calibration.py /path/to/rgb/dir /path/to/metadata.json
```

**Annotate objects interactively:**
```bash
python3 object_annotator.py /path/to/sfm/sparse /path/to/metadata.json
```

## 📋 Workflow Steps

```
ROS bag file
    ↓
1️⃣ extract_rgbd_from_bag.py (RGB-D extraction)
    ↓
2️⃣ demo.py (VGGSfM 3D reconstruction)
    ↓
3️⃣ apriltag_calibration.py (tag detection + calibration)
    ↓
4️⃣ object_annotator.py (optional: manual annotation)
    ↓
📊 Final output report
```

## ⚙️ System Requirements

- Python 3.8+
- PyTorch 2.0+
- OpenCV 4.0+
- ROS (optional, for bag file processing)

## 📚 More Help

- **Detailed workflow guide**: see `WORKFLOW_README.md`
- **FAQ**: see the FAQ section in `QUICK_START.md`
- **Parameter tuning**: refer to the “parameter tuning” chapter in `WORKFLOW_README.md`

## 💾 Output Directory Structure

```
reconstruction_output/
├── [bag_name]/
│   ├── rgb/                    # RGB images
│   ├── depth/                  # Depth images
│   └── info/metadata.json      # Camera intrinsics
├── [bag_name]_sfm/
│   └── sparse/                 # COLMAP reconstruction results
├── apriltag_detections/        # Tag detection visualizations
├── calibration.json            # Calibration result coordinates
└── RECONSTRUCTION_REPORT.md    # Final report
```
