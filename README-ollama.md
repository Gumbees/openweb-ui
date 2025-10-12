# Ollama Standalone Deployment

This document describes how to deploy Ollama as a standalone service with GPU acceleration, separated from the main Open-WebUI stack.

## Architecture

The Ollama service has been extracted into its own Docker Compose file (`docker-compose-ollama.yml`) to enable:

- **GPU Acceleration**: Optimized configuration for AMD ROCm and NVIDIA CUDA
- **Scalability**: Independent scaling and resource allocation
- **Isolation**: Better resource isolation and management
- **Flexibility**: Can be deployed on different nodes or infrastructure

## Prerequisites

### For AMD GPUs (ROCm)
```bash
# Ensure ROCm drivers are installed on the host
# Ubuntu/Debian
sudo apt update
sudo apt install rocm-dev rocm-libs

# Add user to render group
sudo usermod -a -G render $USER
```

### For NVIDIA GPUs (CUDA)
```bash
# Install NVIDIA Container Toolkit
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
sudo systemctl restart docker
```

## Setup Instructions

### 1. Create Required Networks

Before deploying either service, create the external networks:

```bash
# Create the Ollama internal network
docker network create --driver overlay --attachable ollama_network

# Create the shared Open-WebUI stack network
docker network create --driver overlay --attachable openwebui_stack
```

### 2. Prepare Volume Directory

Create the bind mount directory for Ollama data:

```bash
sudo mkdir -p /mnt/docker_vol/openweb_ui/ollama
sudo chown -R $USER:$USER /mnt/docker_vol/openweb_ui/ollama
```

### 3. Configure Environment Variables

Copy and customize the environment file:

```bash
cp env.example .env
```

Edit `.env` and configure the Ollama-specific variables:

```bash
# For AMD GPUs
OLLAMA_IMAGE_TAG=rocm
ROCR_VISIBLE_DEVICES=all

# For NVIDIA GPUs (uncomment these lines in docker-compose-ollama.yml)
# OLLAMA_IMAGE_TAG=latest
# NVIDIA_VISIBLE_DEVICES=all
# NVIDIA_DRIVER_CAPABILITIES=compute,utility

# Adjust memory limits based on your hardware
OLLAMA_MEMORY_LIMIT=8G

# Network configuration
OLLAMA_NETWORK_NAME=ollama_network
OPENWEBUI_STACK_NETWORK=openwebui_stack
```

### 4. Deploy Ollama Service

Deploy the standalone Ollama service:

```bash
# Deploy using Docker Compose
docker-compose -f docker-compose-ollama.yml up -d

# Or using Docker Swarm
docker stack deploy -c docker-compose-ollama.yml ollama
```

### 5. Deploy Open-WebUI Stack

Deploy the main Open-WebUI stack:

```bash
# Deploy using Docker Compose
docker-compose up -d

# Or using Docker Swarm
docker stack deploy -c docker-compose.yml openwebui
```

## Configuration Options

### GPU Selection

#### AMD GPUs
- Use `OLLAMA_IMAGE_TAG=rocm`
- Configure `ROCR_VISIBLE_DEVICES` to select specific GPUs
- Set appropriate `HSA_OVERRIDE_GFX_VERSION` for your GPU architecture

#### NVIDIA GPUs
- Use `OLLAMA_IMAGE_TAG=latest` or a CUDA-specific tag
- Uncomment NVIDIA runtime and device configurations in the compose file
- Configure `NVIDIA_VISIBLE_DEVICES` to select specific GPUs

### Performance Tuning

Adjust these environment variables in `.env`:

```bash
# Number of models to keep loaded simultaneously
OLLAMA_MAX_LOADED_MODELS=1

# Number of parallel requests
OLLAMA_NUM_PARALLEL=1

# How long to keep models loaded (5m, 10m, -1 for forever)
OLLAMA_KEEP_ALIVE=5m

# Memory limit for the container
OLLAMA_MEMORY_LIMIT=8G
```

### Network Configuration

The setup uses external networks to enable communication between services:

- `ollama_network`: Internal network for Ollama service
- `openwebui_stack`: Shared network for communication with Open-WebUI

## Verification

### Check Service Status

```bash
# Check Ollama service
docker-compose -f docker-compose-ollama.yml ps

# Check if Ollama API is responding
curl http://localhost:11434/api/tags
```

### Test GPU Acceleration

```bash
# Enter Ollama container
docker exec -it $(docker-compose -f docker-compose-ollama.yml ps -q ollama) bash

# Check GPU visibility
# For AMD:
rocm-smi

# For NVIDIA:
nvidia-smi
```

### Pull and Run a Model

```bash
# Pull a model
docker exec -it $(docker-compose -f docker-compose-ollama.yml ps -q ollama) ollama pull llama2:7b

# Test the model
curl -X POST http://localhost:11434/api/generate \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama2:7b",
    "prompt": "Hello, how are you?",
    "stream": false
  }'
```

## Troubleshooting

### GPU Not Detected

#### AMD GPUs
```bash
# Check ROCm installation
rocm-smi

# Verify device permissions
ls -la /dev/kfd /dev/dri

# Check container GPU access
docker exec -it $(docker-compose -f docker-compose-ollama.yml ps -q ollama) rocm-smi
```

#### NVIDIA GPUs
```bash
# Check NVIDIA runtime
nvidia-smi

# Verify container toolkit
docker run --rm --gpus all nvidia/cuda:11.0-base nvidia-smi

# Check container GPU access
docker exec -it $(docker-compose -f docker-compose-ollama.yml ps -q ollama) nvidia-smi
```

### Network Issues

```bash
# Check if networks exist
docker network ls | grep -E "(ollama_network|openwebui_stack)"

# Create missing networks
docker network create --driver overlay --attachable ollama_network
docker network create --driver overlay --attachable openwebui_stack

# Check network connectivity
docker exec -it $(docker-compose -f docker-compose-ollama.yml ps -q ollama) ping open-webui
```

### Performance Issues

1. **Memory**: Increase `OLLAMA_MEMORY_LIMIT` if models are getting killed
2. **GPU Memory**: Reduce `OLLAMA_MAX_LOADED_MODELS` if running out of VRAM
3. **CPU**: Adjust `OLLAMA_NUM_PARALLEL` based on your CPU cores
4. **Storage**: Ensure `/mnt/docker_vol/openweb_ui/ollama` has sufficient space

## Migration from Integrated Setup

If migrating from the previous integrated setup:

1. Stop the old stack: `docker-compose down`
2. Backup Ollama data: `sudo cp -r /var/lib/docker/volumes/openwebui_ollama_data_volume/_data /mnt/docker_vol/openweb_ui/ollama/`
3. Create networks as described above
4. Deploy the new setup

## Monitoring

Monitor resource usage:

```bash
# Container stats
docker stats $(docker-compose -f docker-compose-ollama.yml ps -q)

# GPU utilization
# AMD:
watch -n 1 rocm-smi

# NVIDIA:
watch -n 1 nvidia-smi
```
