# Central Monitoring Stack

Central server for distributed monitoring infrastructure with Grafana, Prometheus, Loki, and Caddy.

## Components

- **Caddy**: Automatic HTTPS/SSL, reverse proxy, Basic Auth for remote access
- **Grafana**: Visualization dashboard (port 3000, behind Caddy)
- **Prometheus**: Metrics storage with remote write receiver (2 years retention)
- **Loki**: Log aggregation with remote push receiver (3 months retention)
- **Promtail**: Optional log collector for central server containers

## Requirements

- Docker 20.10+
- Docker Compose 2.0+
- Domain name pointing to server (for SSL)
- Minimum 4GB RAM, 8GB recommended
- Minimum 150GB disk space

## Deployment

### Option 1: Portainer GitOps (Recommended)

1. Create this repository on GitHub
2. In Portainer, go to **Stacks** → **Add stack**
3. Select **Git Repository**
4. Configure:
   - **Name**: `monitoring-central`
   - **Repository URL**: `https://github.com/YOUR-USERNAME/monitoring-central`
   - **Branch**: `main`
   - **Compose path**: `docker-compose.yml`
5. Add environment variables (see below)
6. Enable **GitOps** for auto-updates
7. Deploy

### Option 2: Manual Deployment

```bash
# Clone repository
git clone https://github.com/YOUR-USERNAME/monitoring-central.git
cd monitoring-central

# Create .env file
cp .env.example .env
nano .env  # Edit with your values

# Deploy
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f
```

## Environment Variables

Required environment variables (set in Portainer or .env file):

### Basic Configuration

```env
# Domain for SSL certificate (required)
DOMAIN=monitoring.example.com

# Grafana admin credentials
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=your-secure-password-here

# Prometheus retention (default: 2y)
PROMETHEUS_RETENTION=2y
```

### For Remote VPS Access

```env
# Basic Authentication for remote VPS hosts
BASIC_AUTH_USER=remote
BASIC_AUTH_PASSWORD_HASH=$2a$12$...your-bcrypt-hash...
```

#### Generate Basic Auth Password Hash

```bash
# Install htpasswd
sudo apt-get install apache2-utils

# Generate bcrypt hash
htpasswd -nbB remote your-password

# Output will be: remote:$2a$12$...
# Copy only the hash part (after "remote:")
# Use this as BASIC_AUTH_PASSWORD_HASH value
```

### For DigitalOcean VPC

```env
# VPC private IP (optional, for documentation)
VPC_PRIVATE_IP=10.116.0.2
```

## Network Configuration

### DigitalOcean VPC Setup (Recommended)

If using DigitalOcean VPC for edge hosts:

1. Create VPC in DigitalOcean console
2. Assign central server and edge hosts to VPC
3. Edge hosts connect directly to Prometheus/Loki via private IPs
4. No authentication needed for VPC connections

See [../docs/VPC-SETUP.md](../docs/VPC-SETUP.md) for details.

### HTTPS + Basic Auth (For Remote VPS)

Remote VPS hosts connect via:
- **Prometheus**: `https://monitoring.example.com/prometheus/api/v1/write`
- **Loki**: `https://monitoring.example.com/loki/api/v1/push`

Caddy handles SSL and Basic Authentication.

## Accessing Grafana

After deployment:

1. Open `https://monitoring.example.com` in browser
2. Login with admin credentials from environment variables
3. Verify datasources are connected:
   - Configuration → Data sources
   - Both Prometheus and Loki should be green

## Importing Dashboards

### Pre-built Dashboards

Import these dashboard IDs from grafana.com:

- **15120** - Docker monitoring (cAdvisor + Node Exporter)
- **9628** - PostgreSQL monitoring
- **13639** - Loki logs dashboard

Steps:
1. Go to **Dashboards** → **Import**
2. Enter dashboard ID
3. Select Prometheus/Loki datasources
4. Click **Import**

### Custom Dashboards

Place JSON dashboard files in:
```
grafana/provisioning/dashboards/
```

They will be auto-loaded on Grafana restart.

## Firewall Configuration

### Required Ports

Open these ports on central server:

```bash
# For HTTPS/SSL (Grafana access)
sudo ufw allow 80/tcp    # HTTP (redirects to HTTPS)
sudo ufw allow 443/tcp   # HTTPS

# For SSH access
sudo ufw allow 22/tcp

# For Portainer (if used)
sudo ufw allow 9443/tcp

# Enable firewall
sudo ufw enable
```

### For VPC Setup

Allow Prometheus and Loki from VPC network:

```bash
# Allow from VPC IP range (example: 10.116.0.0/20)
sudo ufw allow from 10.116.0.0/20 to any port 9090  # Prometheus
sudo ufw allow from 10.116.0.0/20 to any port 3100  # Loki
```

Or use DigitalOcean Cloud Firewall for better management.

## Monitoring

### Check Container Health

```bash
# View all containers
docker ps

# Check logs
docker logs monitoring-caddy
docker logs monitoring-prometheus
docker logs monitoring-loki
docker logs monitoring-grafana

# Check resource usage
docker stats
```

### Check Disk Usage

```bash
# Prometheus data
du -sh /var/lib/docker/volumes/monitoring-central_prometheus-data

# Loki data
du -sh /var/lib/docker/volumes/monitoring-central_loki-data

# Grafana data
du -sh /var/lib/docker/volumes/monitoring-central_grafana-data
```

Set up disk usage alerts in Grafana to avoid running out of space.

### Verify Remote Write is Working

In Grafana:
1. Go to **Explore**
2. Select **Prometheus**
3. Run query:
   ```promql
   up{job="cadvisor"}
   ```
4. Should see metrics from all edge hosts

### Verify Logs are Flowing

In Grafana:
1. Go to **Explore**
2. Select **Loki**
3. Run query:
   ```logql
   {job="docker"}
   ```
4. Should see logs from all containers across all hosts

## Troubleshooting

### SSL Certificate Fails

```bash
# Check DNS
dig monitoring.example.com +short

# Check Caddy logs
docker logs monitoring-caddy

# Ensure ports 80/443 are open
sudo ufw status

# Check Let's Encrypt rate limits (max 5 failures per hour)
# Use staging if testing:
# Edit Caddyfile, add under global options:
# acme_ca https://acme-staging-v02.api.letsencrypt.org/directory
```

### Prometheus Not Receiving Data

```bash
# Check Prometheus logs
docker logs monitoring-prometheus

# Test remote write endpoint
curl -X POST http://localhost:9090/api/v1/write

# Verify remote write receiver is enabled
docker exec monitoring-prometheus ps aux | grep remote-write-receiver
```

### Loki Not Receiving Logs

```bash
# Check Loki logs
docker logs monitoring-loki

# Test push endpoint
curl http://localhost:3100/ready

# Check Loki config
docker exec monitoring-loki cat /etc/loki/loki-config.yaml
```

### Grafana Can't Connect to Datasources

```bash
# Verify all containers are on same network
docker network inspect monitoring-central_monitoring

# Check datasource URLs in Grafana
# Should be: http://prometheus:9090 and http://loki:3100
```

See [../docs/TROUBLESHOOTING.md](../docs/TROUBLESHOOTING.md) for more issues and solutions.

## Maintenance

### Update Stack

With GitOps:
- Push changes to GitHub
- Portainer auto-updates

Manual:
```bash
git pull
docker-compose pull
docker-compose up -d
```

### Backup Configuration

```bash
# Backup Grafana dashboards
docker exec monitoring-grafana grafana-cli admin export-dashboards > grafana-backup.json

# Backup volumes (stop containers first)
docker-compose down
tar -czf prometheus-backup.tar.gz /var/lib/docker/volumes/monitoring-central_prometheus-data
tar -czf loki-backup.tar.gz /var/lib/docker/volumes/monitoring-central_loki-data
tar -czf grafana-backup.tar.gz /var/lib/docker/volumes/monitoring-central_grafana-data
docker-compose up -d
```

### Rotate Logs

Loki and Prometheus handle retention automatically based on configuration.

To manually compact:
```bash
# Prometheus compaction (automatic, but can trigger manually)
curl -X POST http://localhost:9090/-/reload

# Loki compaction (runs automatically every 10 minutes)
# Check logs: docker logs monitoring-loki | grep compactor
```

## Security

1. **Change default passwords**: Update `GRAFANA_ADMIN_PASSWORD`
2. **Use strong Basic Auth**: For remote VPS access
3. **Keep software updated**: Regularly pull latest images
4. **Use VPC when possible**: Free and secure for DO hosts
5. **Enable firewall**: Only expose necessary ports
6. **Monitor for anomalies**: Set up alerts for unusual activity

## Performance Tuning

For large deployments (>10 hosts, >100 containers):

### Prometheus
```yaml
# Increase resources in docker-compose.yml
deploy:
  resources:
    limits:
      memory: 8G
      cpus: '4'
```

### Loki
```yaml
# In loki-config.yaml, increase:
limits_config:
  ingestion_rate_mb: 20
  ingestion_burst_size_mb: 40
```

### Server Sizing

- **5 hosts**: 4GB RAM, 2 vCPU, 150GB disk
- **10 hosts**: 8GB RAM, 4 vCPU, 300GB disk
- **20+ hosts**: Consider distributed Prometheus/Loki setup

## Getting Help

- Check [../docs/TROUBLESHOOTING.md](../docs/TROUBLESHOOTING.md)
- Review logs: `docker-compose logs`
- GitHub Issues: [Report problems here]
- Official docs:
  - [Prometheus](https://prometheus.io/docs/)
  - [Loki](https://grafana.com/docs/loki/)
  - [Grafana](https://grafana.com/docs/grafana/)
  - [Caddy](https://caddyserver.com/docs/)

## License

MIT License
