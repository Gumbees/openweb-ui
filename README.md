# Open-WebUI Docker Compose Setup

This Docker Compose configuration provides a complete Open-WebUI stack with optional standalone Ollama deployment for local AI model hosting, following the same architecture patterns as enterprise-grade applications.

## Features

- **Open-WebUI**: Self-hosted web interface for AI chat
- **Standalone Ollama (AMD ROCm/NVIDIA CUDA)**: Separate GPU-accelerated AI model hosting
- **PostgreSQL**: Database for persistent data (optional - uses SQLite by default)
- **Redis**: Caching layer (optional but recommended)
- **Cloudflare Tunnel**: Secure remote access (optional)
- **Network Isolation**: Secure internal networks
- **Standalone Architecture**: Ollama runs as a separate service for better resource management
- **GPU Optimization**: Both AMD ROCm and NVIDIA CUDA support

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

3. **Initialize Docker Swarm (required for this stack):**
   ```bash
   docker swarm init
   ```

4. **Create the external Traefik network (overlay):**
   ```bash
   docker network create --driver=overlay --attachable traefik_public
   ```

5. **Deploy the stack to Swarm:**
   ```bash
   docker stack deploy -c docker-compose.yml openwebui
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

### Authentication Options

#### Local Authentication (Default)
Standard username/password authentication with local user accounts.

#### Entra ID SSO (Azure AD)
Configure OAuth/OpenID Connect for single sign-on:

1. **Register an App in Entra ID:**
   - Go to Azure Portal → Entra ID → App registrations
   - Create a new registration
   - Set redirect URI to: `https://your-domain.com/oauth/callback`
   - Note the Application (client) ID and create a client secret

2. **Configure OAuth settings in `.env`:**
   ```bash
   OAUTH_CLIENT_ID=your-application-client-id
   OAUTH_CLIENT_SECRET=your-client-secret
   OPENID_PROVIDER_URL=https://login.microsoftonline.com/your-tenant-id/v2.0
   OAUTH_SCOPES=openid email profile
   OAUTH_PROVIDER_NAME=Entra ID
   ```

3. **Optional settings:**
   ```bash
   OAUTH_USERNAME_CLAIM=preferred_username  # or 'email'
   OAUTH_EMAIL_CLAIM=email
   OAUTH_MERGE_ACCOUNTS_BY_EMAIL=false
   ```

### AI Model Providers

You can configure multiple AI providers:

#### Standalone Ollama (GPU-Accelerated)
- Use the separate `docker-compose-ollama.yml` file for GPU-accelerated Ollama
- Supports both AMD ROCm and NVIDIA CUDA
- Network integration with main Open-WebUI stack
- Pull models: `docker compose -f docker-compose-ollama.yml exec ollama ollama pull llama2`

#### External APIs
- **OpenAI**: Set `OPENAI_API_KEY`
- **Anthropic**: Set `ANTHROPIC_API_KEY`

### Standalone Ollama Deployment

For GPU-accelerated Ollama deployment:
1. Use the separate `docker-compose-ollama.yml` file
2. Configure GPU settings in `.env` file
3. Create required external networks before deployment
4. **Note**: GPU acceleration requires appropriate drivers (AMD ROCm or NVIDIA CUDA)

See the Ollama standalone deployment section below for detailed instructions.

### Database Options

#### SQLite (Default)
No additional configuration needed. Data stored in volume.

#### PostgreSQL
```bash
ENABLE_POSTGRES=1
POSTGRES_PASSWORD=your_secure_password
```

### Networks

This stack is Swarm-ready and uses overlay networks:
- `stack` (overlay, attachable): Internal stack network for all services
- `traefik_public` (external overlay): For Traefik to route public/private domains

Notes:
- Services expecting proxy traffic (e.g., `open-webui`) are attached to both `stack` and `traefik_public`.
- All other services are attached only to `stack`.

## Service Management

### Deploy/Update the stack:
```bash
docker stack deploy -c docker-compose.yml openwebui
```

### Stop services:
```bash
docker stack rm openwebui
```

### View logs:
```bash
docker service logs -f openwebui_open-webui | cat
```

### Update services:
```bash
docker stack deploy -c docker-compose.yml openwebui
```

## Standalone Ollama Deployment

### Prerequisites

Create required networks before deployment:

```bash
# Create the Ollama internal network
docker network create --driver overlay --attachable ollama_network

# Create the shared Open-WebUI stack network
docker network create --driver overlay --attachable openwebui_stack
```

### Deployment

1. Deploy the standalone Ollama service:
   ```bash
   docker-compose -f docker-compose-ollama.yml up -d
   ```

2. Deploy the main Open-WebUI stack:
   ```bash
   docker-compose up -d
   ```

For detailed GPU configuration and troubleshooting, refer to the environment variables in `.env.example`.

## Ollama Management

### Pull models:
```bash
docker compose -f docker-compose-ollama.yml exec ollama ollama pull llama2
docker compose -f docker-compose-ollama.yml exec ollama ollama pull codellama
docker compose -f docker-compose-ollama.yml exec ollama ollama pull mistral
```

### List models:
```bash
docker compose -f docker-compose-ollama.yml exec ollama ollama list
```

### Remove models:
```bash
docker compose -f docker-compose-ollama.yml exec ollama ollama rm model_name
```

### Test Ollama API:
```bash
curl http://localhost:11434/api/tags
```

## Security Considerations

1. **Change default passwords** in the `.env` file
2. **Generate a strong secret key** for `WEBUI_SECRET_KEY`
3. **Disable signup** (`ENABLE_SIGNUP=false`) after creating admin accounts or when using SSO
4. **Use Cloudflare Tunnel** for secure remote access instead of port forwarding
5. **Enable authentication** (`WEBUI_AUTH=true`)
6. **SSO Security**:
   - Keep OAuth client secrets secure and rotate them regularly
   - Use HTTPS for all OAuth redirect URIs
   - Configure appropriate scopes in Entra ID (minimum required permissions)
   - Consider setting `OAUTH_MERGE_ACCOUNTS_BY_EMAIL=true` if users might have both local and SSO accounts

## Troubleshooting

### Common Issues

1. **Permission errors**: Check volume permissions and ensure container can write to data directories
2. **Port conflicts**: Change `OPEN_WEBUI_PORT` in `.env`
3. **Memory issues**: Increase `OLLAMA_MEMORY_LIMIT` for larger models
4. **Network issues**: Check Docker network connectivity
5. **OAuth/SSO Issues**:
   - Verify redirect URI matches exactly (including protocol and path)
   - Check that client secret hasn't expired
   - Ensure `OPENID_PROVIDER_URL` includes correct tenant ID
   - Verify required API permissions are granted in Entra ID
   - Check logs for specific OAuth error messages

### Health Checks

Check service health:
```bash
docker stack ps openwebui
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

All volumes now use Docker's default local storage. Data is stored in Docker-managed volumes under `/var/lib/docker/volumes/` (on most systems). If you need custom mount points, you can modify the volume definitions directly in the `docker-compose.yml` file.

### Resource Limits

Adjust memory limits:
```bash
OPEN_WEBUI_MEMORY_LIMIT=4G
```



## Support

For issues and questions:
- Open-WebUI: https://github.com/open-webui/open-webui
- Ollama: https://github.com/ollama/ollama
- Docker Compose: https://docs.docker.com/compose/ 