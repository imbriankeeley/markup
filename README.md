# Tailscale + Dokploy Secure Deployment Guide

This guide details how to secure your Google Cloud instance by disabling public SSH access, using Tailscale for all SSH connections, and ensuring that Dokploy can still access the root directory to manage deployments.

---

## 1. Prerequisites

- **Google Cloud Instance:**  
  - Ubuntu 24.04 (or similar) with Docker, Docker Compose, and Dokploy already installed.
- **Tailscale Account:**  
  - An active Tailscale account with an authentication key.
- **Dokploy Setup:**  
  - Dokploy Cloud or a local Dokploy agent ready for deployment tasks.
- **Access:**  
  - Root privileges on your instance (via `sudo`).
- **Familiarity With:**  
  - Basic Linux commands, Google Cloud Console (for firewall rules), and editing configuration files.

---

## 2. Install and Configure Tailscale for SSH Access

### a. Install Tailscale

1. **Update Your Package Index:**
   ```bash
   sudo apt update
Install Tailscale:
```bash
Copy
curl -fsSL https://tailscale.com/install.sh | sh
```
b. Authenticate and Connect to Your Tailscale Network
Bring Up Tailscale:
Replace <YOUR_AUTH_KEY> with your Tailscale authentication key and choose a hostname (e.g., erp-prod-vm):

```bash
Copy
sudo tailscale up --authkey=<YOUR_AUTH_KEY> --hostname erp-prod-vm
```
Verify the Connection:
Check the Tailscale-assigned IP address:

bash
Copy
tailscale ip -4
You should see an IP in the Tailscale range (e.g., 100.x.x.x).

Test SSH via Tailscale:
From another Tailscale-connected device, run:

bash
Copy
ssh <tailscale-ip> -l <your-username>
Confirm that you can access the instance through its Tailscale IP.

3. Disable Public SSH Access
a. Update Google Cloud Firewall Rules
Open Google Cloud Console:
Navigate to VPC network > Firewall rules.

Create (or Update) a Firewall Rule to Allow SSH Only from Tailscale:

Name: allow-ssh-tailscale
Targets: Your VM instance (or a specific network tag like erp-prod).
Source IP Ranges: Use the Tailscale subnet (typically 100.64.0.0/10) or specific Tailscale IPs.
Protocols/Ports: TCP port 22.
Example using the gcloud CLI:

bash
Copy
gcloud compute firewall-rules create allow-ssh-tailscale \
  --direction=INGRESS \
  --priority=1000 \
  --network=default \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=100.64.0.0/10 \
  --target-tags=erp-prod
Block Public SSH Access:

Remove or disable any firewall rules that allow SSH (port 22) from 0.0.0.0/0.
Ensure that only the Tailscale-specific rule is active.
b. (Optional) Bind SSH Only to the Tailscale Interface
Edit the SSH Configuration:

bash
Copy
sudo nano /etc/ssh/sshd_config
Set the ListenAddress:
Add a line with your Tailscale IP (from the tailscale ip -4 output):

plaintext
Copy
ListenAddress 100.x.x.x
Comment out or remove other ListenAddress entries.

Restart SSH:

bash
Copy
sudo systemctl restart sshd
4. Ensure Dokploy Can Access the Root Directory
Dokploy must be able to manage system tasks (like updating files and restarting Docker) even though public SSH is disabled.

a. Install and Configure the Dokploy Agent Locally
Install the Dokploy Agent:
Follow Dokploy’s installation documentation. For example:
bash
Copy
curl -O https://dokploy.com/agent/install.sh
sudo bash install.sh
Configure the Agent:
Edit the agent configuration file (commonly found at /etc/dokploy/agent.conf).
Ensure it runs with root-level privileges (or via sudo).
Confirm it is set to access the Docker socket. For example:
bash
Copy
DOCKER_SOCKET=/var/run/docker.sock
Set Up the Agent as a Systemd Service:
Create a systemd unit file (if one is not already provided):
bash
Copy
sudo nano /etc/systemd/system/dokploy-agent.service
Insert the following:
ini
Copy
[Unit]
Description=Dokploy Agent Service
After=docker.service

[Service]
ExecStart=/usr/local/bin/dokploy-agent --config /etc/dokploy/agent.conf
Restart=always
User=root
Environment=DOCKER_SOCKET=/var/run/docker.sock

[Install]
WantedBy=multi-user.target
Reload systemd and start the service:
bash
Copy
sudo systemctl daemon-reload
sudo systemctl enable dokploy-agent
sudo systemctl start dokploy-agent
Verify the Agent:
Check logs to ensure the agent is running correctly:
bash
Copy
sudo journalctl -u dokploy-agent -f
b. Alternatively, Use Dokploy’s Local CLI
If you’re not using a dedicated agent, ensure that any Dokploy CLI commands are executed locally on the VM with the necessary root privileges (using sudo where required). The CLI should access Docker through /var/run/docker.sock.

5. Validate the Complete Setup
a. Test Tailscale SSH Access
From a Tailscale-connected Device:
Try connecting using:
bash
Copy
ssh <tailscale-ip> -l <your-username>
Confirm access.
b. Confirm Public SSH is Blocked
From Outside Tailscale:
Attempt to SSH to the VM’s public IP. The connection should fail.
c. Verify Dokploy Operations
Trigger a Deployment:
Use the Dokploy Cloud GUI or CLI to trigger a deployment.
Monitor Logs:
Check that the Dokploy agent can access the root directory, manage Docker, and perform system tasks as expected.
d. Confirm Outbound Connectivity
Firewall Check:
Ensure that while inbound SSH is restricted, the instance can still make outbound connections required by Dokploy to communicate with its cloud service.
6. Additional Security Considerations
Keep Software Updated:
Regularly update Tailscale, Dokploy, and system packages.
Monitor Logs:
Use Google Cloud Operations Suite or another logging solution to monitor for access attempts and system events.
Plan for Backup Access:
Consider setting up an alternative secure management channel in case Tailscale experiences issues.
Conclusion
By following this guide, you will:

Secure SSH Access:
Disable public SSH and restrict access to only those devices connected via Tailscale.
Enable Secure Management:
Allow Dokploy to manage your system with root-level access through a locally running agent.
Enhance Security:
Leverage Google Cloud firewall rules and local configurations to provide a layered security approach.
Save this file for future reference and modify it as needed for your specific environment. Happy deploying!

