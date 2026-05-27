# 🚀 Ultimate Guide: Deploying Semaphore UI with Docker & Bind Mounts

## 📖 Overview & Scenario

### What is this Guide?
This is a comprehensive, practical documentation for deploying **Semaphore UI**, a lightweight and modern alternative to Ansible Tower/AWX. This guide focuses on using **Docker Compose** with **Bind Mounts** for data persistence, ensuring you have full control over your configuration files and database.

### Why Semaphore UI?
*   **Lightweight:** Written in Go, it consumes minimal resources compared to AWX.
*   **User-Friendly:** Provides a clean web interface to run Ansible playbooks.
*   **Secure:** Manages SSH keys, vault passwords, and environment variables securely.
*   **Open Source:** Fully open-source and community-driven.

### The Scenario
You are a DevOps Engineer or System Administrator managing a home lab or small infrastructure. You want to automate server configurations using Ansible but find the command-line interface (CLI) difficult to share with your team or track historically. You need a centralized dashboard that is easy to back up, restore, and maintain. You chose **Bind Mounts** because you want to see your database and config files directly on your host file system for easy backup and inspection.

### Key Challenges Addressed
1.  **Data Persistence:** How to keep your data safe even if the container is deleted.
2.  **Permission Issues:** Solving the common `permission denied` error when using Bind Mounts.
3.  **Backup Strategy:** Simple methods to back up and restore your automation data.
4.  **Verification:** Steps to ensure the service is running correctly.

---

## 📑 Table of Contents

1. [Prerequisites & Requirements](#1-prerequisites--requirements)
2. [Installation & Setup](#2-installation--setup)
3. [Verification & Troubleshooting](#3-verification--troubleshooting)
4. [Initial Configuration & Usage](#4-initial-configuration--usage)
5. [Backup & Restore Operations](#5-backup--restore-operations)
6. [Maintenance & Updates](#6-maintenance--updates)
7. [Uninstall & Cleanup](#7-uninstall--cleanup)

---

## 1. Prerequisites & Requirements

Before starting, ensure your system meets the following requirements.

### Hardware Requirements
*   **CPU:** Minimum 1 Core (2 Cores recommended).
*   **RAM:** Minimum 512 MB (1 GB recommended).
*   **Storage:** At least 5 GB of free space for Docker images and data.

### Software Requirements
*   **Operating System:** Linux (Ubuntu 20.04/22.04, CentOS 8+, Debian 11+) or macOS.
*   **Docker Engine:** Must be installed and running.
*   **Docker Compose:** V2 plugin installed.

### Network Requirements
*   **Port 4000:** Must be open on your firewall for web access.
*   **Internet Access:** Required to pull the Docker image.

### Check Installed Versions

Use the following commands to verify your Docker installation.

**Check Docker Version**
This command displays the installed Docker version and build information.
```bash
docker --version
```
*   `docker`: The main CLI command.
*   `--version`: Flag to show version info.

**Check Docker Compose Version**
This command ensures Docker Compose V2 is available.
```bash
docker compose version
```
*   `docker compose`: The V2 plugin command.
*   `version`: Flag to show compose version.

**Check System Resources**
Ensure you have enough disk space.
```bash
df -h .
```
*   `df`: Disk free space command.
*   `-h`: Human-readable format (GB/MB).
*   `.`: Current directory.

---

## 2. Installation & Setup

This section guides you through setting up Semaphore UI using **Bind Mounts**. This method maps local directories to the container, making data management transparent.

### Step 1: Create Project Directory

Create a dedicated folder for your Semaphore setup to keep things organized.

**Create Directory and Navigate**
Creates the folder `semaphore-ui` in your home directory and moves into it.
```bash
mkdir -p ~/semaphore-ui
cd ~/semaphore-ui
```
*   `mkdir -p`: Creates directory and parents if they don't exist.
*   `~/semaphore-ui`: Path to the new directory.
*   `cd`: Change directory command.

### Step 2: Prepare Data Directories

Create the local folders that will hold your database and configurations. This helps avoid permission issues during the first run.

**Create Data Folders**
Creates three sub-directories for database, config, and temporary files.
```bash
mkdir -p ./data/semaphore_db
mkdir -p ./data/semaphore_config
mkdir -p ./data/semaphore_tmp
```
*   `./data/semaphore_db`: Will store the SQLite database file.
*   `./data/semaphore_config`: Will store `config.json`.
*   `./data/semaphore_tmp`: Will store temporary execution logs/files.

### Step 3: Create Docker Compose File

Create the `docker-compose.yml` file with the correct configuration. We use `user: "0:0"` to run as root inside the container, which simplifies permission handling for Bind Mounts in home labs.

**Create and Edit File**
Opens the nano editor to create the compose file.
```bash
nano docker-compose.yml
```

**Copy the Following Content:**

```yaml
version: '3.8'

services:
  semaphore:
    container_name: semaphore-ui
    image: semaphoreui/semaphore:v2.17.26
    user: "0:0" 
    ports:
      - "4000:3000" 
    environment:
      SEMAPHORE_DB_DIALECT: sqlite
      SEMAPHORE_ADMIN: admin
      SEMAPHORE_ADMIN_PASSWORD: semaphoreui#2620#
      SEMAPHORE_ADMIN_NAME: Admin
      SEMAPHORE_ADMIN_EMAIL: admin@localhost
    volumes:
      - ./data/semaphore_db:/var/lib/semaphore
      - ./data/semaphore_config:/etc/semaphore
      - ./data/semaphore_tmp:/tmp/semaphore
    restart: unless-stopped
```

**Explanation of Key Fields:**
*   `image: semaphoreui/semaphore:v2.17.26`: Pins the specific version for stability.
*   `user: "0:0"`: Runs the container as Root to avoid `permission denied` errors on host folders.
*   `ports: "4000:3000"`: Maps host port 4000 to container port 3000.
*   `volumes`: Maps local `./data` folders to container paths for persistence.
*   `environment`: Sets up the initial admin user and SQLite database.

### Step 4: Pull and Start Service

Download the image and start the container in detached mode.

**Pull Image**
Downloads the Semaphore image from Docker Hub without starting it.
```bash
docker compose pull
```
*   `pull`: Downloads the image layers.

**Start Container**
Creates and starts the container in the background.
```bash
docker compose up -d
```
*   `up`: Creates and starts containers.
*   `-d`: Detached mode (runs in background).

---

## 3. Verification & Troubleshooting

After starting the service, you must verify it is running correctly and troubleshoot any issues.

### Verify Container Status

Check if the container is up and running.

**List Running Containers**
Filters the process list to show only Semaphore.
```bash
docker ps | grep semaphore
```
*   `docker ps`: Lists running containers.
*   `| grep semaphore`: Filters output for "semaphore".

**Expected Output:**
You should see `STATUS: Up` and `PORTS: 0.0.0.0:4000->3000/tcp`.

### Check Logs

View the last 50 lines of logs to ensure no errors occurred during startup.

**View Logs**
Displays the standard output of the container.
```bash
docker logs semaphore-ui --tail 50
```
*   `logs`: Fetches logs from a container.
*   `--tail 50`: Shows only the last 50 lines.

**Success Indicator:**
Look for `INFO[0000] Listening on :3000`.

### Troubleshooting: Permission Denied Error

If you see `panic: open /etc/semaphore/config.json: permission denied`, it means the container cannot write to the host folder.

**Fix Permissions (Method 1: Chmod)**
Grants read/write/execute permissions to all users for the data folder.
```bash
chmod -R 777 ./data
```
*   `chmod`: Change mode/command.
*   `-R`: Recursive (applies to all subfolders).
*   `777`: Read/Write/Execute for Owner, Group, Others.

**Fix Permissions (Method 2: Chown)**
Changes ownership to UID 1001 (standard non-root user in many containers).
```bash
sudo chown -R 1001:1001 ./data
```
*   `chown`: Change owner.
*   `1001:1001`: User ID and Group ID.

**Restart After Fix**
Restarts the container to apply changes.
```bash
docker compose down
docker compose up -d
```

### Web Access Verification

Open your browser and navigate to:
`http://<YOUR_SERVER_IP>:4000`

Login with:
*   **Username:** `admin`
*   **Password:** `semaphoreui#2620#`

---

## 4. Initial Configuration & Usage

Once logged in, follow these steps to configure Semaphore for real-world use.

### 1. Create a Project

Projects isolate keys, inventories, and repositories.

1.  Click **"New Project"**.
2.  Enter Name: `Home Lab Automation`.
3.  Click **Create**.

### 2. Add SSH Keys

Semaphore needs SSH keys to connect to your target servers.

1.  Go to **Key Store** > **New Key**.
2.  **Type:** Select `SSH Private Key`.
3.  **Name:** `My Server Key`.
4.  **Private Key:** Paste the content of your `~/.ssh/id_rsa` file.
5.  Click **Save**.

### 3. Add Inventory

Define the servers you want to manage.

1.  Go to **Inventory** > **New Inventory**.
2.  **Type:** `Static`.
3.  **Content:**
    ```ini
    [webservers]
    192.168.0.93 ansible_user=root
    192.168.0.245 ansible_user=root
    ```
4.  Click **Save**.

### 4. Add Repository

Link your Git repository containing Ansible playbooks.

1.  Go to **Repositories** > **New Repository**.
2.  **Git URL:** `https://github.com/yourusername/ansible-playbooks.git`
3.  If private, select the SSH Key added earlier.
4.  Click **Save**.

### 5. Run a Playbook

1.  Go to **Task Templates** > **New Template**.
2.  **Name:** `Update Servers`.
3.  **Playbook:** Select `site.yml` (or your playbook file).
4.  **Inventory:** Select `Home Lab`.
5.  **SSH Key:** Select `My Server Key`.
6.  Click **Run**.

You will see real-time logs in the dashboard.

---

## 5. Backup & Restore Operations

Since we used Bind Mounts, backing up is as simple as copying a folder.

### Backup Procedure

Stop the container to ensure database consistency, then archive the data.

**Stop Container**
Stops the running service gracefully.
```bash
docker compose down
```

**Create Backup Archive**
Compresses the `data` folder into a tarball with a timestamp.
```bash
tar -czf semaphore-backup-$(date +%F).tar.gz ./data
```
*   `tar`: Tape archive utility.
*   `-czf`: Create, Gzip compress, File name.
*   `$(date +%F)`: Inserts current date (YYYY-MM-DD).
*   `./data`: Source directory.

**Start Container**
Restarts the service after backup.
```bash
docker compose up -d
```

### Restore Procedure

To restore from a backup, replace the current data folder with the backup.

**Stop Container**
```bash
docker compose down
```

**Remove Current Data**
⚠️ **Warning:** This deletes current data. Ensure you have a backup.
```bash
rm -rf ./data/*
```
*   `rm -rf`: Remove recursively and force.
*   `./data/*`: All contents of the data folder.

**Extract Backup**
Replace `semaphore-backup-2026-05-28.tar.gz` with your actual file name.
```bash
tar -xzf semaphore-backup-2026-05-28.tar.gz -C .
```
*   `-xzf`: Extract, Gzip decompress, File name.
*   `-C .`: Extract to current directory.

**Start Container**
```bash
docker compose up -d
```

---

## 6. Maintenance & Updates

Regular maintenance ensures security and stability.

### Update Semaphore Version

To upgrade to a newer version (e.g., v2.18.0):

**Edit Compose File**
Open the file and change the image tag.
```bash
nano docker-compose.yml
```
Change `image: semaphoreui/semaphore:v2.17.26` to `image: semaphoreui/semaphore:v2.18.0`.

**Pull New Image**
Downloads the new version.
```bash
docker compose pull
```

**Recreate Container**
Stops old container and starts new one with updated image.
```bash
docker compose up -d
```

### Check Disk Usage

Monitor how much space your data is consuming.

**Check Data Folder Size**
Displays the size of the data directory.
```bash
du -sh ./data
```
*   `du`: Disk usage.
*   `-sh`: Summarize, Human-readable.

### Log Rotation

Docker handles log rotation by default, but you can check log size.

**Check Container Log Size**
```bash
docker inspect semaphore-ui --format='{{.LogPath}}'
ls -lh $(docker inspect semaphore-ui --format='{{.LogPath}}')
```

---

## 7. Uninstall & Cleanup

If you want to completely remove Semaphore from your system.

### Remove Container and Network

Stops and removes the container and network.
```bash
docker compose down
```

### Remove Data Volumes (Bind Mounts)

Since we used Bind Mounts, Docker won't delete your data folder automatically. You must delete it manually.

⚠️ **Warning:** This permanently deletes all projects, keys, and history.

**Delete Data Directory**
Removes the local data folder.
```bash
rm -rf ./data
```

**Remove Project Folder**
If you want to remove the entire setup.
```bash
cd ~
rm -rf ~/semaphore-ui
```

### Remove Docker Image

Free up disk space by removing the downloaded image.

**Remove Image**
Deletes the Semaphore image from local storage.
```bash
docker rmi semaphoreui/semaphore:v2.17.26
```
*   `rmi`: Remove Image.

---

## 📝 Conclusion

You have successfully deployed **Semaphore UI** using Docker Compose with Bind Mounts. This setup provides:
1.  **Full Control:** Direct access to database and config files.
2.  **Easy Backup:** Simple tar-based backup strategy.
3.  **Persistence:** Data survives container restarts and deletions.
4.  **Troubleshooting:** Clear steps to handle permission issues.

This guide is designed for practical, real-world usage in home labs and small enterprise environments. For further customization, refer to the [Official Semaphore Documentation](https://docs.semaphoreui.com/).

---
*Document prepared for Sumon's Home Lab Infrastructure.*
*Last Updated: May 28, 2026*