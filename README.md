# Open-WebUI Docker Compose Setup

This Docker Compose configuration provides a complete Open-WebUI stack with Ollama for local AI model hosting, following the same architecture patterns as enterprise-grade applications.

## Features

- **Open-WebUI**: Self-hosted web interface for AI chat
- **Ollama**: Local AI model hosting (optional but recommended)
- **PostgreSQL**: Database for persistent data (optional - uses SQLite by default)
- **Redis**: Caching layer (optional but recommended)
- **Cloudflare Tunnel**: Secure remote access (optional)
- **Network Isolation**: Secure internal networks
- **GPU Support**: NVIDIA GPU support for Ollama
- **ARM64 Compatible**: Works on Apple Silicon and ARM64 systems

## Quick Start

1. **Copy and configure environment variables:**
   ```bash
   cp env.example .env
   # Edit .env file with your preferred settings
   ```

2. **Generate a secret key:**
   ```bash
   # Generate a secure secret key
   openssl rand -hex 32
   # Add this to WEBUI_SECRET_KEY in your .env file
   ```

3. **Create the external network:**
   ```bash
   docker network create containers_internet
   ```

4. **Start the services:**
   ```bash
   docker compose up -d
   ```

5. **Access Open-WebUI:**
   - Open your browser to `http://localhost:3000` (or the port you configured)
   - Create your first admin account

## Configuration

### Basic Setup

The minimal configuration requires:
- `WEBUI_SECRET_KEY`: Generate with `openssl rand -hex 32`
- `CONTAINER_NAME_PREFIX`: Unique prefix for your containers
- `TZ`: Your timezone

### AI Model Providers

You can configure multiple AI providers:

#### Local Models (Ollama)
- Set `ENABLE_OLLAMA=1`
- Ollama will be available at `http://ollama:11434` internally
- Pull models: `docker compose exec ollama ollama pull llama2`

#### External APIs
- **OpenAI**: Set `OPENAI_API_KEY`
- **Anthropic**: Set `ANTHROPIC_API_KEY`

### GPU Support

For NVIDIA GPU support with Ollama:
1. Install NVIDIA Container Toolkit
2. Set `OLLAMA_GPU_COUNT=1` (or number of GPUs)
3. Update `OLLAMA_DEVICE_GPU=/dev/nvidia0:/dev/nvidia0`

### NPU Support (Rockchip RK3588/RK3576)

For **Rockchip RK3588/RK3576** boards (Orange Pi 5, etc.), you can use **RKLlama** instead of Ollama for NPU acceleration:

1. **Enable RKLlama** and disable Ollama:
   ```bash
   ENABLE_RKLLAMA=1
   ENABLE_OLLAMA=0
   ```

2. **Configure NPU device access** (uses DRI devices):
   ```bash
   RKLLAMA_DEVICE_DRI=/dev/dri:/dev/dri
   ```

3. **Verify NPU devices exist**:
   ```bash
   ls -la /dev/dri/
   # Should show: card0, card1, render128, render129
   ```

4. **Compatible with Armbian** on Orange Pi 5 Plus and other RK3588 boards
5. **Models**: Use `.rkllm` format models instead of standard Ollama models

### Database Options

#### SQLite (Default)
No additional configuration needed. Data stored in volume.

#### PostgreSQL
```bash
ENABLE_POSTGRES=1
POSTGRES_PASSWORD=your_secure_password
```

### Networks

The setup uses multiple isolated networks for security:
- `container_internet`: External internet access (cloudflared, open-webui for external APIs)
- `open_webui_app`: Internal app communication with no internet (cloudflared, open-webui)
- `open_webui_db`: Database network with no internet (open-webui, postgresql, redis)
- `open_webui_models`: Model communication with no internet (open-webui, ollama)
- `home_iot`: Optional home network integration

**Network Architecture:**
```
Internet ←→ container_internet ←→ cloudflared
                                     ↓
                              open_webui_app ←→ open-webui ←→ open_webui_db ←→ postgresql
                                                      ↓                    ↓
                                              open_webui_models ←→ ollama   redis
```

This architecture ensures:
- Databases are completely isolated from the internet
- Only necessary services have internet access
- Cloudflared can only access the app layer, not databases
- Models can be accessed by the app but are isolated from databases

## Service Management

### Start services:
```bash
docker compose up -d
```

### Stop services:
```bash
docker compose down
```

### View logs:
```bash
docker compose logs -f open-webui
docker compose logs -f ollama
```

### Update services:
```bash
docker compose pull
docker compose up -d
```

## Ollama Management

### Pull models:
```bash
docker compose exec ollama ollama pull llama2
docker compose exec ollama ollama pull codellama
docker compose exec ollama ollama pull mistral
```

### List models:
```bash
docker compose exec ollama ollama list
```

### Remove models:
```bash
docker compose exec ollama ollama rm model_name
```

## Security Considerations

1. **Change default passwords** in the `.env` file
2. **Generate a strong secret key** for `WEBUI_SECRET_KEY`
3. **Disable signup** (`ENABLE_SIGNUP=false`) after creating admin accounts
4. **Use Cloudflare Tunnel** for secure remote access instead of port forwarding
5. **Enable authentication** (`WEBUI_AUTH=true`)

## Troubleshooting

### Common Issues

1. **Permission errors**: Check `PUID` and `PGID` values
2. **Port conflicts**: Change `OPEN_WEBUI_PORT` in `.env`
3. **GPU not detected**: Ensure NVIDIA Container Toolkit is installed
4. **Network issues**: Verify external network exists: `docker network ls`

### Health Checks

Check service health:
```bash
docker compose ps
```

All services include health checks for monitoring.

### Logs

View specific service logs:
```bash
docker compose logs -f [service_name]
```

## Backup and Restore

### Backup volumes:
```bash
docker run --rm -v openwebui_open_webui_data:/data -v $(pwd):/backup alpine tar czf /backup/openwebui-backup.tar.gz -C /data .
```

### Restore volumes:
```bash
docker run --rm -v openwebui_open_webui_data:/data -v $(pwd):/backup alpine tar xzf /backup/openwebui-backup.tar.gz -C /data
```

## Advanced Configuration

### Custom Volume Mounts

Edit volume configuration in `.env`:
```bash
OPEN_WEBUI_DATA_BASE=/path/to/your/data
OPEN_WEBUI_DATA_VOLUME_TYPE=bind
```

### Resource Limits

Adjust memory limits:
```bash
OPEN_WEBUI_MEMORY_LIMIT=4G
```

### Network Configuration

For home network integration:
```bash
ENABLE_HOME_IOT_NETWORK=true
HOME_IOT_NETWORK=your_network_name
OPEN_WEBUI_IP=192.168.1.100
```

## Support

For issues and questions:
- Open-WebUI: https://github.com/open-webui/open-webui
- Ollama: https://github.com/ollama/ollama
- Docker Compose: https://docs.docker.com/compose/ 