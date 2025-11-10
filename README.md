# Jetson Jupyter Lab Setup Guide

Complete guide for running GPU-accelerated Jupyter Lab on NVIDIA Jetson using Docker.

---

## 📋 Prerequisites

- NVIDIA Jetson device (Orin/Xavier/Nano) running JetPack
- Docker installed with `nvidia-container-runtime`
- SSH or direct terminal access to your Jetson

---

## 🚀 Quick Start

### 1. Start Jupyter Lab

Run this in **Terminal 1** (keep this window open):

```bash
docker run -it --rm --gpus all -p 8888:8888 \
  -v ~/my_jupyter_work:/workspace \
  dustynv/l4t-ml:r36.4.0 \
  jupyter lab --ip=0.0.0.0 --port=8888 --allow-root \
  --IdentityProvider.token='mynotebook' \
  --ServerApp.root_dir=/workspace
```

### 2. Access in Browser

Open this URL immediately:
```
http://127.0.0.1:8888/lab?token=mynotebook
```

---

## 🔍 Verify GPU Access

Inside Jupyter Lab, create a new notebook and run:

```python
import torch
print(f"GPU available: {torch.cuda.is_available()}")
print(f"GPU name: {torch.cuda.get_device_name(0)}")
```

**Expected output:**
```
GPU available: True
GPU name: Orin
```

---

## 🛠️ Command Breakdown

### Container Management

```bash
# List running containers
docker ps --filter ancestor=dustynv/l4t-ml:r36.4.0

# Stop Jupyter Lab (from Terminal 2)
docker stop $(docker ps -q --filter ancestor=dustynv/l4t-ml:r36.4.0)

# Force stop if needed
docker kill $(docker ps -q --filter ancestor=dustynv/l4t-ml:r36.4.0)
```

### Token Retrieval

```bash
# Get token while Jupyter is running
docker exec $(docker ps -q --filter ancestor=dustynv/l4t-ml:r36.4.0) \
  python3 -c "import json; d=json.load(open('/root/.local/share/jupyter/runtime/jpserver-1.json')); print(d['token'])"

# Alternative: Extract from logs
docker logs $(docker ps -q --filter ancestor=dustynv/l4t-ml:r36.4.0) 2>&1 | grep -o "token=[a-z0-9]*" | head -1
```

### Check GPU Access

```bash
# Verify nvidia-smi inside container
docker exec $(docker ps -q --filter ancestor=dustynv/l4t-ml:r36.4.0) nvidia-smi

# Test PyTorch GPU
docker exec $(docker ps -q --filter ancestor=dustynv/l4t-ml:r36.4.0) \
  python3 -c "import torch; print(torch.cuda.is_available())"
```

---

## 📁 Persistent Workspace

The `-v ~/my_jupyter_work:/workspace` flag creates a persistent folder:

- **Host location**: `/home/lion/my_jupyter_work`
- **Jupyter location**: `/workspace`

**Verify files are saved:**
```bash
ls ~/my_jupyter_work
```

---

## 🔧 Troubleshooting

### Port Already Allocated
```bash
# Error: "Bind for 0.0.0.0:8888 failed"
# Solution: Stop existing container
docker stop $(docker ps -q --filter ancestor=dustynv/l4t-ml:r36.4.0)
```

### Empty Token File
```bash
# Error: No token in jpserver-*.json
# Solution: Start with explicit token
--IdentityProvider.token='mynotebook'
```

### GPU Not Available
```bash
# Check Docker runtime
docker info | grep -i runtime

# Restart with explicit GPU runtime
docker run --runtime nvidia -it --rm --gpus all ...
```

### Cannot Access from Another Device
```bash
# Find Jetson IP
hostname -I | awk '{print $1}'

# Use: http://<JETSON_IP>:8888/lab?token=mynotebook
```

---

## 📌 Terminal Window Guide

| Window | Purpose | Commands |
|--------|---------|----------|
| **Terminal 1** | Run Jupyter server | Keep this open with `docker run ...` |
| **Terminal 2** | Docker management, checks | `docker exec`, `docker stop`, `ls ~/my_jupyter_work` |
| **Browser** | Use Jupyter Lab | `http://127.0.0.1:8888/lab?token=mynotebook` |

---

## 🧹 Cleanup & Shutdown

### Graceful Shutdown
1. **Terminal 1**: Press `Ctrl+C` twice
2. **Verify**: `docker ps` (should show no containers)

### Manual Cleanup
```bash
# Stop all matching containers
docker stop $(docker ps -q --filter ancestor=dustynv/l4t-ml:r36.4.0)

# Remove unused images (optional)
docker image prune -a --filter "label=version=r36.4.0"
```

---

## 💡 Advanced Usage

### Change Token & Port
```bash
docker run -it --rm --gpus all -p 8889:8889 \
  dustynv/l4t-ml:r36.4.0 \
  jupyter lab --ip=0.0.0.0 --port=8889 --allow-root \
  --IdentityProvider.token='your_secure_token_here'
```

### Mount Additional Volumes
```bash
-v /path/to/data:/data \
-v /path/to/models:/models \
```

### Run Jupyter Notebook (Classic) instead of Lab
Replace `jupyter lab` with `jupyter notebook`.

---

## 📦 Docker Image Info

- **Image**: `dustynv/l4t-ml:r36.4.0`
- **Built for**: JetPack 6.x (L4T R36.4.0)
- **Includes**: PyTorch, TensorFlow, CUDA, cuDNN, NumPy, SciPy, scikit-learn
- **Size**: ~12GB (first download required)

---

## 🎯 Quick Reference Card

```
Startup:  docker run ... --IdentityProvider.token='mynotebook'
Access:   http://127.0.0.1:8888/lab?token=mynotebook
Verify:   python -c "import torch; print(torch.cuda.is_available())"
Stop:     docker stop $(docker ps -q --filter ancestor=dustynv/l4t-ml:r36.4.0)
Workspace: ~/my_jupyter_work ↔ /workspace
```

---

**Last Updated**: 2025-11-10  
**Tested On**: Jetson Orin Nano, JetPack 6.1, L4T R36.4.0
