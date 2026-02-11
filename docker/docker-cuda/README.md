# Docker Setup for NVIDIA GPUs

This directory contains Docker configuration files for running LLaMA Factory with NVIDIA GPU support.

## Prerequisites

### Linux-specific Requirements

Before running the Docker container with GPU support, you need to install the following packages:

1. **Docker**: The container runtime
   ```bash
   # Ubuntu/Debian
   sudo apt-get update
   sudo apt-get install docker.io

   # Or install Docker Engine from the official repository:
   # https://docs.docker.com/engine/install/
   ```

2. **Docker Compose** (if using the docker-compose method):
   ```bash
   # Ubuntu/Debian
   sudo apt-get install docker-compose

   # Or install the latest version:
   # https://docs.docker.com/compose/install/
   ```

3. **NVIDIA Container Toolkit** (required for GPU support):
   ```bash
   # Add the NVIDIA GPG key and repository
   distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
   curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
   curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

   # Install nvidia-container-toolkit
   sudo apt-get update
   sudo apt-get install -y nvidia-container-toolkit

   # Restart Docker to apply changes
   sudo systemctl restart docker
   ```

   **Note**: Without `nvidia-container-toolkit`, the Docker container will not be able to access your NVIDIA GPU.

### Verify GPU Access

After installation, verify that Docker can access your GPU:

```bash
sudo docker run --rm --gpus all nvidia/cuda:12.4.0-base-ubuntu22.04 nvidia-smi
```

If successful, you should see your GPU information displayed.

## Usage

### Using Docker Compose (Recommended)

```bash
cd docker/docker-cuda/
docker compose up -d
docker compose exec llamafactory bash
```

### Using Docker Run

```bash
# Build the image
docker build -f ./docker/docker-cuda/Dockerfile \
    --build-arg PIP_INDEX=https://pypi.org/simple \
    --build-arg EXTRAS=metrics \
    -t llamafactory:latest .

# Run the container
docker run -dit --ipc=host --gpus=all \
    -p 7860:7860 \
    -p 8000:8000 \
    --name llamafactory \
    llamafactory:latest

# Enter the container
docker exec -it llamafactory bash
```

## Troubleshooting

### GPU Not Detected

If your GPU is not detected inside the container:

1. Ensure `nvidia-container-toolkit` is installed
2. Check that the Docker daemon has been restarted after installation
3. Verify your NVIDIA drivers are properly installed: `nvidia-smi`
4. Check Docker GPU support: `docker run --rm --gpus all ubuntu nvidia-smi`

### Permission Denied

If you get permission errors, ensure your user is in the docker group:

```bash
sudo usermod -aG docker $USER
# Log out and back in for changes to take effect
```

## DGX-SPARK (ARM64 / Grace Blackwell)

DGX-SPARK uses **ARM64**; the default CUDA image above is x86_64 and will not run on it. Use the dedicated Dockerfile and build **on the DGX-SPARK device** (or another linux/arm64 host):

**Prerequisites:** Docker with GPU access and [NGC login](https://docs.nvidia.com/dgx/dgx-spark/ngc.html) for the base image:

```bash
docker login nvcr.io
# Username: $oauthtoken, Password: <your NGC API key>
```

**Build (from repo root):**

```bash
docker build -f docker/docker-cuda/Dockerfile.dgx-spark \
    --build-arg PIP_INDEX=https://pypi.org/simple \
    -t llamafactory:dgx-spark .
```

**Run:**

```bash
docker run -it --rm --gpus=all --ipc=host \
    --ulimit memlock=-1 --ulimit stack=67108864 \
    -v "$PWD":/workspace \
    -p 7860:7860 -p 8000:8000 \
    llamafactory:dgx-spark
```

**Or with Docker Compose (from repo root):**

```bash
docker compose -f docker/docker-cuda/docker-compose.dgx-spark.yml up -d
docker compose -f docker/docker-cuda/docker-compose.dgx-spark.yml exec llamafactory bash
```

Reference: [NVIDIA LLaMA Factory playbook for DGX Spark](https://build.nvidia.com/spark/llama-factory).

**CI:** The GitHub Actions workflow (`.github/workflows/docker.yml`) builds and pushes the DGX-SPARK image to **GitHub Container Registry (GHCR)** as `ghcr.io/<owner>/llamafactory:latest-dgx-spark` (or `:<version>-dgx-spark` on release). The workflow **tries anonymous pulls from `nvcr.io` first**; if pulling the base image fails (e.g. 401/403), add an NGC API key as repository secret `NGC_API_KEY` (Settings → Secrets and variables → Actions). Get the key from [NGC Setup → API Key](https://ngc.nvidia.com/setup/api-key).

## Additional Notes

- The default image is built on Ubuntu 22.04 (x86_64), CUDA 12.4, Python 3.11, PyTorch 2.6.0, and Flash-attn 2.7.4
- For different CUDA versions, you may need to adjust the base image in the Dockerfile
- Make sure your NVIDIA driver version is compatible with the CUDA version used in the Docker image
