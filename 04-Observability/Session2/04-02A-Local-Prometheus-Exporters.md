# Local Prometheus and Exporters

## Overview

This lab shows the simplest Prometheus flow on a computer:

`Windows Exporter or Node Exporter -> Prometheus -> PromQL`

Complete only the section for your operating system. The commands use these tested release lines:

- Prometheus `3.13.1` LTS
- Node Exporter `1.11.1`
- Windows Exporter `0.31.8`

> Release versions change. If a link is no longer available, use the same steps with the current stable asset from the official download page.

---

## Option A: Windows

Run PowerShell as Administrator.

### Step 1: Install Windows Exporter

```powershell
$WindowsExporterVersion = "0.31.8"
$Msi = "windows_exporter-$WindowsExporterVersion-amd64.msi"
$Url = "https://github.com/prometheus-community/windows_exporter/releases/download/v$WindowsExporterVersion/$Msi"

Invoke-WebRequest -Uri $Url -OutFile $Msi
Start-Process msiexec.exe -ArgumentList "/i $Msi ENABLED_COLLECTORS=cpu,memory,logical_disk LISTEN_PORT=9182" -Wait
```

Validate the exporter:

```text
http://localhost:9182/metrics
```

Windows Exporter is installed as a Windows service. Confirm it is running:

```powershell
Get-Service windows_exporter
```

### Step 2: Install Prometheus

```powershell
$PrometheusVersion = "3.13.1"
$Archive = "prometheus-$PrometheusVersion.windows-amd64.zip"
$Url = "https://github.com/prometheus/prometheus/releases/download/v$PrometheusVersion/$Archive"

Invoke-WebRequest -Uri $Url -OutFile $Archive
New-Item -ItemType Directory -Path C:\Prometheus -Force
Expand-Archive -Path $Archive -DestinationPath C:\Prometheus -Force
```

Create this file:

```text
C:\Prometheus\prometheus-3.13.1.windows-amd64\prometheus.yml
```

Add:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: windows
    static_configs:
      - targets: ["localhost:9182"]
```

Start Prometheus:

```powershell
Set-Location C:\Prometheus\prometheus-3.13.1.windows-amd64
.\prometheus.exe --config.file=.\prometheus.yml
```

Keep this terminal open.

### Step 3: Query Windows metrics

Open `http://localhost:9090` and try:

```promql
up{job="windows"}
```

```promql
100 - (avg by (instance) (rate(windows_cpu_time_total{mode="idle"}[5m])) * 100)
```

```promql
100 - (100 * windows_memory_physical_free_bytes / windows_memory_physical_total_bytes)
```

---

## Option B: Linux

The commands below use an `amd64` Linux system.

### Step 1: Install and start Node Exporter

```bash
NODE_EXPORTER_VERSION="1.11.1"
wget "https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz"
tar xzf "node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz"
cd "node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64"
./node_exporter
```

Keep this terminal open and validate:

```text
http://localhost:9100/metrics
```

### Step 2: Install Prometheus

Open a second terminal:

```bash
PROMETHEUS_VERSION="3.13.1"
wget "https://github.com/prometheus/prometheus/releases/download/v${PROMETHEUS_VERSION}/prometheus-${PROMETHEUS_VERSION}.linux-amd64.tar.gz"
tar xzf "prometheus-${PROMETHEUS_VERSION}.linux-amd64.tar.gz"
sudo mv "prometheus-${PROMETHEUS_VERSION}.linux-amd64" /opt/prometheus
sudo nano /opt/prometheus/prometheus.yml
```

Use this configuration. Keep the job name `node`; Grafana dashboard `1860` expects this default name.

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: node
    static_configs:
      - targets: ["localhost:9100"]
```

Start Prometheus:

```bash
/opt/prometheus/prometheus --config.file=/opt/prometheus/prometheus.yml
```

### Step 3: Query Linux metrics

Open `http://localhost:9090` and try:

```promql
up{job="node"}
```

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

```promql
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))
```

---

## Option C: macOS

Homebrew selects the correct binary for Intel or Apple Silicon.

### Step 1: Install the exporter and Prometheus

```bash
brew install node_exporter prometheus
brew services start node_exporter
```

Validate Node Exporter:

```text
http://localhost:9100/metrics
```

### Step 2: Configure Prometheus

```bash
mkdir -p ~/prometheus-lab
nano ~/prometheus-lab/prometheus.yml
```

Add:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: node
    static_configs:
      - targets: ["localhost:9100"]
```

Start Prometheus:

```bash
prometheus --config.file="$HOME/prometheus-lab/prometheus.yml"
```

### Step 3: Query macOS metrics

Open `http://localhost:9090` and try:

```promql
up{job="node"}
```

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

Node Exporter exposes Darwin-specific memory series. Confirm their names first:

```promql
{__name__=~"node_memory_.*"}
```

With current Node Exporter releases, this estimates active plus wired memory:

```promql
100 * ((node_memory_active_bytes + node_memory_wired_bytes) / node_memory_total_bytes)
```

---

## Verification

In Prometheus, open **Status -> Target health**. Both targets should be `UP`:

- `prometheus`
- `windows` or `node`

Then run:

```promql
sum by (job) (up)
```