# Your Local Agentic AI Content Engine
Create an AI content engine in less than 60 minutes for free!  Note, some of my specific configs are included below; tune to your own use case!

# Step 1 - Get an OCI (or similar) Server

## 1. Sign Up for OCI Always Free

*   **Access the Free Tier:** Go to the [Oracle Cloud website](https://www.oracle.com/cloud/free/).
*   **Create Account:** Sign up for a Free Tier account.
    *   *Note:* A credit card is required for identity verification, but you will not be charged for Always Free resources unless you explicitly upgrade.

## 2. Launch Your Ampere VM Instance

Once logged into your OCI console, follow these steps to create your virtual machine (VM) instance:

*   **Navigate to Instances:** In the OCI console, select **Compute > Instances**.
*   **Create New Instance:** Click the "Create Instance" button.
*   **Instance Details:**
    *   **Name:** Provide a memorable name (e.g., `ai-content-factory`).
    *   **Image:** Click "Change image."
        *   Select **"Oracle Cloud Marketplace."**
        *   Search for `ubuntu-22-04-arm`.
        *   Choose the pre-configured image: `https://cloudmarketplace.oracle.com/marketplace/en_US/listing/165367725`.
        *   Click "Select Image."
    *   **Shape:** Click "Change shape."
        *   Select the **"Ampere ARM"** processor.
        *   Choose the **"VM.Standard.A1.Flex"** shape.
        *   Configure the shape to **4 OCPUs** and **24 GB of Memory**. These resources are part of the OCI Always Free tier.
    *   **SSH Key:**
        *   Set up your **SSH Key** (needed to connect to your instance).
        *   If you don't have one, OCI can generate it for you.
    *   **Networking:**
        *   Configure your **networking details**. The default Virtual Cloud Network (VCN) is usually sufficient for initial setup.
*   **Create Instance:** Click the "Create" button.
*   **Connect:** Allow a few minutes for provisioning. Once the instance is running, connect to it using an SSH client and your generated SSH key.

# Step 2 - Firewall Setup for Your OCI AI Content Engine

This guide provides simple steps to open the necessary ports for your AI Content Factory on Oracle OCI. You'll configure two layers of firewall rules:

1.  **Oracle OCI Console:** Your cloud-level firewall (Network Security Group or Security List).
2.  **Your Server's Firewall:** The internal firewall on your server (UFW for Ubuntu).

### **Overview: Ports to Open**

You only need to open two main ports externally to the internet:

*   **Port 80 (HTTP):** Used by Traefik for automatic SSL certificate generation (Let's Encrypt) and for redirecting traffic to HTTPS.
*   **Port 443 (HTTPS):** Used by Traefik for all secure web traffic to n8n, OpenWebUI, and Crawl4AI.

**Important:** All other component-specific ports (e.g., 8080 for Llama.cpp, 5678 for n8n) are handled *internally* by Docker and Traefik. **Do not open these directly to the internet.**

### **1: Open Ports in Oracle OCI Console**

This involves adding Ingress (incoming) rules to your Virtual Cloud Network's Security List or Network Security Group.

1.  **Log in** to your Oracle OCI Console.
2.  **Navigate:** Go to **Networking > Virtual Cloud Networks**.
3.  **Select your VCN:** Click on the Virtual Cloud Network associated with your AI Content Factory instance.
4.  **Access Security Rules:**
    *   If your instance uses the **Default Security List**, click on it.
    *   If you assigned your instance to a custom **Network Security Group (NSG)**, go to **Networking > Network Security Groups** and select your NSG.
5.  **Add Ingress Rules:** Click "Add Ingress Rules."

    *   **Rule 1 (for Port 80):**
        *   **Source Type:** `CIDR`
        *   **Source CIDR:** `0.0.0.0/0` (Allows traffic from any IP)
        *   **IP Protocol:** `TCP`
        *   **Destination Port Range:** `80`
        *   **Description (Optional):** `Allow HTTP for Traefik SSL setup`

    *   **Rule 2 (for Port 443):**
        *   **Source Type:** `CIDR`
        *   **Source CIDR:** `0.0.0.0/0` (Allows traffic from any IP)
        *   **IP Protocol:** `TCP`
        *   **Destination Port Range:** `443`
        *   **Description (Optional):** `Allow HTTPS for Traefik`

    *   Click "Add Ingress Rules" to save your changes.

### **2: Open Ports on Your Server's Firewall (UFW)**

Connect to your OCI instance via SSH to configure UFW (Uncomplicated Firewall).

1.  **SSH into your OCI instance:**
    ```bash
    ssh -i /path/to/your/ssh/key ubuntu@<YOUR_OCI_INSTANCE_PUBLIC_IP>
    ```
2.  **Enable UFW (if not already enabled):**
    ```bash
    sudo ufw enable
    ```
    *   If prompted about SSH connections, type `y` and press Enter.
3.  **Allow SSH (Port 22):** (Essential to maintain your connection)
    ```bash
    sudo ufw allow ssh
    ```
    *(Alternatively: `sudo ufw allow 22`)*
4.  **Allow HTTP (Port 80):**
    ```bash
    sudo ufw allow http
    ```
    *(Alternatively: `sudo ufw allow 80`)*
5.  **Allow HTTPS (Port 443):**
    ```bash
    sudo ufw allow https
    ```
    *(Alternatively: `sudo ufw allow 443`)*
6.  **Verify UFW status:**
    ```bash
    sudo ufw status
    ```
    Confirm that `80/tcp` and `443/tcp` (and `22/tcp` or `ssh`) are listed as `ALLOW Anywhere`.
    
---
---
# Start Your AI Brain (Ampere Llama.cpp Container) 🧠

This container runs your AI models. It's the core component for generating AI content.

*   **Start the server:** (--privileged=true optional!)
    ```bash
    sudo docker run --privileged=true --name llama --entrypoint /bin/bash -it amperecomputingai/llama.cpp:latest
    ```

*   **Download AI models (if needed):**
    ```bash
    huggingface-cli download AmpereComputing/llama-3.2-1b-instruct-gguf Llama-3.2-1B-Instruct-Q8R16.gguf --local-dir /models && ./llama-cli -m /models/Llama-3.2-1B-Instruct-Q8R16.gguf -t 3 -tb 3
    
    huggingface-cli download unsloth/DeepSeek-R1-0528-Qwen3-8B-GGUF DeepSeek-R1-0528-Qwen3-8B-Q4_K_M.gguf --local-dir /models && ./llama-cli -m /models/DeepSeek-R1-0528-Qwen3-8B-Q4_K_M.gguf -t 3 -tb 3

    huggingface-cli download unsloth/gemma-3-4b-it-qat-GGUF gemma-3-4b-it-qat-Q4_K_M.gguf --local-dir /models && ./llama-cli -m /models/gemma-3-4b-it-qat-Q4_K_M.gguf -t 3 -tb 3
    
    ```

    *   **Once you've tested inference and all is working, get out of the CLI via:**
    ```bash
      control + c
    ```

*   **Load model for use via llama-server:**
    ```bash
    ./llama-server -m /models/Llama-3.2-1B-Instruct-Q8R16.gguf --jinja -t 3 -tb 3 --host 0.0.0.0 --port 8080
    ```
    use --jinja for Qwen family models

*   **Keep container running:** To exit the CLI without stopping the container, press `CTRL + P`, then `CTRL + Q`.

  
---
# Step 2: Bring the Whole Crew Online with Docker Compose 🚀

This step launches all the other components of your AI content factory, orchestrating them with Docker Compose. Ensure the `docker-compose.yml` file below is accurately saved on your OCI instance (e.g., in `/home/ubuntu/ai/` as mentioned in the Traefik volume mount, or adjust path if needed).

*   **First, for Traefik and SSL configuration**, create a configuration file called `traefik_dynamic.yml` with the following content to protect your services from abuse (configure according to your requirements):
      ```yaml
    tls:
      stores:
        default:
    #      defaultCertificate:
    #        certFile: /etc/traefik/certs/default.crt
    #        keyFile: /etc/traefik/certs/default.key
      options:
        default:
          minVersion: VersionTLS12
          cipherSuites:
            - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
            - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
          curvePreferences:
            - CurveP256
            - CurveP384
    #      sniStrict: true

    http:
      middlewares:
        rate-limit:
          rateLimit:
            average: 100
            burst: 50
    ```



*   **Ensure `docker-compose.yml` is ready, then launch it:**
    ```bash
    docker compose up -d
    ```
    This command brings up your PostgreSQL database (for storing data), n8n (workflow automation), OpenWebUI (AI interface), Traefik (traffic controller), Redis (fast caching), and Crawl4AI (web-scraping), all in the background.

*   **Your `docker-compose.yml` blueprint:**

    ```yaml

      version: '3.8'
      
      services:
        # PostgreSQL Database (Local) with pgvector
        postgres:
          image: ankane/pgvector:latest
          container_name: postgres_local
          ports:
            - "5432:5432" 
         environment:
            POSTGRES_USER: n8n_user
            POSTGRES_PASSWORD: UPDATE-ME
            POSTGRES_DB: n8n_db
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
      
        # n8n Workflow Automation (Configured for Local Postgres & Redis)
        n8n:
          image: n8nio/n8n:latest
          container_name: n8n
          environment:
            # --- Local PostgreSQL Connection ---
            - DB_TYPE=postgresdb
            - DB_POSTGRESDB_HOST=postgres
            - DB_POSTGRESDB_PORT=5432
            - DB_POSTGRESDB_DATABASE=n8n_db
            - DB_POSTGRESDB_USER=n8n_user
            - DB_POSTGRESDB_PASSWORD=UPDATE-ME
            - DB_POSTGRESDB_SSL=false
      
            # --- Redis Connection ---
            - EXECUTIONS_MODE=regular
            - CACHE_MODE=redis
            - CACHE_REDIS_HOST=redis
            - CACHE_REDIS_PORT=6379
            # - CACHE_REDIS_PASSWORD=your_redis_password # Uncomment if Redis has a password
            - CACHE_REDIS_DB=0 # Explicitly setting n8n to use DB 0
      
            # --- n8n Specific Configuration ---
            #- N8N_ENCRYPTION_KEY=UPDATE-ME
            - N8N_HOST=n8n.rapigent.com
            - N8N_PORT=5678
            - N8N_PROTOCOL=https
            - N8N_EDITOR_BASE_URL=https://n8n.rapigent.com
            - WEBHOOK_URL=https://n8n.rapigent.com
            - N8N_LOG_LEVEL=info
            - N8N_COMMUNITY_PACKAGES_ENABLED=true
            - NODE_FUNCTION_ALLOW_EXTERNAL=* # Be cautious. List specific modules if possible.
            - N8N_PUSH_BACKEND=websocket
            - N8N_DIAGNOSTICS_ENABLED=false
            - N8N_TEMPLATES_ENABLED=false
            - GENERIC_TIMEZONE=America/Denver # Set to your timezone
          volumes:
            - n8n_data:/home/node/.n8n
          labels:
            - "traefik.enable=true"
            - "traefik.http.routers.n8n.rule=Host(`n8n.rapigent.com`)"
            - "traefik.http.routers.n8n.entrypoints=websecure"
            - "traefik.http.routers.n8n.tls=true"
            - "traefik.http.routers.n8n.tls.certresolver=myresolver"
            - "traefik.http.routers.n8n.tls.options=default"
            - "traefik.http.services.n8n.loadbalancer.server.port=5678"
          networks:
            internal:
              ipv4_address: 172.18.0.6
          depends_on:
            postgres:
              condition: service_started
            redis:
              condition: service_started
          deploy:
            resources:
              limits:
                cpus: '0.5'
                memory: 5g
          restart: unless-stopped
      
        # OpenWebUI AI Interface
        openwebui:
          image: ghcr.io/open-webui/open-webui:main
          container_name: openwebui
          environment:
            # Connects to llama.cpp on host port 8080
            - OPENAI_API_BASE_URLS=http://0.0.0.0:8080/v1
            - WEBUI_URL=https://openwebui.rapigent.com 
            - DATA_DIR=/app/backend/data
            - NO_PROXY=0.0.0.0,localhost,127.0.0.1,host.docker.internal
            - UV_SYSTEM_PYTHON=true
            - WEBUI_NAME=PantaleoneAI
            # --- PostgreSQL Connection for pgvector ---
            - PGVECTOR_URL=postgresql://UPDATE-ME
      
            # --- Redis Connection for OpenWebUI ---
            - REDIS_HOST=redis
            - REDIS_PORT=6379
            # - REDIS_PASSWORD=your_redis_password # Uncomment and set if your Redis has a password
            - REDIS_DB=1 # Using DB 1 for OpenWebUI
          volumes:
            - openwebui_data:/app/backend/data
          labels:
            - "traefik.enable=true"
            - "traefik.http.routers.openwebui.rule=Host(`openwebui.rapigent.com`)"
            - "traefik.http.routers.openwebui.entrypoints=websecure"
            - "traefik.http.routers.openwebui.tls=true"
            - "traefik.http.routers.openwebui.tls.certresolver=myresolver"
            - "traefik.http.routers.openwebui.tls.options=default"
            - "traefik.http.services.openwebui.loadbalancer.server.port=8080"
          networks:
            internal:
              ipv4_address: 172.18.0.7
          depends_on:
            - redis # Wait for redis container to be started
          deploy:
            resources:
              limits:
                cpus: '0.5'
                memory: 2g
          extra_hosts:
            - "host.docker.internal:host-gateway"
          restart: unless-stopped
      
        # Traefik Reverse Proxy (Configuration as provided by user initially)
        traefik:
          image: traefik:v2.11
          container_name: traefik
          command:
            - --providers.docker=true
            - --providers.docker.exposedbydefault=false
            - --entrypoints.web.address=:80
            - --entrypoints.websecure.address=:443
            - --certificatesresolvers.myresolver.acme.httpchallenge=true
            - --certificatesresolvers.myresolver.acme.httpchallenge.entrypoint=web
            - --certificatesresolvers.myresolver.acme.caserver=https://acme-v02.api.letsencrypt.org/directory
            - --certificatesresolvers.myresolver.acme.email=UPDATE-ME
            - --certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json
            - --log.level=INFO
            - --providers.file.filename=/etc/traefik/traefik_dynamic.yml # If you use a dynamic config file
            # Optional: Traefik Dashboard
            # - --api.dashboard=true
            # - --api.insecure=true # For local testing of dashboard only, DO NOT use in production
          ports:
            - "80:80"
            - "443:443"
          volumes:
            - /var/run/docker.sock:/var/run/docker.sock:ro
            - letsencrypt:/letsencrypt
            - /home/ubuntu/ai/traefik_dynamic.yml:/etc/traefik/traefik_dynamic.yml:ro # Mount your dynamic config
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
            # Labels for secure Traefik Dashboard (if enabled in command and DNS is set for traefik.rapigent.com)
            # - "traefik.http.routers.traefik-dashboard.rule=Host(`traefik.rapigent.com`)"
            # - "traefik.http.routers.traefik-dashboard.entrypoints=websecure"
            # - "traefik.http.routers.traefik-dashboard.tls=true"
            # - "traefik.http.routers.traefik-dashboard.tls.certresolver=myresolver"
            # - "traefik.http.routers.traefik-dashboard.service=api@internal"
            # - "traefik.http.middlewares.my-auth.basicauth.users=YOUR_USER:YOUR_HASHED_PASSWORD" # Example: admin:$$apr1$$...
            # - "traefik.http.routers.traefik-dashboard.middlewares=my-auth"
          deploy:
            resources:
              limits:
                cpus: '0.25' # Using previously agreed optimization
                memory: 256m # Using previously agreed optimization
          restart: unless-stopped
      
        # Redis Caching
        redis:
          image: redis:latest
          container_name: redis
          # command: ["redis-server", "--requirepass", "your_strong_redis_password"] # Uncomment to set password
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
                cpus: ‘0.25’
                memory: 256m
          restart: unless-stopped
      
        # Crawl4AI Web Scraping
      #  crawl4ai:
      #   image: unclecode/crawl4ai:latest
      #    container_name: crawl4ai
      #    environment:
      #      - PLAYWRIGHT_BROWSERS_PATH=/ms-playwright
            # FastAPI in crawl4ai image listens on 0.0.0.0:8000 by default
      #    volumes:
      #      - crawl4ai_data:/app/data
      #      - /dev/shm:/dev/shm # Important for Playwright/Chrome stability
      #    labels:
      #      - "traefik.enable=true"
      #      - "traefik.http.routers.crawl4ai.rule=Host(`crawl4ai.rapigent.com`)"
      #      - "traefik.http.routers.crawl4ai.entrypoints=websecure"
      #      - "traefik.http.routers.crawl4ai.tls=true"
      #      - "traefik.http.routers.crawl4ai.tls.certresolver=myresolver"
      #      - "traefik.http.services.crawl4ai.loadbalancer.server.port=8000" # crawl4ai listens on port 8000
      #    networks:
      #      internal:
      #        ipv4_address: 172.18.0.4
      #    deploy:
      #      resources:
      #        limits:
      #          cpus: '0.5'
      #          memory: 3g # Monitor this, Playwright can be memory hungry
      #    restart: unless-stopped
      
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
   
    ```
    *   **Remember to replace `email` with *your* actual email address in the `traefik` service configuration for SSL certificate issuance!**
 
    *   **Finally, makle sure all containers on the same network and add the llama container to the network ai_internal:**
         ```bash
             docker network connect --ip 172.18.0.8 ai_internal llama
         ```

    *   **For your PGVector Database, connect to it, then enable the PG Vector extension**
         ```bash
             docker exec -it postgres_local psql -U n8n_user -d n8n_db
         ```
         
         ```sql
            CREATE EXTENSION vector;
         ```
 
# Updating & Upgrading

*   **Remove containers via docker compose**
    ```bash
    docker compose down
    ```

    **Remove llama.cpp**
    ```bash
    docker stop my_container or docker stop "1a2b3c4d5e6f"
    ```

    **Remove the llama.cpp volume**
    ```bash
    docker rm "1a2b3c4d5e6f"
    ```

    **Download the latest docker image (if needed)**
    ```bash
    sudo docker pull amperecomputingai/llama.cpp:latest
    ```
    I often need to go back to the llama container command line to optimize of download new models.
    
    **Use this command to get back into the shell environment.  Then use the huggingface-cli to download new models per the instructions above**
    ```bash
     docker exec -it llama sh
    ```

# Other helpful commands

*   **Check server CPU and process utilization**
    ```bash
    top
    ```
*   **Check server RAM utilization**
    ```bash
    free -h
    ```

*   **Docker images not connected, then remove em!**
    ```bash
    docker images -f dangling=true

    docker image prune
    ```

*   **Similarly, kill Docker volumes not connected, then remove em!**
    ```bash
    docker volume ls -f dangling=true

    docker volume prune

    #remove a specific volume with
    docker volume rm <volume_name>
    ```
    
*   **Export all N8N credentials & copy to your servers' file system(or workflows or other settings)**
    ```bash
       docker exec -u node -it n8n n8n export:credentials --all --decrypted --output=/tmp/creds.json

       sudo docker cp n8n:/tmp/creds.json/home/ubuntu/automate/creds.json

    #then copy to your your new N8N docker file system via:
    docker cp ./my_file.txt my_container:/app/ 
    ```


*   **Create chat_history table for use with N8N, including vector embeddings(768)**
```sql
CREATE TABLE chat_history (
    id SERIAL PRIMARY KEY, -- A unique identifier for each message
    session_id VARCHAR(255) NOT NULL, -- To group messages belonging to the same chat session
    sender VARCHAR(255), -- The sender of the message
    content TEXT NOT NULL, -- The message content
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP, -- Timestamp when the message was created
    embedding VECTOR(768) -- A vector column to store embeddings (adjust dimension as needed, e.g., 1024 or 512)
);

-- Optional: Create an index on the embedding column for efficient vector search
CREATE INDEX ON chat_history USING hnsw (embedding vector_cosine_ops); -- Using HNSW index for cosine distance
```
    
---

