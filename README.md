# Self-Hosted Agentic AI Content Engine

Deploy a powerful, private AI content engine in under an hour on your favorite cloud provider. This guide provides a full-stack, Docker-based solution optimized for ARM CPUs but easily adaptable for x86 and NVIDIA GPUs.

[![GitHub stars](https://img.shields.io/github/stars/pantaleone-ai/selfhosted-ai-engine?style=social)](https://github.com/pantaleone-ai/selfhosted-ai-engine/stargazers)
[![GitHub forks](https://img.shields.io/github/forks/pantaleone-ai/selfhosted-ai-engine?style=social)](https://github.com/pantaleone-ai/selfhosted-ai-engine/network/members)
[![GitHub issues](https://img.shields.io/github/issues/pantaleone-ai/selfhosted-ai-engine)](https://github.com/pantaleone-ai/selfhosted-ai-engine/issues)

## Table of Contents
- [Prerequisites](#prerequisites)
- [Part 1: Cloud Server Setup](#part-1-cloud-server-setup)
- [Part 2: Server Configuration & Deployment](#part-2-server-configuration--deployment)
- [Part 3: Post-Installation Setup](#part-3-post-installation-setup)
- [Part 4: Hardware Customization (x86 & GPU)](#part-4-hardware-customization-x86--gpu)
- [Part 5: Management and Maintenance](#part-5-management-and-maintenance)

## Prerequisites
*   A **Cloud Account** with any of the providers listed below.
*   A **Registered Domain Name**.
*   An **SSH Key Pair** to connect to your server.
*   Basic command-line familiarity.

## Part 1: Cloud Server Setup

Choose your cloud provider, create a virtual machine, and open ports `80` (HTTP) and `443` (HTTPS) in its firewall. Once you can connect via SSH, proceed to **Part 2**.

<details>
<summary><b>Oracle Cloud Infrastructure (OCI) - Recommended Free Tier</b></summary>

1.  **Sign Up:** Get an [OCI Always Free](https://www.oracle.com/cloud/free/) account.
2.  **Create Instance:**
    *   Navigate to **Compute > Instances** and click **Create Instance**.
    *   **Image:** Select `Canonical Ubuntu 22.04`.
    *   **Shape:** Select `Ampere` > `VM.Standard.A1.Flex` and configure it with **4 OCPUs** and **24 GB Memory**.
    *   **Networking:** Ensure "Assign a public IPv4 address" is checked.
    *   **SSH:** Add your public SSH key.
3.  **Configure Firewall:**
    *   Go to your instance's VCN details -> **Security Lists** -> **Default Security List**.
    *   Add two **Ingress Rules** for `0.0.0.0/0` source: one for TCP on port `80` and another for TCP on port `443`.

</details>

<details>
<summary><b>Amazon Web Services (AWS)</b></summary>

1.  **Sign Up:** Get an [AWS Free Tier](https://aws.amazon.com/free/) account.
2.  **Launch EC2 Instance:**
    *   Go to the **EC2** service and click **Launch Instance**.
    *   **Image:** Select `Ubuntu Server 22.04 LTS` with `64-bit (Arm)` architecture.
    *   **Instance Type:** Choose an ARM-based `t4g` instance (e.g., `t4g.medium`).
    *   **Key Pair:** Assign your SSH key.
    *   **Network/Firewall:** Create a new Security Group and add Inbound Rules to allow `HTTP` and `HTTPS` from `Anywhere` (0.0.0.0/0).
3.  **Connect:** Use the public IP with the username `ubuntu`.

</details>

<details>
<summary><b>Google Cloud Platform (GCP)</b></summary>

1.  **Sign Up:** Get a [GCP Free Tier](https://cloud.google.com/free) account.
2.  **Create VM Instance:**
    *   Go to **Compute Engine > VM Instances** and click **Create Instance**.
    *   **Machine Config:** Select the `T2A` series (ARM) or `E2` (x86). A `t2a-standard-4` is a good starting point.
    *   **Boot Disk:** Choose `Ubuntu 22.04 LTS` (ensure it's the ARM version if you chose a T2A machine).
    *   **Firewall:** Check both **Allow HTTP traffic** and **Allow HTTPS traffic**.
    *   **SSH:** Add your public SSH key under the `Security` section.

</details>

<details>
<summary><b>Microsoft Azure</b></summary>

1.  **Sign Up:** Get an [Azure Free Account](https://azure.microsoft.com/en-us/free/).
2.  **Create a Virtual Machine:**
    *   Search for "Virtual Machines" and click **Create**.
    *   **Image:** Select `Ubuntu Server 22.04 LTS`.
    *   **Size:** Click "See all sizes" and find an ARM-based instance from the **Dpsv5** or **Epsv5** series.
    *   **Authentication:** Select "SSH public key" and add your key.
    *   **Inbound port rules:** Allow `HTTP` (80) and `HTTPS` (443).
3.  **Network Security:** Double-check that your Network Security Group (NSG) has inbound rules allowing TCP traffic on ports 80 and 443.

</details>

<details>
<summary><b>DigitalOcean</b></summary>

1.  **Sign Up:** Create a [DigitalOcean](https://www.digitalocean.com/) account.
2.  **Create a Droplet:**
    *   Select `Ubuntu 22.04 LTS`.
    *   Choose a plan under **CPU options** with adequate RAM (e.g., 16GB+).
    *   **Authentication:** Add your SSH key.
3.  **Configure Firewall:**
    *   Go to **Networking > Firewalls** and create a new firewall.
    *   Add **Inbound Rules** for `HTTP` and `HTTPS` from all sources.
    *   Apply this firewall to your Droplet.

</details>

## Part 2: Server Configuration & Deployment

These steps are universal for all providers.

1.  **Connect to your server via SSH.**
    ```bash
    ssh -i /path/to/your/private_key ubuntu@<YOUR_PUBLIC_IP>
    ```

2.  **Configure the Server Firewall (UFW):**
    ```bash
    sudo ufw allow ssh   # Port 22
    sudo ufw allow http  # Port 80
    sudo ufw allow https # Port 443
    sudo ufw enable
    ```

3.  **Install Docker and Docker Compose:**
    ```bash
    sudo apt-get update
    sudo apt-get install docker.io docker-compose -y
    sudo usermod -aG docker ${USER}
    newgrp docker
    ```

4.  **Prepare Project Directory and Files:**
    ```bash
    mkdir ai-engine && cd ai-engine
    touch acme.json
    chmod 600 acme.json
    mkdir -p /path/to/your/models
    nano docker-compose.yml
    ```

5.  **Create `docker-compose.yml`:**
    ```yaml
      version: "3.8"
      services:
        postgres:
          image: ankane/pgvector:latest
          container_name: postgres_local
          ports:
            - "5432:5432"
          environment:
            POSTGRES_USER: "user"
            POSTGRES_PASSWORD: "password"
            POSTGRES_DB: "database"
          volumes:
            - postgres_data:/var/lib/postgresql/data
          networks:
            internal:
              ipv4_address: 172.18.0.5
          labels:
            - "traefik.enable=false"
          deploy:
            resources:
              limits:
                cpus: '0.5'
                memory: 512m
          restart: unless-stopped
      
        redis:
          image: redis:latest
          container_name: redis
          volumes:
            - redis_data:/data
          networks:
            internal:
              ipv4_address: 172.18.0.3
          labels:
            - "traefik.enable=false"
          deploy:
            resources:
              limits:
                cpus: '0.25'
                memory: 256m
          restart: unless-stopped
      
        n8n:
          image: n8nio/n8n:latest
          container_name: n8n
          environment:
            DB_TYPE: "postgresdb"
            DB_POSTGRESDB_HOST: "postgres"
            DB_POSTGRESDB_PORT: 5432
            DB_POSTGRESDB_DATABASE: "database"
            DB_POSTGRESDB_USER: "user"
            DB_POSTGRESDB_PASSWORD: "password"
            DB_POSTGRESDB_SSL: "false"
            EXECUTIONS_MODE: "regular"
            CACHE_MODE: "redis"
            CACHE_REDIS_HOST: "redis"
            CACHE_REDIS_PORT: 6379
            CACHE_REDIS_DB: 0
            N8N_HOST: "your-domain.com"
            N8N_PORT: 5678
            N8N_PROTOCOL: "https"
            N8N_EDITOR_BASE_URL: "https://your-domain.com"
            WEBHOOK_URL: "https://your-domain.com"
            N8N_LOG_LEVEL: "info"
            N8N_COMMUNITY_PACKAGES_ENABLED: "true"
            NODE_FUNCTION_ALLOW_EXTERNAL: "*"
            N8N_PUSH_BACKEND: "websocket"
            N8N_DIAGNOSTICS_ENABLED: "false"
            N8N_TEMPLATES_ENABLED: "false"
            GENERIC_TIMEZONE: "America/Denver"
          volumes:
            - n8n_data:/home/node/.n8n
          labels:
            - "traefik.enable=true"
            - "traefik.http.routers.n8n.rule=Host(`your-domain.com`)"
            - "traefik.http.routers.n8n.entrypoints=websecure"
            - "traefik.http.routers.n8n.tls=true"
            - "traefik.http.routers.n8n.tls.certresolver=resolver"
            - "traefik.http.services.n8n.loadbalancer.server.port=5678"
          networks:
            internal:
              ipv4_address: 172.18.0.6
          depends_on:
            - postgres
            - redis
          deploy:
            resources:
              limits:
                cpus: '1'
                memory: 4g
          restart: unless-stopped
      
        openwebui:
          image: ghcr.io/open-webui/open-webui:main
          container_name: openwebui
          environment:
            OPENAI_API_BASE_URLS: "http://llama:8080/"
            WEBUI_URL: "https://your-webui-domain.com"
            DATA_DIR: "/app/backend/data"
            NO_PROXY: "0.0.0.0,localhost,127.0.0.1,host.docker.internal"
            UV_SYSTEM_PYTHON: "true"
            WEBUI_NAME: "WebUI"
            REDIS_HOST: "redis"
            REDIS_PORT: 6379
            REDIS_DB: 1
          volumes:
            - openwebui_data:/app/backend/data
          labels:
            - "traefik.enable=true"
            - "traefik.http.routers.openwebui.rule=Host(`your-webui-domain.com`)"
            - "traefik.http.routers.openwebui.entrypoints=websecure"
            - "traefik.http.routers.openwebui.tls=true"
            - "traefik.http.routers.openwebui.tls.certresolver=resolver"
            - "traefik.http.services.openwebui.loadbalancer.server.port=8080"
          networks:
            internal:
              ipv4_address: 172.18.0.7
          depends_on:
            - llama
            - redis
          deploy:
            resources:
              limits:
                cpus: '0.5'
                memory: 2g
          extra_hosts:
            - "host.docker.internal:host-gateway"
          restart: unless-stopped
      
        llama:
          image: amperecomputingai/llama.cpp:latest
          container_name: llama
          entrypoint: /llm/llama-server
          command:
            - "--model"
            - "/models/model.gguf"
            - "--host"
            - "0.0.0.0"
            - "--port"
            - "8080"
            - "--threads"
            - "3"
            - "--jinja"
            - "--flash-attn"
            - "auto"
            - "-sm"
            - "row"
            - "--temp"
            - "0.6"
            - "--top-k"
            - "20"
            - "--top-p"
            - "0.95"
            - "-c"
            - "40960"
            - "-n"
            - "32768"
            - "--no-mmap"
            - "--mlock"
          ports:
            - "8080:8080"
          volumes:
            - /path/to/your/models:/models
            - /path/to/your/data:/data
          networks:
            internal:
              ipv4_address: 172.18.0.8
          deploy:
            resources:
              limits:
                cpus: '3'
                memory: 18g
          restart: unless-stopped
      
        traefik:
          image: traefik:v2.11
          container_name: traefik
          command:
            - "--providers.docker=true"
            - "--providers.docker.exposedbydefault=false"
            - "--entrypoints.web.address=:80"
            - "--entrypoints.websecure.address=:443"
            - "--certificatesresolvers.resolver.acme.httpchallenge.entrypoint=web"
            - "--certificatesresolvers.resolver.acme.caserver=https://acme-v02.api.letsencrypt.org/directory"
            - "--certificatesresolvers.resolver.acme.email=your-email@example.com"
            - "--certificatesresolvers.resolver.acme.storage=/letsencrypt/acme.json"
            - "--log.level=INFO"
            - "--providers.file.filename=/etc/traefik/traefik_dynamic.yml"
          ports:
            - "80:80"
            - "443:443"
          volumes:
            - /var/run/docker.sock:/var/run/docker.sock:ro
            - ./acme.json:/letsencrypt/acme.json
            - /path/to/your/traefik_dynamic.yml:/etc/traefik/traefik_dynamic.yml:ro
          networks:
            internal:
              ipv4_address: 172.18.0.2
          labels:
            - "traefik.enable=true"
            - "traefik.http.routers.http-catchall.rule=HostRegexp(`{host:.+}`)"
            - "traefik.http.routers.http-catchall.entrypoints=web"
            - "traefik.http.routers.http-catchall.middlewares=redirect-to-https@docker"
            - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
            - "traefik.http.middlewares.redirect-to-https.redirectscheme.permanent=true"
          deploy:
            resources:
              limits:
                cpus: '0.25'
                memory: 256m
          restart: unless-stopped
      
      networks:
        internal:
          driver: bridge
          ipam:
            config:
              - subnet: 172.18.0.0/16
      
      volumes:
        postgres_data:
        n8n_data:
        openwebui_data:
        redis_data:
        letsencrypt:
        crawl4ai_data:
        langfuse_data:
        llama_data:
    ```

6.  **Launch the AI Engine:**
    ```bash
    docker compose up -d
    ```

## Part 3: Post-Installation Setup

You must enable the `pgvector` extension in your database for AI memory and embeddings.

1.  **Connect to the PostgreSQL container:**
    ```bash
    docker exec -it postgres_local psql -U user -d database
    ```
    *(Use the `POSTGRES_USER` and `POSTGRES_DB` values from your `docker-compose.yml`)*

2.  **Enable the extension and exit:**
    ```sql
    CREATE EXTENSION vector;
    \q
    ```

Your services are now live and accessible at the domains you configured.

## Part 4: Hardware Customization (x86 & GPU)

The default configuration is for ARM CPUs. Use these instructions to adapt to different hardware.

### Option 1: Running on a Standard x86 CPU (Intel/AMD)

In your `docker-compose.yml`, find the `llama` service and simply change the Docker image:
```yaml
  llama:
    # image: amperecomputingai/llama.cpp:latest  # <-- This is for ARM64
    image: ghcr.io/ggerganov/llama.cpp:server    # <-- Use this for x86-64
    # ... rest of the configuration is unchanged
```

### Option 2: Accelerating with an NVIDIA GPU

This provides a significant performance boost but requires a GPU-enabled server and extra setup.

1.  **Prepare the Host:** On your GPU-enabled VM, install the latest [NVIDIA drivers](https://www.nvidia.com/Download/index.aspx) and the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html). Reboot after installation.

2.  **Modify `docker-compose.yml`:** Update the `llama` service as follows:
    ```yaml
    llama:
      # image: amperecomputingai/llama.cpp:latest          # <-- Replace CPU image
      image: ghcr.io/ggerganov/llama.cpp:server-cuda     # <-- Use CUDA-enabled image
      container_name: llama
      command:
        - "--model"
        - "/models/your-model.gguf"
        - "--host"
        - "0.0.0.0"
        - "--n-gpu-layers"                                 # <-- Add this flag
        - "99"                                             # <-- Offload all possible layers to GPU
        # ... other flags
      deploy:                                              # <-- Add this entire deploy block
        resources:
          reservations:
            devices:
              - driver: nvidia
                count: 1
                capabilities: [gpu]
      restart: unless-stopped
      # ... rest of config (ports, volumes, networks)
    ```

## Part 5: Management and Maintenance

*   **View Logs for a Service:**
    ```bash
    docker compose logs -f <service_name>
    ```
*   **Stop All Services:**
    ```bash
    docker compose down
    ```
*   **Update All Services:**
    ```bash
    docker compose pull && docker compose up -d
    ```
*   **Manage AI Models:**
    Get a shell inside the llama container to download new models.
    ```bash
    docker exec -it llama /bin/bash
    huggingface-cli download-login
    huggingface-cli download GGUF-repo/model-name --local-dir /models
    exit
    docker compose restart llama
    ```*   **System Cleanup:**
    ```bash
    docker image prune    # Remove unused images
    docker volume prune   # Remove unused volumes
    ```
