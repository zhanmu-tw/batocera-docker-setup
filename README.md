# Docker + GPU Setup on Batocera

Complete guide for installing Docker with GPU support on Batocera Linux.

If you are not interested in installing GPU support, ignore any instructions marked \*[GPU]

I realized that my batocera VM is sitting there doing nothing for most of the day, but as a dedicated GPU for those rare times I'm allowed free time. I also like to have monitoring on my server for temperatures and usages, so having the dGPU dedicated to a VM limited me. I needed docker.

## Overview

This guide enables Docker containers with NVIDIA GPU access on Batocera, allowing you to run AI workloads, monitoring tools, and GPU-accelerated applications alongside your gaming setup.

**Tested Configuration:**

- Batocera 41 (Buildroot 2024.05.2)
- Kernel: 6.11.10
- NVIDIA GTX 1080 Ti (11GB)
- NVIDIA Driver: 560.35.03

---

## Prerequisites

- Batocera installed and running
- NVIDIA GPU passed through to Batocera VM (if running in VM) \*[GPU]
- SSH access to Batocera

---

## 1. Docker Installation

### Install Docker 27.3.1 LTS

```bash
cd /userdata/system
mkdir -p docker/bin
cd docker

# Download Docker 27.3.1 LTS
wget https://download.docker.com/linux/static/stable/x86_64/docker-27.3.1.tgz
tar xzvf docker-27.3.1.tgz
mv docker/* bin/
rm -rf docker docker-27.3.1.tgz

# Create Docker data directory
mkdir -p /userdata/docker-data

# Verify installation
/userdata/system/docker/bin/docker --version
# Should output: Docker version 27.3.1

```

---

## 2. NVIDIA Utilities Setup \*[GPU]

### Check Your Driver Version

First, verify your NVIDIA driver version:

```bash
modinfo nvidia | grep version
```

### Extract nvidia-smi and Libraries

**Important:** Replace `560.35.03` with your actual driver version from the previous step.

```bash
cd /userdata/system
mkdir -p nvidia-utils

# Download nvidia driver package matching your kernel module version
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/560.35.03/NVIDIA-Linux-x86_64-560.35.03.run
chmod +x NVIDIA-Linux-x86_64-560.35.03.run

# Extract (don't install)
./NVIDIA-Linux-x86_64-560.35.03.run --extract-only
cd NVIDIA-Linux-x86_64-560.35.03

# Copy utilities
cp nvidia-smi /userdata/system/nvidia-utils/
cp nvidia-debugdump /userdata/system/nvidia-utils/
cp nvidia-persistenced /userdata/system/nvidia-utils/
cp nvidia-cuda-mps-control /userdata/system/nvidia-utils/
cp nvidia-cuda-mps-server /userdata/system/nvidia-utils/

# Copy libraries
mkdir -p /userdata/system/nvidia-utils/lib
cp libnvidia-ml.so.* /userdata/system/nvidia-utils/lib/
cp libcuda.so.* /userdata/system/nvidia-utils/lib/
cp libnvidia-ptxjitcompiler.so.* /userdata/system/nvidia-utils/lib/

# Clean up
cd /userdata/system
rm -rf NVIDIA-Linux-x86_64-560.35.03 NVIDIA-Linux-x86_64-560.35.03.run
```

### Create Library Symlinks

Containers need unversioned library names. Adjust version numbers to match your files:

```bash
cd /userdata/system/nvidia-utils/lib/

# Create symlinks (adjust version numbers if different)
ln -sf libcuda.so.560.35.03 libcuda.so.1
ln -sf libcuda.so.560.35.03 libcuda.so
ln -sf libnvidia-ml.so.560.35.03 libnvidia-ml.so.1
ln -sf libnvidia-ml.so.560.35.03 libnvidia-ml.so
ln -sf libnvidia-ptxjitcompiler.so.560.35.03 libnvidia-ptxjitcompiler.so.1
ln -sf libnvidia-ptxjitcompiler.so.560.35.03 libnvidia-ptxjitcompiler.so

# Verify symlinks
ls -la
```

---

## 3. Batocera Service Creation

In troubleshooting this, I found that the service needed to mount cgroup v1, and that cgroup2 was mounted. As such the service script will ensure this is set during the service startup

### Download the script

Found in this repo called docker and save it to /userdata/system/services/docker

### Make executable

```bash
chmod +x /userdata/system/services/docker
```

## 4. System PATH Configuration

This involves editing a file in profile.d which is not saved across reboot, however batocera provides a command called batocera-save-overlay to persist changes

Add Docker to your system PATH:

```bash
# Create profile script
cat > /etc/profile.d/99-docker.sh << 'EOF'
# Docker and Nvidia utilities
export PATH=/userdata/system/nvidia-utils:/userdata/system/docker/bin:$PATH

# Uncomment to allow running nvidia-smi in the terminal
# export LD_LIBRARY_PATH=/userdata/system/nvidia-utils/lib:$LD_LIBRARY_PATH
EOF

chmod +x /etc/profile.d/99-docker.sh

# Save to overlay (persists across reboots)
batocera-save-overlay

# Apply to current session
source /etc/profile.d/99-docker.sh
```

---

## 5. Start and Test

### Start Docker Service

```bash
# Start Docker
batocera-services start docker

# Verify Docker is running
docker ps

# Verify GPU is accessible on host *[GPU]
 nvidia-smi

# Verify cgroup v1 is properly mounted
mount | grep cgroup | grep devices
# Should show: devices on /sys/fs/cgroup/devices type cgroup
```

### Test GPU in Container \*[GPU]

```bash
docker run --rm --privileged \
  -v /userdata/system/nvidia-utils:/usr/local/nvidia:ro \
  -e LD_LIBRARY_PATH=/usr/local/nvidia/lib \
  -e PATH=/usr/local/nvidia:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
  nvidia/cuda:12.6.0-base-ubuntu22.04 \
  nvidia-smi
```

**Expected output:**

```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 560.35.03              Driver Version: 560.35.03      CUDA Version: 12.6     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
|   0  NVIDIA GeForce GTX 1080 Ti     Off |   00000000:06:00.0  On |                  N/A |
+-----------------------------------------------------------------------------------------+
```

---

## Service Management

```bash
# Start Docker
batocera-services start docker

# Stop Docker
batocera-services stop docker

# Restart Docker
batocera-services restart docker

# Check status
batocera-services status docker
```

---

## Running GPU Containers \*[GPU]

### Standard GPU Container Command

For all GPU-enabled containers, use this format:

```bash
docker run --rm --privileged \
  -v /userdata/system/nvidia-utils:/usr/local/nvidia:ro \
  -e LD_LIBRARY_PATH=/usr/local/nvidia/lib \
  -e PATH=/usr/local/nvidia:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
  [other options] \
  [image] \
  [command]
```

### Example: Ollama (LLM Inference)

```bash
docker run -d \
  --name ollama \
  --privileged \
  -v /userdata/system/nvidia-utils:/usr/local/nvidia:ro \
  -e LD_LIBRARY_PATH=/usr/local/nvidia/lib \
  -e PATH=/usr/local/nvidia:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
  -v /userdata/docker-data/ollama:/root/.ollama \
  -p 11434:11434 \
  ollama/ollama
```

---

## Troubleshooting

### Container fails with BPF error

```
error setting cgroup config for procHooks process: bpf_prog_query(BPF_CGROUP_DEVICE) failed
```

**Solution:** Check that cgroup v1 is mounted:

```bash
mount | grep cgroup
```

Should show `devices on /sys/fs/cgroup/devices type cgroup`, NOT `cgroup2`.

If cgroup v2 is mounted, restart the Docker service:

```bash
batocera-services restart docker
```

### nvidia-smi not found in container \*[GPU]

**Solution:** Check that library symlinks exist:

```bash
ls -la /userdata/system/nvidia-utils/lib/
```

You should see symlinks like `libnvidia-ml.so` -> `libnvidia-ml.so.560.35.03`

### Docker commands not found after reboot

**Solution:** Verify PATH configuration persisted:

```bash
cat /etc/profile.d/99-docker.sh
```

If missing, re-run the PATH configuration section and ensure you run `batocera-save-overlay`.

---

## Technical Details

### Why cgroup v1?

Batocera's kernel (6.11.10) with cgroup v2 uses BPF for device control. Docker's runc cannot properly utilize this BPF device controller, causing containers to fail when accessing devices like GPUs.

Cgroup v1 uses a different, older device controller mechanism that works reliably with Docker.

### File Locations

- Docker binaries: `/userdata/system/docker/bin/`
- Docker data: `/userdata/docker-data/`
- NVIDIA utilities: `/userdata/system/nvidia-utils/` \*[GPU]
- Service script: `/userdata/system/services/docker`
- PATH configuration: `/etc/profile.d/99-docker.sh`
- Docker logs: `/userdata/system/docker/docker.log`

### Software Versions

- **Docker:** 27.3.1 LTS
- **NVIDIA Driver:** 560.35.03 (match to your system) \*[GPU]

---

## Security Considerations

This setup uses `--privileged` for containers. This is acceptable because:

- Batocera runs as root by default
- This is a single-user system
- The entire system is in a VM with hardware passthrough
- Not hosting internet-facing services

## Bonus

Run portainer

```bash
# Create directory for Portainer Agent data (if needed)
mkdir -p /userdata/containers/portainer-agent

# Run Portainer Agent
docker run -d \
  --name portainer-agent \
  --privileged \
  --restart always \
  -e PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  -v /:/host \
  -p 9001:9001 \
  portainer/agent:latest
```
