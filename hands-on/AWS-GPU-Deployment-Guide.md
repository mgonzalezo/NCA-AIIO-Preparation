# AWS GPU Deployment Guide: NVIDIA GPU-Powered VM Setup

## Overview

This hands-on guide walks you through deploying a GPU-powered virtual machine on Amazon Web Services (AWS) with NVIDIA drivers, Docker, and the NVIDIA Container Toolkit. This lab is essential for understanding GPU infrastructure deployment and operations covered in the NCA-AIIO certification.

**What you'll accomplish:**
- Launch an EC2 GPU instance
- SSH into the instance and verify GPU/driver installation
- Install Docker and NVIDIA Container Toolkit
- Run a containerized GPU test to confirm GPU access

**Estimated time:** 25-35 minutes

---

## Prerequisites (5-10 min)

### Required

1. **AWS Account** with permissions to launch EC2 instances
2. **Key pair (PEM)** and a default VPC/subnet configured in your AWS account
3. **Basic terminal/SSH access** knowledge

### GPU Instance Types and Use Cases

Before launching, understand which instance type matches your workload:

| Instance Family | GPU Model | Use Case | Notes |
|----------------|-----------|----------|-------|
| **g6** | NVIDIA L4 | Inference/video | Great for inference and video processing |
| **g5** | NVIDIA A10G | Graphics/inference | Balanced graphics and AI inference |
| **p4d** | NVIDIA A100 | Training | High-performance training workloads |
| **p5** | NVIDIA H100/H200 | Frontier training | Cutting-edge large model training |

**References:**
- [AWS GPU Instance Types](https://aws.amazon.com/ec2/instance-types/#Accelerated_Computing)
- [NVIDIA Developer: AWS](https://developer.nvidia.com/aws)

### Service Quota Note

If this is your first time using GPU instances, you may need a **service quota increase** before launching p4/p5/g6 instances. AWS support can raise quotas upon request.

---

## Part A: Launch the EC2 GPU Instance (8-12 min)

### Step 1: Pick a Region

1. Log into the [AWS Console](https://console.aws.amazon.com/)
2. Navigate to **EC2** service
3. Choose an AWS region where your target instance family is available
   - Example: **us-east-1** (N. Virginia)

### Step 2: Choose an AMI (Amazon Machine Image)

1. In EC2, click **Launch instance**
2. Navigate to **Application and OS Images (Amazon Machine Image)**
3. Search for one of the following:

**Recommended for beginners:**
- **"Deep Learning Base GPU AMI (Amazon Linux 2/2023)"** or
- **"Deep Learning AMI GPU PyTorch"**

These AMIs include:
- Pre-installed NVIDIA drivers
- CUDA toolkit
- Maintained by AWS for compatibility

**Alternative:** Use a vanilla Ubuntu AMI and install drivers manually (see Optional section later)

**Best practice:** Prefer Deep Learning AMIs for a smoother first run. You can always use plain Ubuntu later and install drivers manually for learning purposes.

### Step 3: Choose Instance Type

Select an instance that matches your workload and budget:

**For lightweight GPU work and testing:**
- **g6.xlarge** (L4 GPU)

**For balanced graphics/inference:**
- **p4d.24xlarge** (A100 GPU)

**For heavy training:**
- **p5.48xlarge** (H100 GPU)

See [AWS Vantage Instances](https://instances.vantage.sh/) for detailed pricing comparison.

### Step 4: Configure Key Pair & Network

1. **Key pair:** Select your existing key pair (or create a new one)
2. **Network settings:**
   - Use the default VPC/subnet
   - Create a **Security Group** that allows:
     - **SSH (TCP port 22)** from your IP address

**Example Security Group rule:**
```
Type: SSH
Protocol: TCP
Port: 22
Source: My IP (automatically detects your IP)
```

### Step 5: Configure Storage

Set storage to **50-200 GB gp3 EBS** volume

**Why this size?**
- Containers require space
- Datasets need storage
- CUDA libraries and frameworks take significant disk space

### Step 6: Launch

1. Review your configuration
2. Click **Launch instance**
3. Wait until the instance state shows **Running**
4. Note the **Public IPv4 address** or **Public DNS**

**Expected result:**

```
Instance ID: i-0abc123def456789
Instance state: Running
Public IPv4 address: 54.123.45.67
Instance type: g6.xlarge
```

---

## Part B: Connect & Verify the GPU (3-5 min)

### Step 1: SSH into the Instance

Open your terminal and connect via SSH:

**For Amazon Linux 2/2023 AMI:**
```bash
ssh -i /path/to/your-key.pem ec2-user@EC2_PUBLIC_IP
```

**For Ubuntu-based AMI:**
```bash
ssh -i /path/to/your-key.pem ubuntu@EC2_PUBLIC_IP
```

Replace:
- `/path/to/your-key.pem` with your actual key file path
- `EC2_PUBLIC_IP` with your instance's public IP address

**Example:**
```bash
ssh -i ~/.ssh/my-gpu-key.pem ec2-user@54.123.45.67
```

**Expected output:**
```
The authenticity of host '54.123.45.67 (54.123.45.67)' can't be established.
ECDSA key fingerprint is SHA256:abcd1234efgh5678...
Are you sure you want to continue connecting (yes/no)? yes

       __|  __|_  )
       _|  (     /   Amazon Linux 2023
      ___|\___|___|

[ec2-user@ip-172-31-12-34 ~]$
```

### Step 2: Check the GPU & Driver

Run the NVIDIA System Management Interface command:

```bash
nvidia-smi
```

**Expected output (example with g6.xlarge / L4 GPU):**

```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.129.03             Driver Version: 535.129.03   CUDA Version: 12.2      |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  NVIDIA L4                      Off | 00000000:00:1E.0 Off |                    0 |
| N/A   32C    P8              14W / 72W  |      0MiB / 23034MiB |      0%      Default |
|                                         |                      |                  N/A |
+-----------------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

**What to verify:**
- **GPU Name**: Should show NVIDIA L4, A10G, A100, or H100 depending on instance type
- **Driver Version**: Confirms NVIDIA driver is installed (e.g., 535.129.03)
- **CUDA Version**: Shows supported CUDA version (e.g., 12.2)

If you used a **Deep Learning Base GPU AMI**, drivers are already present as shown above.

**Troubleshooting:**
- If `nvidia-smi: command not found`, drivers aren't installed. See "Optional: Manual driver/CUDA install" section below.

---

## Part C: Install Docker & NVIDIA Container Toolkit (8-12 min)

### Why Docker + NVIDIA Container Toolkit?

- **Docker**: Containerizes AI/ML applications for portability
- **NVIDIA Container Toolkit**: Enables GPU access inside containers

If your AMI already includes Docker, you can skip to "Add the NVIDIA Container Toolkit" section.

### Install Docker

#### For Amazon Linux 2023 / Amazon Linux 2:

```bash
sudo yum update -y
sudo yum install -y docker
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
```

**Expected output:**
```
Complete!
Installing : docker-20.10.x
...
Created symlink /etc/systemd/system/multi-user.target.wants/docker.service
```

#### For Ubuntu:

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo usermod -aG docker $USER
newgrp docker
```

**Expected output:**
```
Reading package lists... Done
Building dependency tree... Done
docker.io is already the newest version (24.0.5-0ubuntu1~22.04.1).
```

**Verify Docker installation:**
```bash
docker --version
```

**Expected output:**
```
Docker version 24.0.7, build 24.0.7-0ubuntu2~22.04.1
```

### Add the NVIDIA Container Toolkit

The NVIDIA Container Toolkit allows Docker containers to access NVIDIA GPUs.

#### Step 1: Add Repository and Key (Ubuntu/Debian)

```bash
# Add NVIDIA's repository and GPG key
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

**Expected output:**
```
deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://nvidia.github.io/libnvidia-container/stable/deb/$(ARCH) /
```

#### Step 2: Install NVIDIA Container Toolkit

```bash
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
```

**Expected output:**
```
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following NEW packages will be installed:
  nvidia-container-toolkit nvidia-container-toolkit-base
0 upgraded, 2 newly installed, 0 to remove and 12 not upgraded.
Need to get 12.5 MB of archives.
After this operation, 49.2 MB of additional disk space will be used.
...
Setting up nvidia-container-toolkit (1.14.3-1) ...
```

#### Step 3: Configure Docker to Use NVIDIA Runtime

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

**Expected output:**
```
INFO[0000] Loading config from /etc/docker/daemon.json
INFO[0000] Config file does not exist; using empty config
INFO[0000] Wrote updated config to /etc/docker/daemon.json
INFO[0000] It is recommended that docker daemon be restarted.
```

**Verify the NVIDIA runtime is configured:**
```bash
cat /etc/docker/daemon.json
```

**Expected output:**
```json
{
    "runtimes": {
        "nvidia": {
            "args": [],
            "path": "nvidia-container-runtime"
        }
    }
}
```

**References:**
- [NVIDIA Container Toolkit Documentation](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)

---

## Part D: Quick GPU Test Inside a Container (2-4 min)

### Run a CUDA Container and Check GPU Access

Now test that Docker containers can access the GPU:

```bash
docker run --rm --gpus all nvidia/cuda:12.5.0-runtime-ubuntu22.04 nvidia-smi
```

**Command breakdown:**
- `docker run`: Run a container
- `--rm`: Remove container after it exits
- `--gpus all`: Grant access to all GPUs
- `nvidia/cuda:12.5.0-runtime-ubuntu22.04`: Official NVIDIA CUDA image
- `nvidia-smi`: Command to run inside the container

**Expected output:**

```
Unable to find image 'nvidia/cuda:12.5.0-runtime-ubuntu22.04' locally
12.5.0-runtime-ubuntu22.04: Pulling from nvidia/cuda
aece8493d397: Pull complete
9ad3b22ad103: Pull complete
...
Status: Downloaded newer image for nvidia/cuda:12.5.0-runtime-ubuntu22.04

+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.129.03             Driver Version: 535.129.03   CUDA Version: 12.2      |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  NVIDIA L4                      Off | 00000000:00:1E.0 Off |                    0 |
| N/A   33C    P8              14W / 72W  |      0MiB / 23034MiB |      0%      Default |
|                                         |                      |                  N/A |
+-----------------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

**What this confirms:**
- Docker can successfully access the GPU
- The NVIDIA Container Toolkit is working correctly
- Your containerized workloads will have GPU access

**Alternative:** If you prefer NGC (NVIDIA GPU Cloud) images:
```bash
# First, log in with NGC API key if using private images
docker run --rm --gpus all nvcr.io/nvidia/cuda:12.5.0-base-ubuntu22.04 nvidia-smi
```

---

## Optional: Manual Driver/CUDA Install (If You Didn't Use a DLAMI)

If you launched a plain Ubuntu or Amazon Linux 2023 image and need to install drivers/CUDA manually, follow NVIDIA's official installation guide.

### Installation Steps

1. **Visit NVIDIA's CUDA Installation Guide for Linux:**
   - [NVIDIA CUDA Installation Guide](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/)

2. **Match your driver to the CUDA/toolkit version** you need

3. **Install the driver:**
   - Download and install the appropriate driver package
   - Reboot the instance after installation

4. **Verify installation:**
   ```bash
   nvidia-smi
   ```

**Note:** Using Deep Learning AMIs is recommended for beginners as drivers are pre-configured and tested.

---

## Troubleshooting

### Issue: `nvidia-smi: command not found`

**Cause:** NVIDIA driver isn't installed or path isn't set

**Solution:**
1. Install drivers via DLAMI or manually following [NVIDIA's Linux installation guide](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/)
2. Reboot the instance after installation
3. Verify with `nvidia-smi`

**References:**
- [AWS DLAMI Documentation](https://docs.aws.amazon.com/dlami/)
- [NVIDIA Driver Installation](https://docs.nvidia.com/datacenter/tesla/tesla-installation-notes/)

### Issue: Container Can't See GPUs

**Cause:** NVIDIA Container Toolkit not installed or Docker not configured

**Solution:**
1. Verify NVIDIA Container Toolkit is installed:
   ```bash
   nvidia-ctk --version
   ```
2. Check Docker daemon configuration:
   ```bash
   cat /etc/docker/daemon.json
   ```
   Should contain nvidia runtime configuration
3. Restart Docker:
   ```bash
   sudo systemctl restart docker
   ```

**Reference:** [NVIDIA Container Toolkit Documentation](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/)

### Issue: Wrong Instance Type for Workload

**Problem:** Using g6 (L4) for training or p5 (H100) for inference

**Solution:**
- **Inference/video**: Use **g6** (L4)
- **Graphics/inference**: Use **p4d** (A100)
- **Heavy training**: Use **p5/p5e** (H100/H200)

**Reference:** [AWS Instance Types Comparison](https://aws.amazon.com/ec2/instance-types/#Accelerated_Computing)

### Issue: SSH Connection Refused

**Cause:** Security Group doesn't allow SSH from your IP

**Solution:**
1. Go to EC2 Console → Security Groups
2. Find the security group attached to your instance
3. Edit inbound rules
4. Add rule: Type=SSH, Protocol=TCP, Port=22, Source=My IP

---

## Clean Up (IMPORTANT)

When you're finished with the lab, **clean up resources to avoid charges:**

### Option 1: Stop the Instance (Preserves instance, stops compute charges)

```bash
# From AWS Console:
# EC2 → Instances → Select instance → Instance state → Stop
```

**Note:** EBS storage charges still apply when stopped.

### Option 2: Terminate the Instance (Deletes everything, stops all charges)

```bash
# From AWS Console:
# EC2 → Instances → Select instance → Instance state → Terminate
```

**Warning:** Termination is permanent. All data on the instance will be lost unless you created an AMI snapshot.

### Verify Cleanup

1. Go to EC2 Dashboard
2. Check:
   - **Instances**: Should show "Terminated" or "Stopped"
   - **Volumes**: Should be deleted (if instance was terminated)
   - **Elastic IPs**: Release any unused Elastic IPs (charged when not attached)

---

## What You've Accomplished

You now have hands-on experience with:

- Launching GPU instances on AWS
- Verifying NVIDIA driver installation with `nvidia-smi`
- Installing and configuring Docker for GPU workloads
- Setting up the NVIDIA Container Toolkit
- Running containerized GPU applications
- Understanding GPU instance types for different workloads

### Next Steps

**Build on this foundation:**

1. **Deploy AI Frameworks:**
   - Pull PyTorch or TensorFlow GPU containers
   - Run training or inference workloads
   - Experiment with Triton Inference Server

2. **Explore Advanced Topics:**
   - Multi-GPU setups with NCCL
   - GPU monitoring and metrics (Prometheus + NVIDIA DCGM)
   - Kubernetes with GPU scheduling
   - NVIDIA MIG (Multi-Instance GPU) for A100/H100

3. **Cost Optimization:**
   - Use Spot Instances for training workloads (up to 90% savings)
   - Implement auto-scaling for inference
   - Use Reserved Instances for predictable workloads

**Related NCA-AIIO Topics:**
- GPU virtualization (vGPU, MIG)
- Container orchestration (Kubernetes)
- AI infrastructure monitoring
- Workload scheduling and resource allocation

---

## Additional Resources

### AWS Documentation
- [AWS EC2 GPU Instances](https://aws.amazon.com/ec2/instance-types/#Accelerated_Computing)
- [AWS Deep Learning AMIs](https://docs.aws.amazon.com/dlami/)
- [AWS GPU Instance Pricing](https://aws.amazon.com/ec2/pricing/on-demand/)

### NVIDIA Documentation
- [NVIDIA Driver Installation Guide](https://docs.nvidia.com/datacenter/tesla/tesla-installation-notes/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/)
- [NVIDIA CUDA Toolkit](https://developer.nvidia.com/cuda-toolkit)
- [NVIDIA NGC Catalog](https://catalog.ngc.nvidia.com/)

### Community Resources
- [NVIDIA Developer Forums](https://forums.developer.nvidia.com/)
- [AWS re:Post](https://repost.aws/)
- [Stack Overflow: CUDA tag](https://stackoverflow.com/questions/tagged/cuda)

---

**Lab Version:** 1.0 (February 2026)
**Based on:** NCA-AIIO Hands-on Lab Materials
