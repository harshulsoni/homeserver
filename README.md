# Homeserver Stack

A complete Docker Swarm-based homeserver setup with VPN, DNS/AdBlocking, reverse proxy, monitoring, logging, and automated backups.

## Stack Components

### Network & VPN
- **WireGuard** (ghcr.io/wg-easy/wg-easy:15) - VPN server for secure remote access
  - Ports: 51820 (UDP), 51821 (TCP)

### DNS & Network Services
- **AdGuard Home** - DNS filtering and ad-blocking
  - Ports: 53 (DNS), 3000 (Web UI), 1180 (HTTP), 1153 (DNS TCP)

### Reverse Proxy & SSL
- **Nginx Proxy Manager** - Reverse proxy with Let's Encrypt SSL support
  - Ports: 80 (HTTP), 81 (Admin), 443 (HTTPS)

### Media Services
- **MediaFlow Proxy** - Media streaming proxy
  - Port: 8888

### Monitoring & Observability
- **Prometheus** - Metrics collection and storage
  - Port: 9090
  - Data retention: 7 days
- **Grafana** - Metrics visualization and dashboards
  - Port: 3100
- **cAdvisor** - Container metrics collection
- **Node Exporter** - System-level metrics
- **AlertManager** - Alert routing and management
  - Port: 9093

### Logging
- **Loki** - Log aggregation and storage
  - Port: 3101
- **Promtail** - Log collector for system and container logs

### Uptime Monitoring
- **Uptime Kuma** - Service uptime monitoring and status pages
  - Port: 3001

### Backup & Storage
- **Docker Volume Backup** - Automated daily backups to Dropbox
  - Schedule: 2:00 AM daily
  - Retention: 180 days

## Prerequisites

- Docker Swarm cluster initialized (`docker swarm init`)
- Ubuntu 24.04.3 LTS or compatible Linux
- Dropbox account (optional, for automated backups)

## Setup Instructions

### 1. Initialize Docker Swarm

If not already initialized:

```bash
docker swarm init
```

### 2. Create Environment File

Create a `.env` file in the project root with the following variables:

```env
# WireGuard Configuration
WIREGUARD_HOST_NAME=your-domain.com
WIREGUARD_INIT_USERNAME=admin
WIREGUARD_INIT_PASSWORD=your-secure-password

# MediaFlow Proxy
MEDIAFLOW_PROXY_API_PASSWORD=your-mediaflow-password

# Backup Configuration
DAILY_BACKUP_FILENAME=homeserver-backup-%Y-%m-%d.tar.gz

# Dropbox Configuration (optional, for backups)
DROPBOX_APP_KEY=your-app-key
DROPBOX_APP_SECRET=your-app-secret
DROPBOX_REFRESH_TOKEN=your-refresh-token
```

### 3. Create Docker Configs

Before deploying, create the required configs from YAML files:

**Option A: Via CLI**

```bash
# Prometheus config
docker config create prom_config prometheus.yml

# AlertManager config
docker config create alert_config alert_config.yml

# Promtail config
docker config create promtail_config promtail_config.yml
```

**Option B: Via Portainer**

1. Navigate to **Configs** in Portainer
2. Click **Create Config**
3. For each config file:
   - **Name**: `prom_config`, `alert_config`, `promtail_config`
   - **Content**: Copy and paste the contents of the corresponding YAML file
   - Click **Create**

### 4. Deploy the Stack

**Option A: Via Portainer (Recommended)**

1. In Portainer, go to **Stacks**
2. Click **Add Stack**
3. Choose **Docker Swarm** mode
4. Set stack name to `homeserver`
5. Choose upload method:
   - **Upload**: Select the `docker-compose.yml` file from your system
   - **Code Editor**: Copy and paste the contents
6. Load environment variables from `.env`:
   - In the Web Editor, scroll to **Environment** section
   - Add your variables (or upload `.env` file if supported)
7. Click **Deploy the Stack**
8. Monitor deployment progress in the Stack logs

**Option B: Via CLI**

```bash
docker stack deploy -c docker-compose.yml homeserver
```

### 5. Verify Deployment

**Via Portainer:**
1. Go to **Stacks** and select `homeserver`
2. View services and containers in the stack details
3. Check **Logs** for each service

**Via CLI:**

```bash
# Check service status
docker stack services homeserver

# View stack logs
docker stack ps homeserver

# Check individual service logs
docker service logs homeserver_<service_name>
```

## Service Access

### Portainer Management

**Access Portainer Console:**
- URL: `http://<your-host-ip>:9000` (or configured port)
- Login with your Portainer credentials
- Navigate to **Stacks** → `homeserver` to manage the deployment

**Key Portainer Features:**
- View real-time logs for each service
- Update services and pull latest images
- Monitor container health and resource usage
- Manage volumes and networks
- Configure environment variables
- Scale services up/down

### Local Network (within Docker Swarm)

| Service | URL | Port |
|---------|-----|------|
| Prometheus | http://localhost:9090 | 9090 |
| Grafana | http://localhost:3100 | 3100 |
| AlertManager | http://localhost:9093 | 9093 |
| Loki | http://localhost:3101 | 3101 |
| Uptime Kuma | http://localhost:3001 | 3001 |
| AdGuard Home | http://localhost:3000 | 3000 |
| Nginx Admin | http://localhost:81 | 81 |
| MediaFlow Proxy | http://localhost:8888 | 8888 |

### Remote Access

- **WireGuard VPN**: Connect via `your-domain.com:51820`
- **Nginx Reverse Proxy**: Traffic on ports 80, 443 is managed by Nginx

## Configuration Files

### `prometheus.yml`
Defines Prometheus scrape jobs for:
- cAdvisor (container metrics on port 8080)
- Node Exporter (system metrics on port 9100)

### `alert_config.yml`
AlertManager configuration for alert routing. Currently configured for log-only receivers.

### `promtail_config.yml`
Log collection configuration:
- Collects system logs from `/var/log/`
- Collects container logs via Docker socket
- Sends logs to Loki on port 3100

## Managing Volumes

The stack uses named Docker volumes for persistence:

```bash
# List all volumes
docker volume ls | grep homeserver

# Inspect a volume
docker volume inspect homeserver_prometheus_data

# View volume location on host
docker volume inspect -f '{{.Mountpoint}}' homeserver_prometheus_data
```

### Volumes Backed Up
- wireguard
- adguard_work
- adguard_conf
- nginx_data
- nginx_letsencrypt
- uptime-kuma
- grafana_data
- alertmanager_data

**Note:** loki_data (logs) is excluded to save backup storage space as logs are ephemeral.

## Backup Configuration

The backup service runs daily at 2:00 AM and backs up to Dropbox. To configure Dropbox:

1. Create a Dropbox App at https://www.dropbox.com/developers/apps
2. Generate refresh token
3. Update `.env` with:
   - `DROPBOX_APP_KEY`
   - `DROPBOX_APP_SECRET`
   - `DROPBOX_REFRESH_TOKEN`

Local backup copies are stored in `backup_data` volume under `/archive`.

## Common Tasks

### Update a Service

**Via Portainer:**
1. Go to **Stacks** → `homeserver`
2. Select the service to update
3. Click **Update Service**
4. Check "Pull latest image" option
5. Click **Update**

**Via CLI:**

```bash
# Pull latest image and restart service
docker service update --image <new-image:tag> homeserver_<service_name>
```

### View Service Logs

**Via Portainer:**
1. Go to **Stacks** → `homeserver`
2. Select the service
3. Click **Logs** tab to view real-time logs

**Via CLI:**

```bash
# Real-time logs
docker service logs -f homeserver_<service_name>

# Last 100 lines
docker service logs --tail 100 homeserver_<service_name>
```

### Restart All Services

**Via Portainer:**
1. Go to **Stacks** → `homeserver`
2. Click **Stop Stack** (or Right-click → Stop)
3. Click **Start Stack** to restart

**Via CLI:**

```bash
docker stack down homeserver
docker stack deploy -c docker-compose.yml homeserver
```

### Execute Command in Container

**Via Portainer:**
1. Go to **Containers**
2. Find the container (e.g., `homeserver_prometheus.1.xxxxx`)
3. Click the container
4. Go to **Console** tab
5. Execute commands

**Via CLI:**

```bash
docker exec <container_id> <command>
```

## Networking

The stack uses an overlay network (`server-network`) for service-to-service communication in Docker Swarm. All containers can communicate via service names (e.g., `http://prometheus:9090`).

## Monitoring Setup

### Grafana Dashboards

1. Access Grafana at `http://localhost:3100`
2. Default credentials: `admin` / `admin` (change immediately in production)
3. Add Prometheus data source:
   - URL: `http://prometheus:9090`
4. Import dashboards or create custom ones for:
   - Container metrics from cAdvisor
   - Node metrics from Node Exporter

### Alerting

Configure alerts in Prometheus and route them through AlertManager. Currently, the alert config is minimal - customize in `alert_config.yml`.

## Troubleshooting

### Service won't start

**Via Portainer:**
1. Go to **Stacks** → `homeserver`
2. Click on the failing service
3. Check **Logs** tab for error messages
4. Check **Environment** settings are correct

**Via CLI:**

```bash
# Check service logs
docker stack ps homeserver

# Verify configs exist
docker config ls

# Check Docker Swarm status
docker node ls
docker network ls
```

### Config not found error

**Via Portainer:**
1. Go to **Configs**
2. Verify all three configs exist: `prom_config`, `alert_config`, `promtail_config`
3. Create missing configs manually with the correct YAML content

**Via CLI:**

```bash
docker config create prom_config prometheus.yml
```

### Stack fails to deploy in Portainer

1. **Check Docker Swarm is active:**
   - In Portainer, go to **Cluster** and verify Swarm mode is enabled
2. **Verify all prerequisites:**
   - All environment variables in `.env` are set correctly
   - All Docker configs exist (checked via **Configs** tab)
   - Sufficient disk space available
3. **Check manager node:**
   - Go to **Nodes** and ensure at least one manager node is healthy
4. **Review stack logs:**
   - Click the stack and check the deployment logs for specific errors

### Volume issues

**Via Portainer:**
1. Go to **Volumes**
2. Search for `homeserver_*` volumes
3. Click a volume to inspect details and mountpoint
4. Check available disk space

**Via CLI:**

```bash
# Check volume health
docker volume ls -f name=homeserver

# Inspect volume
docker volume inspect homeserver_prometheus_data
```

### Port conflicts

**Via Portainer:**
1. Go to **Networks** → `server-network`
2. Check exposed ports
3. Verify no other containers use the same ports

**Via CLI:**

```bash
sudo netstat -tulpn | grep LISTEN
```

## Production Considerations

1. **Secrets**: Using `.env` is not recommended for production. Use Docker secrets instead:
   ```bash
   docker secret create wireguard_password -
   ```

2. **Resource Limits**: Add memory and CPU limits to services in `docker-compose.yml`:
   ```yaml
   deploy:
     resources:
       limits:
         cpus: '0.5'
         memory: 512M
   ```

3. **Backup Verification**: Regularly test backup restoration.

4. **SSL Certificates**: Use Nginx Proxy Manager to manage SSL certificates via Let's Encrypt.

5. **Log Rotation**: Configure log rotation for system logs before sending to Loki.

## License

[Specify your license here]

## Support & Contributions

For issues or contributions, please refer to the repository guidelines.