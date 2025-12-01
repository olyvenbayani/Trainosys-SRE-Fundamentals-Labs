
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
1. Click the instance and click connect

<img width="1915" height="545" alt="image" src="https://github.com/user-attachments/assets/fccea819-6a40-4ac7-9812-2b1e4b216c57" />

2. Click Ec2 Instance Connect > Connect

<img width="1827" height="662" alt="image" src="https://github.com/user-attachments/assets/2b4ca88e-1600-4358-a70f-48c1be7221c3" />


### 3. Install Docker on the EC2 Instance
Once connected via SSH:
1. Update system packages: `sudo dnf update -y`.
2. Remove any existing Docker (if present): `sudo dnf remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine -y`.
3. Install Docker: `sudo dnf install -y docker`.
4. Start and enable Docker: `sudo systemctl start docker` and `sudo systemctl enable docker`.
5. Add your user to the Docker group for non-root access: `sudo usermod -a -G docker ec2-user`.
6. Log out and back in (exit SSH and reconnect) for group changes to take effect.
7. Verify: `docker --version` and `docker run hello-world`.
8. Install docker compose:
```
sudo curl -SL https://github.com/docker/compose/releases/download/v2.29.2/docker-compose-linux-x86_64 \
    -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose

docker-compose --version
```

