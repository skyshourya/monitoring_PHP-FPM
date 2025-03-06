# monitoring_PHP-FPM

# PHP-FPM Monitoring with Prometheus, Grafana, and Alertmanager

## Overview
This setup provides real-time monitoring for a PHP-FPM server using `php-fpm_exporter`, Prometheus, and Grafana. It also includes Alertmanager for sending notifications to Slack when PHP-FPM or the EC2 instance goes down.

## Components Used
- **Prometheus**: Collects and stores metrics.
- **PHP-FPM Exporter**: Exposes PHP-FPM metrics to Prometheus.
- **Grafana**: Visualizes metrics with dashboards.
- **Alertmanager**: Handles alerts and sends notifications to Slack.

## Prerequisites
- An Ubuntu-based EC2 instance.
- Docker and Docker Compose installed.
- PHP-FPM installed and running.

## Setup Instructions

### 1. Install Dependencies
```bash
sudo apt update
sudo apt install -y docker.io docker-compose
```

### 2. Docker Compose Configuration on the host where PHP-FPM is running
Create a `docker-compose.yml` file with the following content:
```yaml
version: '3.8'
services:
  php-fpm_exporter:
    image: hipages/php-fpm_exporter
    container_name: php-fpm_exporter
    environment:
      - PHP_FPM_SCRAPE_URI=tcp://<INSTANCE_IP>:9000/status,tcp://<INSTANCE_IP>:9001/status
    ports:
      - "9253:9253"  # Exposing metrics on port 9253
    restart: always

```

### 3. Configure PHP-FPM for Monitoring
Modify the `www.conf` file to enable status monitoring, located at `/etc/php/{php_version}/fpm/pool.d/www.conf` :
```ini
user = www-data
group = www-data
pm.status_path = /status
listen = 0.0.0.0:9000  # 0.0.0.0 for all IPs and xx.xx.xx.xx for a specific IP.
listen.owner = www-data
listen.group = www-data
pm.process_idle_timeout = 10s;

```
Restart PHP-FPM:
```bash
sudo systemctl restart php-fpm
```

### 4. Docker Compose Configuration on the host where Prometheus, Grafana, and Alertmanager will be running.
Create a `docker-compose.yml` file with the following content:
```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alert.rules.yml:/etc/prometheus/alert.rules.yml
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
    ports:
      - "9090:9090"
    depends_on:
      - alertmanager

  alertmanager:
    image: prom/alertmanager
    container_name: alertmanager
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    command:
      - "--config.file=/etc/alertmanager/alertmanager.yml"
    ports:
      - "9093:9093"

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - prometheus
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
    grafana-data:
```

### 5. Start the Monitoring Stack
```bash
docker-compose up -d
```

### 6. Verify Prometheus Targets
Open Prometheus in a browser:
```
http://<EC2-IP>:9090
```
Navigate to `Status -> Targets` and check if `php-fpm-exporter` is up.

### 7. Access Grafana
```
http://<EC2-IP>:3000
```
Login using:
- **Username:** admin
- **Password:** admin

Add Prometheus as a data source and import a PHP-FPM monitoring dashboard.

### 8. Alerting Setup
Alerts are defined in `alert.rules.yml`:
```yaml
  - alert: PHPFPMDown
    expr: php_fpm_up == 0
    for: 20s
    labels:
      severity: critical
    annotations:
      summary: "PHP-FPM is down on {{ $labels.instance }}"
      description: "PHP-FPM process on {{ $labels.instance }} is unreachable for over 20 seconds."
```
Slack notifications are configured in `alertmanager.yml`:
```yaml
receivers:
  - name: 'slack-notifications'
    slack_configs:
      - send_resolved: true
        channel: '#NAME'
        api_url: 'https://hooks.slack.com/services/YOUR_SLACK_WEBHOOK_URL'
```

### 9. Test Alerts
Stop the PHP-FPM container and check if Slack receives an alert:
```bash
docker stop php-fpm-exporter
```

## Conclusion
This setup ensures that your PHP-FPM server is continuously monitored, and any downtime triggers an alert on Slack. You can extend this setup to monitor additional services as needed.

---

**Maintainer:** Shourya Yadav

