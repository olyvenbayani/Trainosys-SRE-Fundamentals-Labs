
## Alternative Cloud-Based Setup: Using AWS EC2 with Docker and VS Code Access
If you prefer a cloud VM over local installation (e.g., for resource-intensive tasks or to avoid local setup), set up an AWS EC2 instance with Docker installed and access it remotely via VS Code. This uses the AWS Free Tier (eligible for new accounts) to avoid costs—stick to t3.micro and monitor usage.

### Prerequisites
- An AWS account (sign up at [aws.amazon.com](https://aws.amazon.com) if you don't have one; Free Tier is available for 12 months from signup before July 15, 2025, or 6 months after).
- Basic familiarity with terminals/SSH.
- Install VS Code locally (see above) with the Remote - SSH extension.

### 1. Launch the EC2 Instance
1. Open the Amazon EC2 console at [https://console.aws.amazon.com/ec2/](https://console.aws.amazon.com/ec2/).
2. Select your preferred AWS Region (e.g., us-east-1).
3. From the dashboard, choose **Launch instance**.
4. Under **Name and tags**, enter a descriptive name (e.g., "SRE-Workshop-EC2").
5. Under **Application and OS Images (Amazon Machine Image)**:
   - Choose **Quick Start**.
   - Select **Amazon Linux** as the OS.
   - Pick an AMI marked **Free Tier eligible** (e.g., Amazon Linux 2023).
6. Under **Instance type**, select a Free Tier eligible type like **t3.micro**.
7. Under **Key pair (login)**:
   - Choose **Create new key pair** (e.g., name it "sre-workshop-key").
   - Download the .pem file and store it securely.
   - (Do not proceed without a key pair for security.)
8. Under **Network settings**:
   - Use the default VPC and subnet.
   - Edit the security group to allow inbound SSH traffic: Add a rule for **SSH** (port 22) from **0.0.0.0/0** (for testing; restrict to your IP in production).
9. Under **Configure storage**, keep the default root volume (e.g., 8-10 GB gp3).
10. Review the summary and choose **Launch instance**.
11. Wait for the instance status to show **running** and pass status checks (view in the EC2 console under Instances).

### 2. Connect to the EC2 Instance via SSH (Initial Verification)
1. In the EC2 console, select your instance and choose **Connect**.
2. On the **SSH client** tab, copy the example command (e.g., `ssh -i "sre-workshop-key.pem" ec2-user@ec2-XXX-XXX-XXX-XXX.compute-1.amazonaws.com`).
3. In your local terminal:
   - Navigate to the directory with your .pem file.
   - Run `chmod 400 sre-workshop-key.pem` (on macOS/Linux) to set permissions.
   - Paste and run the SSH command.
4. Type "yes" if prompted about the host fingerprint.
5. You should now be logged in as ec2-user@ip-address.

### 3. Install Docker on the EC2 Instance
Once connected via SSH:
1. Update system packages: `sudo dnf update -y`.
2. Remove any existing Docker (if present): `sudo dnf remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine -y`.
3. Install Docker: `sudo dnf install -y docker`.
4. Start and enable Docker: `sudo systemctl start docker` and `sudo systemctl enable docker`.
5. Add your user to the Docker group for non-root access: `sudo usermod -a -G docker ec2-user`.
6. Log out and back in (exit SSH and reconnect) for group changes to take effect.
7. Verify: `docker --version` and `docker run hello-world`.

### 4. Install Docker Compose on the EC2 Instance
While still connected via SSH:
1. Create the plugins directory: `sudo mkdir -p /usr/libexec/docker/cli-plugins/` (or `/usr/local/lib/docker/cli-plugins/` if preferred).
2. Download the latest Docker Compose plugin: `sudo curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 -o /usr/libexec/docker/cli-plugins/docker-compose`.
3. Make it executable: `sudo chmod +x /usr/libexec/docker/cli-plugins/docker-compose`.
4. Verify: `docker compose version` (note: use `docker compose` instead of `docker-compose` for v2+).

### 5. Access the EC2 Instance via VS Code
1. Locally, open VS Code and install the **Remote - SSH** extension (search in Extensions view, authored by Microsoft).
2. Open the Command Palette (Ctrl+Shift+P or Cmd+Shift+P on macOS) and select **Remote-SSH: Connect to Host...**.
3. Enter the SSH connection string: `ec2-user@ec2-XXX-XXX-XXX-XXX.compute-1.amazonaws.com` (replace with your instance's public DNS).
4. When prompted, select the platform as **Linux**.
5. For authentication, VS Code will use your local SSH config or prompt for the key—ensure your .pem file is in ~/.ssh/ or configure it in VS Code settings (add to SSH config file via **Remote-SSH: Open SSH Configuration File...**).
   - Example config entry in ~/.ssh/config:
     ```
     Host sre-workshop-ec2
       HostName ec2-XXX-XXX-XXX-XXX.compute-1.amazonaws.com
       User ec2-user
       IdentityFile ~/.ssh/sre-workshop-key.pem
     ```
   - Then connect using "sre-workshop-ec2" as the host.
6. Once connected, open a folder on the remote (e.g., /home/ec2-user/) via **File > Open Folder**.
7. Install VS Code extensions remotely as needed (e.g., Docker, YAML).
8. Use the integrated terminal in VS Code to run Docker commands, edit files, and proceed with the workshop setup.

**Tips for EC2 Setup**:
- **Costs**: Monitor in AWS Billing—t3.micro is free for ~750 hours/month in Free Tier.
- **Security**: After setup, restrict SSH to your IP in the security group.
- **Persistence**: Install other tools (e.g., Git, load testing) on the EC2 via SSH or VS Code terminal using similar commands (e.g., `sudo dnf install -y git apache2-utils` for Git and ab).
- **Cleanup**: Terminate the instance when done to avoid charges (EC2 console > Instances > Actions > Terminate).
- **Ports**: For workshop apps, add security group rules for ports like 5000, 9090 (allow from your IP or 0.0.0.0/0 temporarily).

## Additional Notes
- **No Local Python or Other Runtimes Needed**: All lab code runs inside Docker containers.
- **OS Compatibility**: Tested on macOS, Linux, and Windows (use WSL on Windows for terminal commands). EC2 uses Amazon Linux for cloud.
- **Troubleshooting**:
  - Docker errors? Check system requirements and restart the Docker service.
  - Firewall issues? Ensure ports like 5000 and 9090 are open locally or in EC2 security groups.
  - On corporate networks: You may need admin rights or VPN adjustments.
  - SSH connection issues? Verify key permissions (chmod 400) and security group rules.
- **Alternative to Local/EC2 Setup**: If installations are problematic, use GitHub Codespaces (free tier) for a browser-based environment—no local installs required.
- **Preparation Time**: Aim to complete this before the workshop starts.
Once set up, you're ready for Lab 1! If you run into issues, note them down for discussion during the workshop. Happy setting up! 🚀
