# Monitoring-HA

┌──────────────┐
         │   Telegraf   │
         │  (System +   │
         │  MySQL +     │
         │  Django)     │
         └──────┬───────┘
                │
         (metrics collection)
                │
         ┌──────▼───────┐
         │  InfluxDB    │  <-- time-series DB
         └──────┬───────┘
                │
        (visualize & alert)
                │
         ┌──────▼───────┐
         │   Grafana    │
         └──────────────┘
---

# Monitoring Stack with InfluxDB, Grafana, Telegraf, and Django

This project sets up a monitoring stack for a Django application using **InfluxDB v2**, **Grafana**, and **Telegraf**.
It collects system metrics, MySQL metrics, and Django application metrics via a Prometheus exporter, and visualizes them in Grafana dashboards.

---

## 🚀 Features

* **InfluxDB v2** – Time-series database to store metrics.
* **Grafana** – Visualization of metrics from InfluxDB.
* **Telegraf** – Metrics collector (system, MySQL, Django app).
* **Django Prometheus Exporter** – Exposes app-level metrics at `/metrics`.

---

## 📦 Prerequisites

* Ubuntu 22.04+ (tested on Jammy).
* Ansible installed on control node.
* Docker + Docker Compose.
* Python3 + pip.

---

## ⚙️ Setup Steps

### 1️⃣ Clone this repo

```bash
git clone <repo-url>
cd <repo>
```

---

### 2️⃣ Configure Environment Variables

Edit `.env` file for InfluxDB & Grafana:

```ini
# InfluxDB
DOCKER_INFLUXDB_INIT_MODE=setup
DOCKER_INFLUXDB_INIT_USERNAME=<username>
DOCKER_INFLUXDB_INIT_PASSWORD=<password>
DOCKER_INFLUXDB_INIT_ORG=<org-name>
DOCKER_INFLUXDB_INIT_BUCKET=<bucket-name>
DOCKER_INFLUXDB_INIT_ADMIN_TOKEN=<your-admin-token>
INFLUXDB_PORT=<port-number>   #8086

# Grafana
GF_SECURITY_ADMIN_USER=<username>
GF_SECURITY_ADMIN_PASSWORD=<password>
GRAFANA_PORT=<port-number>   #3000
```

---

### 3️⃣ Deploy InfluxDB & Grafana using Ansible

Run playbook (`site.yml`):

```bash
ansible-playbook -i inventory.ini site.yml
```

This will:

* Install Docker + Docker Compose.
* Create persistence directories.
* Deploy InfluxDB + Grafana stack.

Check services:

* Grafana → `http://<monitor-ip>:3000`
* InfluxDB → `http://<monitor-ip>:8086`
---

If any container is exited, check logs:

```bash
sudo docker logs <container>
```

Manage Containers (Optional)
```bash
docker-compose down -v   # Stops & removes containers, networks, and volumes (data loss if not mapped)
docker-compose up -d     # Starts containers in background
```

---

### 4️⃣ Install Telegraf (where our app is present/running)

```bash
curl -s https://repos.influxdata.com/influxdata-archive.key | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/influxdata.gpg
echo "deb [signed-by=/etc/apt/trusted.gpg.d/influxdata.gpg] https://repos.influxdata.com/ubuntu jammy stable" | sudo tee /etc/apt/sources.list.d/influxdata.list
sudo apt update
sudo apt -y install telegraf
```

Configure `/etc/telegraf/telegraf.conf`:

* System metrics (`cpu`, `mem`, `disk`, `net`, `system`).
  
* Django app metrics via Prometheus:
  [inputs.procstat]


  ```toml
  [[inputs.prometheus]]
  urls = ["http://<django-app-ip>:7000/metrics"]
  metric_version = 2
  ```
* MySQL metrics:

  ```toml
  [[inputs.mysql]]
  servers = ["root:password@tcp(127.0.0.1:3306)/"]
  # Replace root:password with your MySQL username and password
  # If your MySQL is running on another VM, replace 127.0.0.1 with its IP
  ```

Point output to InfluxDB:

```toml
[[outputs.influxdb_v2]]
  urls = ["http://<monitor-ip>:8086"]
  token = "<your-admin-token>"
  organization = "<org-name>"
  bucket = "<bucket-name>"
```

Enable & start Telegraf:

```bash
sudo systemctl enable telegraf
sudo systemctl start telegraf
```

---

### 5️⃣ Configure Django for Prometheus

Install dependencies:

```bash
sudo apt install python3-pip
python3 -m venv venv
source venv/bin/activate
pip install django-prometheus requests
```

Edit `settings.py`:

```python
INSTALLED_APPS = [
    ...
    'django_prometheus',
]

MIDDLEWARE = [
    'django_prometheus.middleware.PrometheusBeforeMiddleware',
    ...
    'django_prometheus.middleware.PrometheusAfterMiddleware',
]
```

Edit `urls.py`:

```python
from django.urls import path, include

urlpatterns = [
    ...
    path('', include('django_prometheus.urls')),
]
```

Run Django app:

```bash
python manage.py runserver 0.0.0.0:7000
```

Check app:
`http://<django-app-ip>:7000`
Metrics exposed at:
`http://<django-app-ip>:7000/metrics`

---

## 📊 Verification

1. **Django App** → `http://<django-app-ip>:7000`
2. **Grafana UI** → `http://<monitor-ip>:3000`

   * Login with admin/admin.
   * Add InfluxDB data source (bucket: `telegraf`).
3. **InfluxDB UI** → `http://<monitor-ip>:8086`

   * Login with admin credentials.
   * Verify data ingestion from Telegraf.

---

## 📌 Notes

* Update MySQL credentials in `telegraf.conf`.
* Replace `<monitor-ip>` and `<django-app-ip>` with actual VM IPs.
* Use dashboards in Grafana to visualize metrics (CPU, RAM, MySQL, Django).

---
Copy the [telegraf.conf](./telegraf.conf) to /etc/telegraf/telegraf.conf
make sure to update:
1. MYSQL credentials
2. Grafana credentials
3. Monitoring server IP(influxdb URL)
