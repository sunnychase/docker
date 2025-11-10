# Jetson Jupyter Lab Setup & Optimization Guide

Complete workflow for running GPU-accelerated Jupyter Lab on NVIDIA Jetson devices with Docker, from startup to teardown.

---

## 📋 Prerequisites

- NVIDIA Jetson device (Orin/Xavier/Nano) running JetPack 6.x
- Docker installed with `nvidia-container-runtime`
- SSH or direct terminal access to your Jetson
- ~15GB free disk space for container image

---

## 🚀 Quick Start: Optimized Setup

**Copy-paste these commands to start immediately with full optimizations:**

```bash
# Terminal 1 (keep open): Start Jupyter with shared memory
docker run -it --rm --gpus all --shm-size=2g -p 8888:8888 \
  -v ~/my_jupyter_work:/workspace \
  dustynv/l4t-ml:r36.4.0 \
  jupyter lab --ip=0.0.0.0 --port=8888 --allow-root \
  --IdentityProvider.token='mynotebook' \
  --ServerApp.root_dir=/workspace

# Browser: Access Jupyter Lab
# URL: http://127.0.0.1:8888/lab?token=mynotebook
```

---

## 🛠️ Detailed Setup & Optimization

### **Step 1: Enable ZRAM (One-Time Setup)**

ZRAM provides ~50% more compressed virtual memory. Run in **Terminal 2** (new terminal):

```bash
sudo systemctl enable nvzramconfig.service
sudo systemctl start nvzramconfig.service
zramctl  # Verify: should show compressed swap devices
```

**Verify output** (example on 8GB Orin Nano):
```
NAME       ALGORITHM DISKSIZE DATA COMPR TOTAL STREAMS MOUNTPOINT
/dev/zram0 lzo-rle       635M   4K   74B   12K       6 [SWAP]
/dev/zram1 lzo-rle       635M   4K   74B   12K       6 [SWAP]
/dev/zram2 lzo-rle       635M   4K   74B   12K       6 [SWAP]
...
```

---

### **Step 2: Start Jupyter with Performance Optimizations**

**Critical**: Use `--shm-size=2g` for PyTorch DataLoader shared memory.

In **Terminal 1** (keep this window open):

```bash
docker run -it --rm --gpus all --shm-size=2g -p 8888:8888 \
  -v ~/my_jupyter_work:/workspace \
  dustynv/l4t-ml:r36.4.0 \
  jupyter lab --ip=0.0.0.0 --port=8888 --allow-root \
  --IdentityProvider.token='mynotebook' \
  --ServerApp.root_dir=/workspace
```

**Key flags explained**:
- `--gpus all` : Enables GPU access
- `--shm-size=2g` : **Required** for PyTorch DataLoader shared memory
- `-v ~/my_jupyter_work:/workspace` : Persists notebooks to host
- `--IdentityProvider.token='mynotebook'` : Sets login token
- `--ServerApp.root_dir=/workspace` : Sets default directory

---

### **Step 3: Access Jupyter Lab**

Open your web browser and navigate to:

```
http://127.0.0.1:8888/lab?token=mynotebook
```

**From another device on your network**, find your Jetson's IP first:

```bash
hostname -I | awk '{print $1}'
```

Then use: `http://<JETSON_IP>:8888/lab?token=mynotebook`

---

### **Step 4: Verify GPU Access**

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

### **Step 5: Maximize Performance (Optional)**

In **Terminal 2**, while Jupyter is running:

```bash
# Set MAXN power mode (maximum performance, adjust -m number if needed)
sudo nvpmodel -m 0

# Maximize CPU/GPU clocks (increases power & heat - ensure cooling)
sudo jetson_clocks

# Monitor real-time performance
tegrastats
```

Press `q` to exit `tegrastats`.

**Warning**: Increases power consumption and heat. To restore defaults:
```bash
sudo jetson_clocks --restore
```

---

## 🔍 Monitoring & Management

### **Container Management**

```bash
# List running containers
docker ps --filter ancestor=dustynv/l4t-ml:r36.4.0

# View container resource usage
docker stats $(docker ps -q --filter ancestor=dustynv/l4t-ml:r36.4.0)

# Get token while Jupyter is running
docker exec $(docker ps -q --filter ancestor=dustynv/l4t-ml:r36.4.0) \
  python3 -c "import json; d=json.load(open('/root/.local/share/jupyter/runtime/jpserver-1.json')); print(d['token'])"

# Check GPU inside container
docker exec $(docker ps -q --filter ancestor=dustynv/l4t-ml:r36.4.0) nvidia-smi
```

### **Check Memory Status**

```bash
# Overall memory (including ZRAM)
free -h

# GPU memory usage
tegrastats
```

---

## 🔧 Troubleshooting

### **Port Already Allocated**
```bash
# Error: "Bind for 0.0.0.0:8888 failed"
# Solution: Stop existing container
docker stop $(docker ps -q --filter ancestor=dustynv/l4t-ml:r36.4.0)
```

### **Empty Token File**
```bash
# Error: No token in jpserver-*.json
# Solution: Start with explicit token (already in startup command)
--IdentityProvider.token='mynotebook'
```

### **GPU Not Available**
```bash
# Check Docker runtime
docker info | grep -i runtime

# Should show: nvidia
# If not, restart with explicit GPU runtime:
docker run --runtime nvidia -it --rm --gpus all ...
```

### **Out of Memory Errors**
```bash
# Ensure ZRAM is enabled
zramctl

# Check container memory usage
docker stats $(docker ps -q --filter ancestor=dustynv/l4t-ml:r36.4.0)

# Reduce batch size in your code
```

### **PyTorch DataLoader Crashes**
```bash
# ERROR: "RuntimeError: DataLoader worker exited unexpectedly"
# FIX: Ensure --shm-size=2g is in your docker run command
```

---

## 📌 Terminal Window Guide

| Window | Purpose | Keep Open? | Commands |
|--------|---------|------------|----------|
| **Terminal 1** | Run Jupyter server | ✅ YES | `docker run ...` (startup command) |
| **Terminal 2** | Optimization & monitoring | ⚠️ As needed | `sudo jetson_clocks`, `tegrastats`, `zramctl` |
| **Browser** | Jupyter Lab interface | ✅ YES (while working) | `http://127.0.0.1:8888/lab?token=mynotebook` |

---

## 🧹 Proper Teardown & Shutdown

### **Method 1: Recommended (Clean Shutdown)**

1. **In Jupyter Lab browser**: `File → Shut Down`
2. **Verify in Terminal 2**: Run `docker ps` (should show no containers)
3. Container automatically removed (due to `--rm` flag)

### **Method 2: Terminal Shutdown**

1. **In Jupyter Lab**: `File → Save All` or `Ctrl+S`
2. **Close browser tab** (optional)
3. **In Terminal 1**: Press `Ctrl+C` twice
4. **Verify in Terminal 2**: `docker ps` (should be empty)

---

## 💡 Advanced Usage

### **Change Token & Port**
```bash
docker run -it --rm --gpus all --shm-size=2g -p 8889:8889 \
  dustynv/l4t-ml:r36.4.0 \
  jupyter lab --ip=0.0.0.0 --port=8889 --allow-root \
  --IdentityProvider.token='your_secure_token_here'
```

### **Mount Additional Volumes**
```bash
docker run ... \
  -v ~/my_jupyter_work:/workspace \
  -v /path/to/data:/data \
  -v /path/to/models:/models \
  ...
```

### **Run Jupyter Notebook (Classic)**
Replace `jupyter lab` with `jupyter notebook` in startup command.

### **Limit Container Resources**
```bash
docker run ... \
  --memory=6g --memory-swap=10g \
  ...
```

---

## 📦 Docker Image Info

- **Image**: `dustynv/l4t-ml:r36.4.0`
- **Built for**: JetPack 6.x (L4T R36.4.0)
- **Includes**: PyTorch, TensorFlow, CUDA, cuDNN, NumPy, SciPy, scikit-learn
- **Size**: ~12GB (first download required)

---

## 🎯 Quick Reference Card

```
╔════════════════════════════════════════════════════════╗
║        JETSON JUPYTER LAB QUICK REFERENCE              ║
╚════════════════════════════════════════════════════════╝

⚡ ONE-TIME SETUP:
  sudo systemctl enable nvzramconfig.service
  sudo systemctl start nvzramconfig.service

🚀 STARTUP (Terminal 1):
  docker run -it --rm --gpus all --shm-size=2g -p 8888:8888 \
    -v ~/my_jupyter_work:/workspace \
    dustynv/l4t-ml:r36.4.0 \
    jupyter lab --ip=0.0.0.0 --port=8888 --allow-root \
    --IdentityProvider.token='mynotebook' \
    --ServerApp.root_dir=/workspace

🌐 ACCESS:
  http://127.0.0.1:8888/lab?token=mynotebook

✅ GPU TEST:
  import torch; print(torch.cuda.is_available())

🔥 PERFORMANCE (Terminal 2):
  sudo nvpmodel -m 0
  sudo jetson_clocks
  tegrastats

💾 WORKSPACE:
  Host: ~/my_jupyter_work
  Jupyter: /workspace

🛑 SHUTDOWN:
  File → Shut Down (in browser)
  # Or: Ctrl+C twice in Terminal 1

🔍 VERIFY:
  docker ps  # Should return nothing
  ls ~/my_jupyter_work  # Check saved files
  zramctl  # Check ZRAM active
```

---

## 📚 Additional Resources

- [Jetson AI Lab RAM Optimization](https://www.jetson-ai-lab.com/tips_ram-optimization.html)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)
- [Jetson Containers](https://github.com/dusty-nv/jetson-containers)

---

**Version**: 2.0 (Optimized)  
**Tested On**: Jetson Orin Nano, JetPack 6.1, L4T R36.4.0  
**Last Updated**: 2025-11-10
