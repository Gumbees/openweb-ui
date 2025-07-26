# Open-WebUI Docker Compose Setup

This Docker Compose configuration provides a complete Open-WebUI stack with Ollama for local AI model hosting, following the same architecture patterns as enterprise-grade applications.

## Features

- **Open-WebUI**: Self-hosted web interface for AI chat
- **Ollama**: Local AI model hosting (optional but recommended)
- **PostgreSQL**: Database for persistent data (optional - uses SQLite by default)
- **Redis**: Caching layer (optional but recommended)
- **Cloudflare Tunnel**: Secure remote access (optional)
- **Network Isolation**: Secure internal networks
- **CPU Inference**: Ollama runs on CPU (no GPU required)
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

3. **Start the services:**
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

#### Local Models (Ollama CPU)
- Set `ENABLE_OLLAMA=1`
- Ollama will be available at `http://ollama:11434` internally
- Pull models: `docker compose exec ollama ollama pull llama2`
- **Note**: CPU version - models will run on CPU (slower but no GPU required)

#### External APIs
- **OpenAI**: Set `OPENAI_API_KEY`
- **Anthropic**: Set `ANTHROPIC_API_KEY`

### Resource Configuration

For CPU-only Ollama deployment:
1. Set `ENABLE_OLLAMA=1`
2. Adjust `OLLAMA_MEMORY_LIMIT=4G` (or higher for larger models)
3. **Note**: CPU inference is slower but requires no special hardware



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
- `open_webui_app`: App network with internet access (open-webui, ollama, cloudflared)
- `open_webui_db`: Database network with no internet (open-webui, postgresql, redis)
- `open_webui_models`: Model communication with no internet (open-webui, ollama)

**Network Architecture:**
```
Internet ←→ open_webui_app ←→ open-webui ←→ open_webui_db ←→ postgresql
                ↓                   ↓                    ↓
           cloudflared       open_webui_models ←→ ollama   redis
```

This architecture ensures:
- Databases are completely isolated from the internet
- App layer has internet access for external APIs and tunneling
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