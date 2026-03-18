# 🚢 Parus 8 Docker Deployment (Experimental) 🐳

This repository contains an experimental, highly scalable Docker Compose configuration for deploying multiple **Parus 8 Web Client** environments using a single entry point.

## 🌟 Key Features

*   **Dynamic Routing:** Access multiple instances via URL paths (e.g., `/inst1`, `/inst2`) on a single port (80).
*   **Shared Resources:** Single **Redis** and **Angie** (Nginx-compatible) instance for all environments to minimize RAM usage.
*   **Universal DNS Detection:** Automatically detects nameservers for both Docker (`127.0.0.11`) and Podman (`10.88.0.1`).
*   **Instance Isolation:** Start, stop, or update specific deployments without affecting others.
*   **Optimized for Parus 8:** Pre-configured with Gzip, long timeouts, and buffer settings for Oracle-based reporting.

## 🏗️ Architecture Overview

The setup splits infrastructure from application logic to prevent container name conflicts:

| Component | Role | Logic |
| :--- | :--- | :--- |
| **Shared Infra** | Redis & Angie | Started once. Acts as the gateway. |
| **Instance** | Web & MQ Services | Unique per deployment name. |

**Routing Logic:**

`http://server-ip/inst1` ➡️ `parus-web-inst1:8080`

`http://server-ip/inst2` ➡️ `parus-web-inst2:8080`


## 📂 Project Structure

```text
p8-docker-deployment/
├── .env/                       # Folder for instance-specific configs
│   ├── inst1.env               # Config for 'inst1'
│   └── inst2.env               # Config for 'inst2'
├── angie.conf.template         # Dynamic routing template
├── docker-compose.shared.yml   # Redis & Angie
├── docker-compose.instance.yml # Web & MQ Services template
└── README.md                   # You are here!
```

## 🚀 Deployment Guide

### 1️⃣ Start the Shared Infrastructure
Run this once to bring up the proxy and cache:

**Unix / Podman:**

```bash
sudo podman compose -p shared -f docker-compose.shared.yml up -d
```

**Windows / PowerShell:**

```pwsh
docker compose -p shared -f docker-compose.shared.yml up -d
```

### 2️⃣ Create an Instance Config

Create `./env/inst1.env` with your Oracle credentials:

```ini
DEPLOYMENT_NAME=inst1

# DB Credentials for Report Service
PARUS_RPT_CONNECTION_KIND=oracle
PARUS_RPT_CONNECTION_STRING=direct=true;host=172.20.10.140;...user=parus_rpt;password=...
```

### 3️⃣ Launch a Specific Instance

Replace `inst1` with your target name:

**Unix / Podman:**

```bash
sudo DEPLOYMENT_NAME=inst1 podman compose -p inst1 -f docker-compose.instance.yml --env-file .env/inst1.env --profile report up -d
```

**Windows / PowerShell:**

```pwsh
$env:DEPLOYMENT_NAME="inst1"
docker compose -p inst1 -f docker-compose.instance.yml --env-file .env/inst1.env --profile report up -d
```

### 4️⃣ Access the Instance

Wait for the containers to start and visit:
`http://your-server-ip/inst1`

## ⚙️ Configuration Details

### Parus 8 Web Settings

To support subfolder routing, the following variables are automatically injected:
* `Hosting__UsePathBase=true`
* `Hosting__PathBase=/${DEPLOYMENT_NAME}`
* `Hosting__UseForwardedHeaders=true`

### Angie Reverse Proxy

The proxy uses a regex-based block to resolve container names dynamically:`location`

```nginx
location ~ ^/(?<inst>[^/]+) {
    # Add trailing slash if missing (e.g., /demo -> /demo/)
    # This must happen BEFORE proxy_pass
    rewrite ^/([^/]+)$ /$1/ permanent;

    set $target "parus-web-$inst";
    
    # Using a variable in proxy_pass requires a resolver (127.0.0.11 for Docker)
    proxy_pass http://$target:8080;
}
```

## ⚠️ Requirements

* **Docker** 20.10+
* **Docker Compose** v2.0+
* Internal network connectivity to the host.**Oracle Database**

## 🛠️ Maintenance

### Stop a shared infrastructure:

**Unix / Podman:**

```bash
sudo podman compose -p shared -f docker-compose.shared.yml down
```

**Windows / PowerShell:**

```pwsh
docker compose -p shared -f docker-compose.shared.yml down
```

### Stop a specific instance (e.g., `inst1`):

**Unix / Podman:**
```bash
sudo DEPLOYMENT_NAME=inst1 podman compose -p inst1 -f docker-compose.instance.yml --env-file .env/inst1.env down
```

**Windows / PowerShell:**
```pwsh
$env:DEPLOYMENT_NAME="inst1"
docker compose -p inst1 -f docker-compose.instance.yml --env-file .env/inst1.env down
```

### Stop all instances:

**Unix / Podman:**
```bash
sudo podman ps --filter "name=parus-web-" --format "{{.Names}}" | xargs -r -n1 -I{} sudo podman compose -p {} -f docker-compose.instance.yml --env-file .env/{}.env down
```

**Windows / PowerShell:**
```pwsh
docker ps --filter "name=parus-web-" --format "{{.Names}}" | ForEach-Object { docker compose -p $_ -f docker-compose.instance.yml --env-file .env/$_.env down }
```

### View logs for an instance:

**Unix / Podman:**
```bash
podman logs -f parus-web-inst1
```

**Windows / PowerShell:**
```pwsh
docker logs -f parus-web-inst1
```

### Check Health Status:

**Unix / Podman:**
```bash
podman ps  # Status should show (healthy)
```

**Windows / PowerShell:**
```pwsh
docker ps  # Status should show (healthy)
```
> [!IMPORTANT]  
> 📫 This is an experimental scenario. Use with caution in production.

## 🤝 **Community Standards**
We value respectful and constructive interactions. Please refer to our [Code of Conduct](CODE_OF_CONDUCT.md) for detailed guidelines on community behavior.

## 📄 **License**
This repository is licensed under a MIT License. Please see the [LICENSE](LICENSE) file for more details.