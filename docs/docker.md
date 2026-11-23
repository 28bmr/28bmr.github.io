---
title: Docker and Docker Compose
permalink: /docker.html
---
# Docker Self-Hosted Service Documentation

Brody Read - CYB 3353

This document outlines the steps I took to install Docker and Coker Compose on a Linux VM and deploy a self-hosted application. I chose Uptime Kuma, an open-source monitoring application that tracks uptime of websites and network services.  

## 1. Virtual Machine and Initial Snapshot

I used my existing Ubuntu Server VM on Proxmox (VM 335340100) named ubuntu-headless.

Before installing Docker, I made a snapshot named pre-docker in the Proxmox Snapshots tab.
![[Pasted image 20261122215752.png]]

#### System Info

`cat /etc/os-release`
Shows the Ubuntu distribution and version (lab requires headless).

`ip -br addr`
Gets the IP address of the VM.

Opening a new PowerShell window on the host windows machine,
`ssh bmr2303@XXX.XXX.XXX.XXX`
SSH into the VM.
## 2. Docker Installation

`sudo apt update && sudo apt install -y ca-certificates curl`
Update package lists and install certificates needed.

`sudo install -m 0755 -d /etc/apt/keyrings`
Create directory for APT repository signing keys.

`sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc`
Download Docker's official GPG.

`echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list`
Add Docker's official repository to APT.

`sudo apt update`
Update package lists.

`sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`
Installs Docker Engine, the buildx plugin, and the docker compose plugin.

#### Post installation check

`sudo systemctl enable --now docker`
Starts the Docker service immediately and when VM boots.

`systemctl is-enabled docker # enabled`
Verify Docker enabled on startup

`sudo docker run --rm hello-world` 
Run docker's test image to check installation.

`docker compose version`
Display docker version.
![[Pasted image 20261122222245.png]]

### 3. Deploying Uptime Kuma

#### Create the Application Directory

`mkdir -p ~/uptime-kuma/data`
`cd ~/uptime-kuma`
Creates directory (and moves to it) and a location for the persistent data.

#### Create the Compose File

`nano compose.yaml`
Makes and opens a new YAML config file.

```nano
services:
	uptime-kuma:
		image: louislam/uptime-kuma:2
		ports:
			- "3001:3001"
		volumes:
			- ./data:/app/data
		restart: unless-stopped
```
Image is for Uptime Kuma v2 and ports maps VM port 3001 to container port 3001. The mount in volumes preserves the application's database and settings outside the container. Restart allows the container to restart after a reboot unless it was stopped (especially useful for an uptime application, which should be running for long periods).

No Dockerfile needed since I used an existing application image.

#### Start the Application

`sudo docker compose config`
Display docker compose configuration.

`sudo docker compose up -d`
Downloads and starts Uptime Kuma in detached mode.

`sudo docker compose ps`
Displays container's running status and ports.
![[Pasted image 20261122222300.png]]

#### Configure the Application

I opened Uptime Kuma in a browser using the VM's IP address and port 3001 and began configuring:
`http://XXX.XXX.XXX.XXX:3001`
![[Pasted image 20261122222225.png|546]]

After creating an admin account, I added an HTTP(s) monitor for https://example.com to show functionality.
![[Pasted image 20261122222447.png]]

### 4. Persistent Storage and Reboot Testing

`sudo docker compose down`
Stops and removes the container and its Compose newtwork.

`sudo docker compose up -d`
Recreates and starts the container using the same persistent data.

After restarting the application, I confirmed that the previous monitor was still present.

I then tested if Uptime Kuma would start after a reboot.

`systemctl is-enabled docker`
Checks that Docker is enabled on startup.

`sudo reboot`
Restarts the VM.

After reconnecting through SSH, I ran:
`cd ~/uptime-kuma`
`sudo docker compose ps`
Goes back into directory and checks the container restarted.
![[Pasted image 20261122222950.png]]

![[Pasted image 20261122223023.png]]

### 5. SSH Security

To improve remote access security, I configured public-key authentication and disabled password authentication.

#### Generate an SSH Key

On my host windows computer, I generated an Ed25519 key.
`ssh-keygen -t ed25519 -C "ubuntu-headless"`
Generates a public/private SSH key pair using Ed25519. I piped the keys to an .ssh directory as ubuntu-headless and ubuntu-headless.pub.

#### Pipe the Public Key

````powershell
Get-Content "C:\Users\brody\.ssh\ubuntu-headless.pub" | ssh bmr2303@XXX.XXX.XXX.XXX "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys"
````
Pipes the public key generated to a new directory on the VM and applies proper permissions to the key directories.

#### Check Key-Based Authentication

`ssh -i "C:\Users\brody\.ssh\ubuntu-headless" bmr2303@XXX.XXX.XXX.XXX`
Connects to VM using private SSH key.

I successfully connected after typing in my passphrase rather than password:
![[Pasted image 20261122223850.png]]

#### Disable Password Authentication

`sudo nano /etc/ssh/sshd_config`
Opens SSH server config file.

I changed the following line:
![[Pasted image 20261122224020.png]]

#### Troubleshooting

After disabling password authentication in sshd_config, I was still able to log in with the password. After looking around, I found that sshd_config had an additional config file within it that contained another yes to password authentication. After changing this to no, password authentication no longer worked.
![[Pasted image 20261122224338.png]]
![[Pasted image 20261122224345.png]]

### 6. Firewall Configuration

I used Ubuntu's Uncomplicated Firewall (UFW) to adjust incoming and outgoing connections. I only allowed SSH and the Uptime Kuma application ports while denying all others.

#### Install and Configure UFW

`sudo apt install -y ufw`
Installs UFW.

`sudo ufw default deny incoming`
Defaults to block everything (allow necessary ports after).

`sudo ufw default allow outgoing`
Outgoing connections permitted by default.

`sudo ufw allow 22/tcp`
Allow incoming SSH.

`sudo ufw allow 3001/tcp`
Allow Uptime Kuma application port.

`sudo ufw enable`
Enables the firewall.

#### Verify Firewall Rules

`sudo ufw status`
Displays firewall configuration.
![[Pasted image 20261122224816.png]]

### 7. Automatic Security Updates

I configured Ubuntu's unattended-upgrades utility to auto install security updates.

#### Installl and Enable Auto Updates

`sudo apt install -y unattended-upgrades`
Installs unattended-upgrades package.

`sudo dpkg-reconfigure unattended-upgrades`
Runs the package config process to enable auto updates.
![[Pasted image 20261122225139.png]]

#### Verify Config

`cat /etc/apt/apt.conf.d/20auto-upgrades`
Displays config controlling automatic updates and unattended upgrades.
![[Pasted image 20261122225130.png]]

### 8. Backup Script

`cd ~/uptime-kuma`
`nano backup.sh`
Navigate to directory and enter backup script with nano
![[Pasted image 20261122232012.png]]
The script uses docker compose stop to prevent data from changing mid backup. The backup is timestamped and compressed with tar (containing data and compose.yaml). The application is then restarted and any backups not in the newest 7 are removed.

#### Execute the backup script

`chmod +x backup.sh`
Makes script exectuable.

`sudo ./backup.sh`
Runs backup script, creating a backup (ran twice).

`ls -lh backups/`
Show backups directory.
![[Pasted image 20261122231251.png]]

`sudo docker compose ps`
Verify Uptime Kuma running after backup.
![[Pasted image 20261122233525.png]]

Verify contents of backups.
![[Pasted image 20261122231336.png]]

#### Restoration Test

I FIRST created a backup in proxmox of the whole VM.
![[Pasted image 20261122233753.png]]

I created a new test monitor in uptime kuma:
![[Pasted image 20261122231818.png]]

I then backed up the test data before deleting it and restoring it.
![[Pasted image 20261122232619.png]]

Finally, I verified that the restore test monitor was restored in uptime kuma.
![[Pasted image 20261122232635.png]]
### Sources

1. CYB 3353 Lecture: Docker (University of Tulsa, Fall '26)
2. CYB 3353 Lecture: Docker Compose (University of Tulsa, Fall '26)
3. CYB 3353 Lecture: Project 2 (University of Tulsa, Fall '26)
4. https://docs.docker.com/engine/install/ubuntu/
5. https://docs.docker.com/compose/
6. https://github.com/louislam/uptime-kuma
