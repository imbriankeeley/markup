
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









x-custom-image: &custom_image
  image: ${IMAGE_NAME:-docker.io/frappe/erpnext}:${VERSION:-version-15}
  pull_policy: ${PULL_POLICY:-always}
  deploy:
    restart_policy:
      condition: always

services:
  tailscale:
    image: tailscale/tailscale:latest
    container_name: tailscaled
    hostname: frappe-dokploy
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - TS_AUTHKEY=<your-tailscale-auth-key>
      - TS_EXTRA_ARGS=--advertise-tags=tag:frappe
    volumes:
      - tailscale_data:/var/lib/tailscale
      - /dev/net/tun:/dev/net/tun
    network_mode: host
    restart: unless-stopped

  backend:
    <<: *custom_image
    volumes:
      - sites:/home/frappe/frappe-bench/sites
    networks:
      - bench-network
      - tailscale_network
    healthcheck:
      test:
        - CMD
        - wait-for-it
        - '0.0.0.0:8000'
      interval: 2s
      timeout: 10s
      retries: 30

  frontend:
    <<: *custom_image
    command:
      - nginx-entrypoint.sh
    depends_on:
      backend:
        condition: service_started
        required: true
      websocket:
        condition: service_started
        required: true
      tailscale:
        condition: service_started
        required: true
    environment:
      BACKEND: backend:8000
      FRAPPE_SITE_NAME_HEADER: ${FRAPPE_SITE_NAME_HEADER:-$$host}
      SOCKETIO: websocket:9000
      UPSTREAM_REAL_IP_ADDRESS: 127.0.0.1
      UPSTREAM_REAL_IP_HEADER: X-Forwarded-For
      UPSTREAM_REAL_IP_RECURSIVE: "off"
    volumes:
      - sites:/home/frappe/frappe-bench/sites
    network_mode: service:tailscale
    healthcheck:
      test:
        - CMD
        - wait-for-it
        - '0.0.0.0:8080'
      interval: 2s
      timeout: 30s
      retries: 30

  queue-default:
    <<: *custom_image
    command:
      - bench
      - worker
      - --queue
      - default
    volumes:
      - sites:/home/frappe/frappe-bench/sites
    networks:
      - bench-network
    healthcheck:
      test:
        - CMD
        - wait-for-it
        - 'redis-queue:6379'
      interval: 2s
      timeout: 10s
      retries: 30
    depends_on:
      configurator:
        condition: service_completed_successfully
        required: true

  queue-long:
    <<: *custom_image
    command:
      - bench
      - worker
      - --queue
      - long
    volumes:
      - sites:/home/frappe/frappe-bench/sites
    networks:
      - bench-network
    healthcheck:
      test:
        - CMD
        - wait-for-it
        - 'redis-queue:6379'
      interval: 2s
      timeout: 10s
      retries: 30
    depends_on:
      configurator:
        condition: service_completed_successfully
        required: true

  queue-short:
    <<: *custom_image
    command:
      - bench
      - worker
      - --queue
      - short
    volumes:
      - sites:/home/frappe/frappe-bench/sites
    networks:
      - bench-network
    healthcheck:
      test:
        - CMD
        - wait-for-it
        - 'redis-queue:6379'
      interval: 2s
      timeout: 10s
      retries: 30
    depends_on:
      configurator:
        condition: service_completed_successfully
        required: true

  scheduler:
    <<: *custom_image
    healthcheck:
      test:
        - CMD
        - wait-for-it
        - 'redis-queue:6379'
      interval: 2s
      timeout: 10s
      retries: 30
    command:
      - bench
      - schedule
    depends_on:
      configurator:
        condition: service_completed_successfully
        required: true
    volumes:
      - sites:/home/frappe/frappe-bench/sites
    networks:
      - bench-network

  websocket:
    <<: *custom_image
    healthcheck:
      test:
        - CMD
        - wait-for-it
        - '0.0.0.0:9000'
      interval: 2s
      timeout: 10s
      retries: 30
    command:
      - node
      - /home/frappe/frappe-bench/apps/frappe/socketio.js
    depends_on:
      configurator:
        condition: service_completed_successfully
        required: true
    volumes:
      - sites:/home/frappe/frappe-bench/sites
    networks:
      - bench-network
      - tailscale_network

  configurator:
    <<: *custom_image
    deploy:
      mode: replicated
      replicas: ${CONFIGURE:-0}
      restart_policy:
        condition: none
    entrypoint: ["bash", "-c"]
    command:
      - >
        [[ $${REGENERATE_APPS_TXT} == "1" ]] && ls -1 apps > sites/apps.txt;
        [[ -n `grep -hs ^ sites/common_site_config.json | jq -r ".db_host // empty"` ]] && exit 0;
        bench set-config -g db_host $$DB_HOST;
        bench set-config -gp db_port $$DB_PORT;
        bench set-config -g redis_cache "redis://$$REDIS_CACHE";
        bench set-config -g redis_queue "redis://$$REDIS_QUEUE";
        bench set-config -g redis_socketio "redis://$$REDIS_QUEUE";
        bench set-config -gp socketio_port $$SOCKETIO_PORT;
    environment:
      DB_HOST: "${DB_HOST:-db}"
      DB_PORT: "3306"
      REDIS_CACHE: redis-cache:6379
      REDIS_QUEUE: redis-queue:6379
      SOCKETIO_PORT: "9000"
      REGENERATE_APPS_TXT: "${REGENERATE_APPS_TXT:-0}"
    volumes:
      - sites:/home/frappe/frappe-bench/sites
    networks:
      - bench-network

  create-site:
    <<: *custom_image
    deploy:
      mode: replicated
      replicas: ${CREATE_SITE:-0}
      restart_policy:
        condition: none
    entrypoint: ["bash", "-c"]
    command:
      - >
        wait-for-it -t 120 $$DB_HOST:$$DB_PORT;
        wait-for-it -t 120 redis-cache:6379;
        wait-for-it -t 120 redis-queue:6379;
        export start=`date +%s`;
        until [[ -n `grep -hs ^ sites/common_site_config.json | jq -r ".db_host // empty"` ]] && \
          [[ -n `grep -hs ^ sites/common_site_config.json | jq -r ".redis_cache // empty"` ]] && \
          [[ -n `grep -hs ^ sites/common_site_config.json | jq -r ".redis_queue // empty"` ]];
        do
          echo "Waiting for sites/common_site_config.json to be created";
          sleep 5;
          if (( `date +%s`-start > 120 )); then
            echo "could not find sites/common_site_config.json with required keys";
            exit 1
          fi
        done;
        echo "sites/common_site_config.json found";
        [[ -d "sites/${SITE_NAME}" ]] && echo "${SITE_NAME} already exists" && exit 0;
        bench new-site --mariadb-user-host-login-scope='%' --admin-password=$${ADMIN_PASSWORD} --db-root-username=root --db-root-password=$${DB_ROOT_PASSWORD} $${INSTALL_APP_ARGS} $${SITE_NAME};
    volumes:
      - sites:/home/frappe/frappe-bench/sites
    environment:
      SITE_NAME: ${SITE_NAME}
      ADMIN_PASSWORD: ${ADMIN_PASSWORD}
      DB_HOST: ${DB_HOST:-db}
      DB_PORT: "${DB_PORT:-3306}"
      DB_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      INSTALL_APP_ARGS: ${INSTALL_APP_ARGS}
    networks:
      - bench-network

  migration:
    <<: *custom_image
    deploy:
      mode: replicated
      replicas: ${MIGRATE:-0}
      restart_policy:
        condition: none
    entrypoint: ["bash", "-c"]
    command:
      - >
        curl -f http://${SITE_NAME}:8080/api/method/ping || echo "Site busy" && exit 0;
        bench --site all set-config -p maintenance_mode 1;
        bench --site all set-config -p pause_scheduler 1;
        bench --site all migrate;
        bench --site all set-config -p maintenance_mode 0;
        bench --site all set-config -p pause_scheduler 0;
    volumes:
      - sites:/home/frappe/frappe-bench/sites
    networks:
      - bench-network

  db:
    image: mariadb:10.6
    deploy:
      mode: replicated
      replicas: ${ENABLE_DB:-0}
      restart_policy:
        condition: always
    healthcheck:
      test: mysqladmin ping -h localhost --password=${DB_ROOT_PASSWORD}
      interval: 1s
      retries: 20
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --skip-character-set-client-handshake
      - --skip-innodb-read-only-compressed
    environment:
      - MYSQL_ROOT_PASSWORD=${DB_ROOT_PASSWORD}
      - MARIADB_ROOT_PASSWORD=${DB_ROOT_PASSWORD}
    volumes:
      - db-data:/var/lib/mysql
    networks:
      - bench-network

  redis-cache:
    deploy:
      restart_policy:
        condition: always
    image: redis:6.2-alpine
    volumes:
      - redis-cache-data:/data
    networks:
      - bench-network
    healthcheck:
      test:
        - CMD
        - redis-cli
        - ping
      interval: 5s
      timeout: 5s
      retries: 3

  redis-queue:
    deploy:
      restart_policy:
        condition: always
    image: redis:6.2-alpine
    volumes:
      - redis-queue-data:/data
    networks:
      - bench-network
    healthcheck:
      test:
        - CMD
        - redis-cli
        - ping
      interval: 5s
      timeout: 5s
      retries: 3

  redis-socketio:
    deploy:
      restart_policy:
        condition: always
    image: redis:6.2-alpine
    volumes:
      - redis-socketio-data:/data
    networks:
      - bench-network
    healthcheck:
      test:
        - CMD
        - redis-cli
        - ping
      interval: 5s
      timeout: 5s
      retries: 3

volumes:
  db-data:
  redis-cache-data:
  redis-queue-data:
  redis-socketio-data:
  sites:
    driver_opts:
      type: "${SITE_VOLUME_TYPE}"
      o: "${SITE_VOLUME_OPTS}"
      device: "${SITE_VOLUME_DEV}"
  tailscale_data:

networks:
  bench-network:
  tailscale_network:
    external: true
    name: tailscale
