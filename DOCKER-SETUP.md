# Docker Compose Setup for MCP Pipe

This setup runs `mcp_pipe.py` as a daemon using Docker Compose with automatic boot start.

## Quick Start

```bash
# Build and start
docker compose up -d

# Check status
docker compose ps

# View logs
docker compose logs -f

# Stop
docker compose down
```

## Files Created

- **Dockerfile**: Python 3.13-slim container with all dependencies
- **docker-compose.yml**: Service configuration with auto-restart
- **.env**: Contains MCP_ENDPOINT token (not committed to git)
- **.env.example**: Template for environment variables
- **mcp-pipe-compose.service**: Systemd service for boot auto-start

## Configuration

### Environment Variables (.env)

```bash
MCP_ENDPOINT=wss://api.xiaozhi.me/mcp/?token=YOUR_TOKEN
```

### MCP Config (mcp_config.json)

Configure MCP servers in `mcp_config.json`. The container mounts this file as read-only, so you can update it without rebuilding:

```json
{
  "mcpServers": {
    "vocab": {
      "type": "http",
      "url": "https://your-mcp-endpoint.com/mcp",
      "disabled": false
    }
  }
}
```

After updating `mcp_config.json`, restart the container:
```bash
docker compose restart
```

### AWS Credentials Configuration

The container is configured to work with AWS services using EC2 instance profile credentials (recommended for EC2 deployments).

#### EC2 Instance Profile (Recommended)

When running on an EC2 instance:
1. The container uses `network_mode: "host"` to access the EC2 metadata service at `169.254.169.254`
2. AWS credentials are automatically retrieved from the instance profile
3. No additional configuration needed

Verify credentials are working:
```bash
# Check if container can access AWS
docker compose exec mcp-pipe python -c "import boto3; print(boto3.client('sts').get_caller_identity())"
```

#### Local AWS Credentials (Alternative)

If not using EC2 instance profile, you can mount local AWS credentials:

1. Uncomment the AWS credentials volume mount in `docker-compose.yml`:
```yaml
volumes:
  - ${HOME}/.aws:/home/mcpuser/.aws:ro
```

2. Ensure your `~/.aws/credentials` and `~/.aws/config` files are properly configured

3. Restart the container:
```bash
docker compose down
docker compose up -d
```

#### AWS Region Configuration

Set AWS region via environment variables in `.env` file:
```bash
AWS_REGION=us-west-2
AWS_DEFAULT_REGION=us-west-2
```

Or export them before starting:
```bash
export AWS_REGION=us-west-2
docker compose up -d
```

## Management Commands

### Container Management
```bash
# Start container
docker compose up -d

# Stop container
docker compose down

# Restart container
docker compose restart

# View logs (follow)
docker compose logs -f

# View last 50 lines
docker compose logs --tail=50

# Check container health
docker compose ps
```

### Systemd Service (Boot Auto-Start)

The systemd service is configured to start Docker Compose automatically at boot.

```bash
# Check systemd service status
sudo systemctl status mcp-pipe-compose.service

# Manually start via systemd
sudo systemctl start mcp-pipe-compose.service

# Stop via systemd
sudo systemctl stop mcp-pipe-compose.service

# Disable auto-start at boot
sudo systemctl disable mcp-pipe-compose.service

# Enable auto-start at boot
sudo systemctl enable mcp-pipe-compose.service
```

### Rebuilding After Code Changes

If you modify `mcp_pipe.py` or `requirements.txt`:

```bash
# Stop, rebuild, and start
docker compose down
docker compose build
docker compose up -d
```

Or in one command:
```bash
docker compose up -d --build
```

## Logs & Monitoring

### Docker Logs
```bash
# Follow logs
docker compose logs -f

# Last 100 lines
docker compose logs --tail=100

# Logs with timestamps
docker compose logs -f --timestamps

# Logs for specific time
docker compose logs --since 2025-11-01T16:00:00
```

### Log Rotation

Logs are automatically rotated:
- Max size: 10MB per file
- Max files: 3
- Total log storage: ~30MB

## Troubleshooting

### Container won't start
```bash
# Check logs for errors
docker compose logs

# Rebuild from scratch
docker compose down
docker compose build --no-cache
docker compose up -d
```

### WebSocket connection issues
```bash
# Check if MCP_ENDPOINT is set correctly
docker compose exec mcp-pipe env | grep MCP_ENDPOINT

# Check mcp_config.json is mounted
docker compose exec mcp-pipe cat /app/mcp_config.json
```

### Health check failing
```bash
# Check container logs
docker compose logs --tail=50

# Inspect health check status
docker inspect mcp-pipe-bridge | grep -A 10 Health
```

### AWS Credentials Issues

```bash
# Check if uvx is installed and accessible
docker compose exec mcp-pipe which uvx
docker compose exec mcp-pipe uvx --version

# Test AWS credentials from inside container
docker compose exec mcp-pipe python -c "import boto3; print(boto3.client('sts').get_caller_identity())"

# Check AWS environment variables
docker compose exec mcp-pipe env | grep AWS

# Test EC2 metadata service access (if on EC2)
docker compose exec mcp-pipe curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/

# If using local credentials, check if mounted correctly
docker compose exec mcp-pipe ls -la /home/mcpuser/.aws/
```

## Security Notes

- `.env` file is excluded from git (contains sensitive token)
- Container runs as non-root user (mcpuser, UID 1000)
- `mcp_config.json` is mounted read-only
- Logs are automatically rotated to prevent disk space issues

## Boot Auto-Start

The systemd service `mcp-pipe-compose.service` ensures the container starts automatically when the system boots:

1. **Service installed**: `/etc/systemd/system/mcp-pipe-compose.service`
2. **Enabled at boot**: Starts after Docker and network are available
3. **Manages lifecycle**: Starts with `docker compose up -d`, stops with `docker compose down`

To test boot auto-start without rebooting:
```bash
# Stop the container
docker compose down

# Start via systemd (simulates boot)
sudo systemctl start mcp-pipe-compose.service

# Verify it started
docker compose ps
```
