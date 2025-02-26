
### **Step 1: Create a Google Cloud VM**
1. **Log in to Google Cloud Console**:
   - Navigate to **Compute Engine > VM Instances**.
   - Click **Create Instance**.

2. **Configure the VM**:
   - Choose an **e2-medium** (2 vCPUs, 4 GB RAM) for lightweight deployments or **e2-standard-2** (2 vCPUs, 8 GB RAM) for more capacity.
   - Select your preferred region and zone.
   - Under **Boot disk**, choose Ubuntu 22.04 LTS.
   - Set the disk size to at least **50 GB SSD**.

3. **Enable IP Forwarding**:
   - Under **Management, security, disks, networking, sole tenancy**, go to the **Networking** tab.
   - Set **IP Forwarding** to **On** (required for Tailscale subnet routing).

4. **Firewall Rules**:
   - Allow UDP port 41641 for Tailscale:
     - Go to **VPC Network > Firewall Rules**, and create two ingress rules:
       - Allow `0.0.0.0/0` for UDP port 41641.
       - Allow `::/0` for UDP port 41641.

5. Click **Create** to launch the VM.

---

### **Step 2: Install Docker**
6. SSH into your VM:
   ```bash
   gcloud compute ssh <vm-name>
   ```

7. Install Docker:
   ```bash
   curl -fsSL https://get.docker.com -o install-docker.sh
   sudo sh install-docker.sh
   ```

8. Add your user to the Docker group (optional):
   ```bash
   sudo usermod -aG docker $USER
   ```
   Log out and back in for this to take effect.

9. Verify Docker installation:
   ```bash
   docker run --rm hello-world
   ```

---

### **Step 3: Install Tailscale**
10. Run Tailscale as a Docker container:
   Create a `docker-compose.yml` file:
   ```yaml
   version: "3.9"
   services:
     tailscale:
       image: tailscale/tailscale:latest
       container_name: tailscaled
       cap_add:
         - NET_ADMIN
         - NET_RAW
       environment:
         TS_AUTHKEY: "<your-auth-key>" # Replace with your Tailscale auth key
       volumes:
         - tailscale_data:/var/lib/tailscale
         - /dev/net/tun:/dev/net/tun
       network_mode: host
       restart: unless-stopped

   volumes:
     tailscale_data:
   ```

11. Start Tailscale:
   ```bash
   docker compose up -d
   ```

12. Authenticate Tailscale:
   Check the container logs for an authentication link:
   ```bash
   docker logs tailscaled
   ```
   Open the link in your browser and log in to your Tailscale account.

13. Advertise subnet routes (optional):
   ```bash
   docker exec tailscaled tailscale up --advertise-routes=10.x.x.x/24 --accept-dns=false
   ```

---

### **Step 4: Install Dokploy**
14. Install Dokploy using its installation script:
   ```bash
   curl -sSL https://dokploy.com/install.sh | sh
   ```

15. Dokploy will automatically set up its Docker containers and expose ports 80 (HTTP) and 443 (HTTPS).

---

### **Step 5: Combine Tailscale and Dokploy**
Modify Dokploy’s `docker-compose.yml` file (usually found in `/var/lib/dokploy/docker-compose.yml`) to include Tailscale as a service:

```yaml
version: "3.9"

services:
  dokploy:
    image: dokploy/dokploy:latest
    container_name: dokploy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - dokploy_data:/data
    restart: unless-stopped

  tailscale:
    image: tailscale/tailscale:latest
    container_name: tailscaled
    cap_add:
      - NET_ADMIN
      - NET_RAW
    environment:
      TS_AUTHKEY: "<your-auth-key>"
    volumes:
      - tailscale_data:/var/lib/tailscale
      - /dev/net/tun:/dev/net/tun
    network_mode: host
    restart: unless-stopped

volumes:
  dokploy_data:
  tailscale_data:
```

Restart both services with Docker Compose:
```bash
docker compose up -d
```

---

```yaml
version: "3.9"

services:
  dokploy:
    image: dokploy/dokploy:latest
    container_name: dokploy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - dokploy_data:/data
    restart: unless-stopped

  frappe:
    image: frappe/frappe:latest
    container_name: frappe
    depends_on:
      - tailscale
    network_mode: service:tailscale
    environment:
      - FRAPPE_SITE_NAME=your_site_name
    volumes:
      - frappe_sites:/home/frappe/frappe-bench/sites

  tailscale:
    image: tailscale/tailscale:latest
    container_name: tailscaled
    hostname: frappe-dokploy
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - TS_AUTHKEY=<your-tailscale-auth-key>
      - TS_EXTRA_ARGS=--advertise-tags=tag:containers
      - TS_STATE_DIR=/var/lib/tailscale
    volumes:
      - tailscale_data:/var/lib/tailscale
      - /dev/net/tun:/dev/net/tun
    network_mode: host
    restart: unless-stopped

volumes:
  dokploy_data:
  frappe_sites:
  tailscale_data:
```

To deploy this setup:

1. Replace `<your-tailscale-auth-key>` with your actual Tailscale auth key.

2. Save the file as `docker-compose.yml` in your project directory.

3. Run the following command to start the services:
   ```
   docker compose up -d
   ```

4. Monitor the Tailscale container logs to ensure it connects successfully:
   ```
   docker logs tailscaled
   ```

5. Once Tailscale is connected, you can access your Frappe instance via its Tailscale IP or MagicDNS hostname.

6. Use Dokploy's web interface to manage your Frappe deployment.

This setup integrates Tailscale as a sidecar for both Dokploy and Frappe, prov
