# PantaleoneAI - Agentic Content @ Scale
Create an AI content engine in less than 60 minutes

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

### **Step 1: Open Ports in Oracle OCI Console**

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

### **Step 2: Open Ports on Your Server's Firewall (UFW)**

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
