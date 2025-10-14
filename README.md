# 🧠 Private AI Stack  
### Deploy Your Own Self-Hosted AI Workflow Engine (Cloud or Local)

Deploy a powerful, private AI content engine in under an hour on your favorite cloud provider. This guide provides a full-stack, Docker-based solution optimized for ARM CPUs but easily adaptable for x86 and NVIDIA GPUs.

[![GitHub stars](https://img.shields.io/github/stars/pantaleone-ai/selfhosted-ai-engine?style=social)](https://github.com/pantaleone-ai/selfhosted-ai-engine/stargazers)
[![GitHub forks](https://img.shields.io/github/forks/pantaleone-ai/selfhosted-ai-engine?style=social)](https://github.com/pantaleone-ai/selfhosted-ai-engine/network/members)
[![GitHub issues](https://img.shields.io/github/issues/pantaleone-ai/selfhosted-ai-engine)](https://github.com/pantaleone-ai/selfhosted-ai-engine/issues)

The **Private AI Stack** is a self-contained system for running **LLM inference**, **automation workflows**, and **creative UI experiences** on your own hardware — cloud or local.

It uses:
- 🐳 **Docker Compose** for orchestration  
- ⚡ **llama.cpp (Ampere-tuned)** for inference  
- 🌐 **OpenWebUI** for chat + creative UI  
- 🔄 **n8n** for workflow automation  
- 🗃️ **PostgreSQL** and **Redis** for persistence  
- 🧩 **Traefik** for routing and reverse proxy  

## 📚 Table of Contents
- [Quick Start](#-quick-start)
- [Cloud Deployment (OCI, AWS, GCP)](#️-cloud-deployment-oci-aws-gcp)
- [Local Deployment (Mac, Windows, Linux)](#-local-deployment-mac-windows-linux)
- [AI Model Configuration](#-ai-model-configuration)
- [Environment and Override Files](#️-env-and-docker-composeoverrideyml)
- [Static Network Configuration](#-static-network-configuration)
- [Verifying Connectivity](#-verifying-connectivity)
- [Security Tips](#️-security-tips)
- [Next Steps](#-next-steps)
- [License](#-license)

---

## 🚀 Quick Start

### 🛠️ Prerequisites

- Docker + Docker Compose (v2+)
- Git
- 16GB+ RAM recommended (24GB+ optimal)
- Optional: GPU (NVIDIA, AMD ROCm, or Apple Metal)

---

## ☁️ Cloud Deployment (OCI, AWS, GCP)

Follow these steps for cloud setup (Oracle Cloud Infrastructure recommended for best ARM performance).

### 1️⃣ Provision a VM

| Provider | Recommended Specs |
|-----------|-------------------|
| **OCI (Ampere)** | 4 CPUs × 80 cores (A1.Flex), 24GB RAM |
| **AWS** | `c7g.4xlarge` (Graviton3 ARM) |
| **GCP** | `t2a-standard-8` (ARM-based) |

Use Ubuntu 22.04 LTS and ensure you open inbound ports:
- **80, 443** (HTTP/HTTPS)
- **8080** (LLM Server)
- **5678** (n8n)
- **3000** (OpenWebUI)

### 2️⃣ SSH Into Your Server

```bash
ssh ubuntu@<your_server_ip>
sudo apt update && sudo apt install -y docker.io docker-compose git
```

### 3️⃣ Clone and Configure

```bash
git clone https://github.com/pantaleone-ai/private-ai-stack.git
cd private-ai-stack
cp .env.example .env
```

### 4️⃣ Launch!

```bash
docker compose up -d
```

---

## 💻 Local Deployment (Mac, Windows, Linux)
### This stack runs locally for full offline inference and automation.

| Platform | Requirements |
|-----------|-------------------|
| **Mac M1/M2/M3** | Docker Desktop (Apple Silicon), Metal acceleration supported |
| **Windows 11** | Docker Desktop with WSL2 enabled |
| **Ubuntu Linux** | Docker Engine + Compose Plugin |

### ⚙️ Setup Steps

```bash
git clone https://github.com/pantaleone-ai/private-ai-stack.git
cd private-ai-stack
cp .env.example .env
```
You can now edit .env to match your system specs and choose CPU or GPU mode.

## 🧩 Example Configurations

### ✅ CPU Mode

```ini
LLAMA_ACCEL=cpu
```

### ⚡ GPU (NVIDIA)

```ini
LLAMA_ACCEL=cuda
```

### 🍏 GPU (Apple Silicon)

```ini
LLAMA_ACCEL=metal
```

### 🔥 GPU (AMD ROCm)

```ini
LLAMA_ACCEL=rocm
```

Then

```bash
docker compose build llama
docker compose up -d
```

---

## 🧠 AI Model Configuration
### By default, the stack expects a model at:
```bash
/models/qwen-3-4b-2507.gguf
```

Download models from anywhere such as:
- huggingface.co 
- Ampere optimized models - https://huggingface.co/AmpereComputing

Then place your .gguf file in the /models directory.  

## ⚙️ .env and docker-compose.override.yml

This project includes:
- .env.example — editable environment settings
- docker-compose.override.yml — static IP, resource controls, and GPU toggles

### 🧱 To Use:
```bash
cp .env.example .env
docker compose up -d
```

Modify .env to customize:
- Model location
- Thread count
- Port mappings
- Auth credentials

---

## 🧩 Optional Enhancements
### You can enable optional extensions for observability and tunneling:
| Profile         | Includes                   | Start Command                                  |
| --------------- | -------------------------- | ---------------------------------------------- |
| `observability` | Adds LangFuse + Prometheus | `docker compose --profile observability up -d` |
| `secure-tunnel` | Adds Cloudflared tunnel    | `docker compose --profile secure-tunnel up -d` |

These can be added in docker-compose.override.yml as modular profiles for clean separation.

 ---

## 🔍 Static Network Configuration
### Each container runs in a defined subnet for consistency:
| Service   | IP          | Port     |
| --------- | ----------- | -------- |
| Postgres  | 172.18.0.5  | 5432     |
| Redis     | 172.18.0.10 | 6379     |
| n8n       | 172.18.0.6  | 5678     |
| OpenWebUI | 172.18.0.7  | 3000     |
| Llama.cpp | 172.18.0.8  | 8080     |
| Traefik   | 172.18.0.9  | 80 / 443 |

 ---

## 🧪 Verifying Connectivity
### Test Llama Server
```bash
curl http://172.18.0.8:8080/models
```

Expected response:
```json
{"models":[{"name":"/models/qwen-3-4b-2507.gguf", "object":"model"}]}
```

### 🧰 Maintenance & Other Helpful Commands

| Action                    | Command                                | Description                                                               |
| :------------------------ | :------------------------------------- | :------------------------------------------------------------------------ |
| View logs (specific service) | `docker compose logs -f [service_name]` | Follow logs for a specific service.                                     |
| Start stack (detached)    | `docker compose up -d`                 | Start services in the background.                                         |
| Stop services             | `docker compose stop`                  | Stop running services without removing containers.                        |
| Start existing services   | `docker compose start`                 | Start previously stopped services.                                        |
| View running services     | `docker compose ps`                    | List all services and their status.                                       |
| Execute command in service | `docker compose exec [service_name] [command]` | Run a command inside a running service container.                       |
| Remove stopped containers | `docker compose rm`                    | Remove stopped service containers.                                        |
| Pull service images       | `docker compose pull`                  | Pull all service images.                                                  |
| View configuration        | `docker compose config`                | Validate and view the Compose file configuration.                         |
| Scale a service           | `docker compose up -d --scale [service_name]=[number]` | Scale a service to a desired number of containers.                  |
| Force recreate containers | `docker compose up -d --force-recreate` | Recreate all containers, even if their configuration hasn't changed.    |
| View resource usage       | `docker stats`                         | View live stream of container(s) resource usage statistics. (Not `compose` specific but very useful when using compose) |

 ---

## 🛡️ Security Tips
- Change all default passwords in .env; never share .env files with sensitive data in public!
- Use strong encryption keys for n8n
- Restrict ports to trusted networks or enable Traefik HTTPS routing

## 🧩 Next Steps
- Integrate n8n workflows for data-driven automation
- Extend OpenWebUI with fine-tuned models
- Add LangFuse for observability
- Use Cloudflared for secure remote access

## 🧾 License
MIT © 2025 — [Pantaleone AI](https://www.pantaleone.net/) & varous open source licenses as specified by each modular software or hardware component.  

### 🧑‍💻 Contributions,! Feedback, Pull Requests Welcome
Each service is modular — meaning you can add your own LLMs, APIs, or automation tools.  Excited to see what you will build!


Note: All code in this repo is provided for example purposes only. It is not intended for use in a production environment and has not been tested for security, reliability, or performance.
