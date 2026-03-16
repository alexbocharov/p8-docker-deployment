# 🚢 Parus 8 Docker Deployment (Experimental) 🐳

This repository contains an experimental, highly scalable Docker Compose configuration for deploying multiple **Parus 8 Web Client** environments using a single entry point.

## 🌟 Key Features

*   **Dynamic Routing:** Use a single port (80) to access multiple instances via URL paths (e.g., `/inst1`, `/inst2`).
*   **Shared Resources:** All environments share a single **Redis** cache and **Angie** (Nginx-compatible) Reverse Proxy to save RAM.
*   **Zero-Config Scaling:** Add a new deployment just by creating a `.env` file—no need to edit `angie.conf` or `docker-compose.yml`.
*   **Optimized for Parus 8:** Includes specific Gzip, timeout, and buffer settings required for heavy Oracle-based reports.

---

## 🏗️ Architecture Overview

The setup uses **Angie** as a smart router. It detects the environment name from the URL path and forwards the request to the correct container:


| Access URL | Internal Target | Redis (Shared) |
| :--- | :--- | :--- |
| `http://ip/inst1` | `parus-web-inst1:8080` | `redis:6379` |
| `http://ip/inst2` | `parus-web-inst2:8080` | `redis:6379` |

---

## 📂 Project Structure

```text
p8-docker-deployment/
├── .env/                  # Folder for instance-specific configs
│   ├── inst1.env          # Config for 'inst1'
│   └── inst2.env          # Config for 'inst2'
├── angie.conf.template    # Dynamic Reverse Proxy configuration template
├── docker-compose.yml     # Main orchestration file
└── README.md              # You are here!
```

## 🚀 How to Deploy a New Instance

### 1️⃣ Create an Environment File

Create a file in (e.g., ):`./env/{DEPLOYMENT_NAME}.env./env/inst1.env`

```ini
# Database Connection
AuthSettings__Connections__0__connectionString=direct=true;host=172.20.10.140;...
Connection__ConnectionString=direct=true;host=172.20.10.140;...

# Instance Specific Apps (Add as many as needed)
AuthSettings__Applications__0__code=Admin
AuthSettings__Applications__0__name=Administrator
```

### 2️⃣ Launch the Environment

Run the following command, replacing with your desired name:`inst1`

**Unix:**
```bash
DEPLOYMENT_NAME=inst1 docker-compose --env-file .env/inst1.env --profile report up -d
```

**Windows:**
```pwsh
# Set the deployment name for the container names
$env:DEPLOYMENT_NAME="inst1"
# Run with the specific env-file and profiles
docker-compose --env-file .env/inst1.env --profile report up -d
```

### 3️⃣ Access via Browser

Wait for the containers to start and visit:
`http://your-server-ip/inst1`

## ⚙️ Configuration Details

### Parus 8 Web Settings

To support subfolder routing, the following variables are automatically injected:
* `Hosting__UsePathBase=true`
* `Hosting__PathBase=/${DEPLOYMENT_NAME}`
* `Hosing__UseForwardedHeaders=true`

### Angie Reverse Proxy

The proxy uses a regex-based block to resolve container names dynamically:`location`

```nginx
location ~ ^/(?<inst>[^/]+)/ {
    proxy_pass http://parus-web-$inst:8080;
}
```

## ⚠️ Requirements

* **Docker** 20.10+
* **Docker Compose** v2.0+
* Internal network connectivity to the host.**Oracle Database**

## 🛠️ Maintenance

To view logs for a specific instance:

```bash
docker logs -f parus-web-inst1
```

To stop a specific instance:

**Unix:**
```bash
DEPLOYMENT_NAME=inst1 docker compose --env-file .env/inst1.env down
```

**Windows:**
```pwsh
$env:DEPLOYMENT_NAME="inst1"
docker compose --env-file .env/inst1.env down
```

📫 This is an experimental scenario. Use with caution in production environments.

## 🤝 **Community Standards**
We value respectful and constructive interactions. Please refer to our [Code of Conduct](CODE_OF_CONDUCT.md) for detailed guidelines on community behavior.

## 📄 **License**
This repository is licensed under a MIT License. Please see the [LICENSE](LICENSE) file for more details.