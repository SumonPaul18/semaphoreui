# Semaphore UI Production Deployment Guide

## Overview and Context

This guide documents the end-to-end deployment, configuration, security hardening, and maintenance of **Semaphore UI** (an open-source Ansible Web UI) in a production-grade environment. The context involves a DevOps engineer managing a private cloud lab with public IPs, utilizing Docker, Linux systems, and integrating with external storage solutions like Ceph.

The primary objectives addressed in this guide are:
1.  **Installation:** Deploying Semaphore UI using Docker Compose with persistent data storage.
2.  **Configuration:** Setting up SQLite/PostgreSQL databases, admin users, and security keys correctly to avoid initialization errors.
3.  **Security:** Implementing secure SSH key management via the Key Store (avoiding insecure host mounts), generating valid Base64 encryption keys, and restricting target server access.
4.  **Persistence & Backup:** Ensuring data durability using Bind Mounts on CephFS and implementing robust backup strategies.
5.  **Operations:** Managing inventories, repositories, and task templates for automated Ansible playbook execution.
6.  **Troubleshooting:** Resolving common issues such as permission denied errors, database migration failures, and invalid configuration formats.

This guide is structured for practical application, minimizing theory and focusing on actionable commands and real-world scenarios.

---

## Table of Contents

1. [Prerequisites and Environment Setup](#1-prerequisites-and-environment-setup)
2. [Semaphore UI Installation with Docker Compose](#2-semaphore-ui-installation-with-docker-compose)
3. [Database Configuration and Initialization](#3-database-configuration-and-initialization)
4. [Security Hardening and Key Management](#4-security-hardening-and-key-management)
5. [Production-Grade Storage Integration (CephFS)](#5-production-grade-storage-integration-cephfs)
6. [Operational Workflow: Inventory, Keys, and Playbooks](#6-operational-workflow-inventory-keys-and-playbooks)
7. [Backup, Maintenance, and Disaster Recovery](#7-backup-maintenance-and-disaster-recovery)
8. [Troubleshooting Common Issues](#8-troubleshooting-common-issues)
9. [Cleanup and Uninstallation](#9-cleanup-and-uninstallation)

---

## 1. Prerequisites and Environment Setup

Before deploying Semaphore UI, ensure the host system meets the necessary requirements. This setup assumes a Linux-based host (Ubuntu/CentOS/RHEL) with Docker installed.

### Requirements
*   **Operating System:** Linux (Ubuntu 20.04+, CentOS 8+, RHEL 8+).
*   **Docker Engine:** Latest stable version.
*   **Docker Compose:** V2 plugin (`docker compose`) or standalone binary.
*   **Network:** Open ports for web access (default 3000 or mapped port).
*   **Storage:** Local disk or mounted network storage (e.g., CephFS) for data persistence.
*   **Ansible:** Installed on target servers (not necessarily on the host, but required for playbooks).

### Verify Docker Installation

Check if Docker and Docker Compose are installed and running.

```bash
docker --version
```
*Explanation:* Displays the installed Docker client version.

```bash
docker compose version
```
*Explanation:* Displays the installed Docker Compose plugin version. Ensure it is V2.x for compatibility with modern syntax.

```bash
systemctl status docker
```
*Explanation:* Checks if the Docker daemon is active and running. If inactive, start it with `systemctl start docker`.

### Create Project Directory

Create a dedicated directory for Semaphore UI configuration and data.

```bash
mkdir -p /opt/semaphore-ui/data
cd /opt/semaphore-ui
```
*Explanation:* Creates the main project directory and a subdirectory for data persistence. Using `/opt` is standard for third-party software installations on Linux.

---

## 2. Semaphore UI Installation with Docker Compose

This section details the creation of a production-ready `docker-compose.yml` file. It uses Bind Mounts for data persistence, allowing easy backup and integration with network storage like Ceph.

### Generate Security Keys

Semaphore requires two Base64-encoded strings for session management and data encryption. These must be generated before deployment.

```bash
openssl rand -base64 32
```
*Explanation:* Generates a random 32-byte string encoded in Base64. Use this output for `SEMAPHORE_COOKIE_HASH`.

```bash
openssl rand -base64 32
```
*Explanation:* Generates another random 32-byte string encoded in Base64. Use this output for `SEMAPHORE_ACCESS_KEY_ENCRYPTION`.

**Note:** Save these values securely. Changing them after initial setup will invalidate existing sessions and decrypt stored secrets, causing data loss.

### Create Docker Compose File

Create the `docker-compose.yml` file with the following content. Replace `<COOKIE_HASH>` and `<ACCESS_KEY>` with the values generated above.

```yaml
services:
  semaphore:
    image: semaphoreui/semaphore:latest
    container_name: semaphore-ui
    restart: unless-stopped
    ports:
      - "4000:3000"
    environment:
      SEMAPHORE_DB_DIALECT: sqlite
      SEMAPHORE_DB_PATH: /data/database.sqlite
      SEMAPHORE_ADMIN_NAME: "Admin User"
      SEMAPHORE_ADMIN_LOGIN: "admin"
      SEMAPHORE_ADMIN_PASSWORD: "StrongPassword123!"
      SEMAPHORE_ADMIN_EMAIL: "admin@example.com"
      SEMAPHORE_COOKIE_HASH: "<COOKIE_HASH>"
      SEMAPHORE_ACCESS_KEY_ENCRYPTION: "<ACCESS_KEY>"
    volumes:
      - .//data
      - semaphore_tmp:/tmp/semaphore
    networks:
      - semaphore-net
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/api/ping"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  semaphore_tmp:
    driver: local

networks:
  semaphore-net:
    driver: bridge
```

*Explanation:*
*   `image`: Uses the official Semaphore UI image.
*   `ports`: Maps host port 4000 to container port 3000.
*   `environment`: Configures SQLite database, admin credentials, and security keys.
*   `volumes`:
    *   `.//data`: Bind mount for persistent database storage.
    *   `semaphore_tmp:/tmp/semaphore`: Named volume for temporary files (cache, cloned repos).
*   `healthcheck`: Ensures the service is responsive before marking it as healthy.

### Set Permissions for Data Directory

Ensure the host directory has correct permissions to allow the container to write data.

```bash
chmod 777 /opt/semaphore-ui/data
```
*Explanation:* Grants read, write, and execute permissions to all users for the data directory. This resolves common "Permission Denied" errors when the container runs as a non-root user. For stricter security, use `chown` with the specific UID/GID of the container user.

### Deploy Semaphore UI

Start the Semaphore UI service in detached mode.

```bash
docker compose up -d
```
*Explanation:* Pulls the image, creates the container, and starts it in the background.

### Verify Deployment

Check the container status and logs to ensure successful startup.

```bash
docker ps | grep semaphore
```
*Explanation:* Lists running containers filtering for "semaphore". Status should be "Up" and "healthy".

```bash
docker logs -f semaphore-ui
```
*Explanation:* Follows the container logs. Look for "Server is running" and absence of panic errors.

---

## 3. Database Configuration and Initialization

Semaphore supports SQLite, MySQL, and PostgreSQL. This guide focuses on SQLite for simplicity, but notes PostgreSQL for high-concurrency environments.

### Understanding Database Engines

*   **SQLite:** A file-based database stored within the container's mounted volume. Suitable for small teams (<10 users). No separate database server is needed.
*   **PostgreSQL/MySQL:** Client-server databases. Recommended for large teams or high-frequency playbook executions to handle concurrent writes better.

### Initial Admin User Creation

If the database is new, Semaphore automatically creates the admin user defined in `environment` variables. If you encounter "Inserting user failed" errors, it may indicate a partial previous installation.

#### Resolve Initialization Errors

If the container crashes due to database initialization issues, reset the database file.

```bash
docker compose down
rm -f /opt/semaphore-ui/data/database.sqlite
docker compose up -d
```
*Explanation:* Stops the container, removes the corrupted or partially initialized SQLite file, and restarts the container to trigger a fresh initialization.

### Manual Admin User Creation (If Needed)

If automatic creation fails, manually create the admin user inside the container.

```bash
docker exec -it semaphore-ui /bin/bash
```
*Explanation:* Opens an interactive bash shell inside the running Semaphore container.

```bash
semaphore user add --login admin --password StrongPassword123! --email admin@example.com --name "Admin User" --admin
```
*Explanation:* Uses the Semaphore CLI to create an admin user. Replace credentials with your desired values.

```bash
exit
```
*Explanation:* Exits the container shell.

---

## 4. Security Hardening and Key Management

Secure management of SSH keys is critical. Avoid mounting host SSH keys directly into the container. Instead, use Semaphore's built-in Key Store.

### Why Not Mount Host SSH Keys?

Mounting `~/.ssh` from the host exposes private keys to the container filesystem. If the container is compromised, all host keys are stolen. It also lacks audit trails and key rotation capabilities.

### Generate Dedicated SSH Keys for Automation

Create specific SSH key pairs for Semaphore to use on target servers.

```bash
ssh-keygen -t ed25519 -f ~/semaphore-keys/prod-server-key -C "semaphore-prod-key"
```
*Explanation:* Generates an Ed25519 SSH key pair. `-f` specifies the file path, and `-C` adds a comment for identification.

### Add SSH Key to Semaphore Key Store

1.  Log in to Semaphore UI at `http://<HOST_IP>:4000`.
2.  Navigate to **Key Store** > **New Key**.
3.  Select **SSH Private Key**.
4.  Paste the content of the private key file.

```bash
cat ~/semaphore-keys/prod-server-key
```
*Explanation:* Displays the private key content to be copied into the Semaphore UI.

5.  Enter the passphrase if one was set during key generation.
6.  Save the key.

### Configure Target Servers

Copy the public key to target servers' `authorized_keys` file.

```bash
ssh-copy-id -i ~/semaphore-keys/prod-server-key.pub user@target-server-ip
```
*Explanation:* Copies the public key to the specified user's home directory on the target server, enabling passwordless SSH access.

### Restrict Key Usage (Advanced Security)

Edit the `authorized_keys` file on the target server to restrict commands.

```text
command="/usr/bin/sudo /usr/bin/apt update",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty ssh-ed25519 AAAA... semaphore-prod-key
```
*Explanation:* Adds restrictions to the SSH key entry. Only the specified command (`apt update`) can be executed. Port forwarding and other features are disabled, enhancing security.

---

## 5. Production-Grade Storage Integration (CephFS)

For production environments, integrate CephFS to ensure data durability and scalability.

### Mount CephFS on Host

Mount the Ceph filesystem to a local directory on the Docker host.

```bash
mkdir -p /mnt/ceph-storage/semaphore-data
mount -t ceph <CEPH_MON_IP>:6789:/ /mnt/ceph-storage -o name=admin,secretfile=/etc/ceph/admin.secret
```
*Explanation:* Mounts the Ceph filesystem. Replace `<CEPH_MON_IP>` with your Ceph monitor IP. Ensure `/etc/ceph/admin.secret` contains the Ceph admin key.

### Update Docker Compose for CephFS

Modify the `volumes` section in `docker-compose.yml` to point to the Ceph mount point.

```yaml
    volumes:
      - /mnt/ceph-storage/semaphore-/data
      - semaphore_tmp:/tmp/semaphore
```
*Explanation:* Changes the bind mount source from local disk to the Ceph-mounted directory. Data written to `/data` in the container is now stored on Ceph.

### Verify Ceph Integration

Write a test file to the data directory and verify its presence on Ceph.

```bash
echo "test" > /mnt/ceph-storage/semaphore-data/test.txt
ls /mnt/ceph-storage/semaphore-data/
```
*Explanation:* Creates a test file and lists the directory contents to confirm write access to Ceph.

---

## 6. Operational Workflow: Inventory, Keys, and Playbooks

This section covers the daily operations of managing Ansible automation through Semaphore UI.

### Create an Inventory

1.  Navigate to **Inventory** > **New Inventory**.
2.  Select **Static**.
3.  Define hosts and groups.

```ini
[webservers]
192.168.1.10 ansible_user=ubuntu
192.168.1.11 ansible_user=ubuntu

[dbservers]
192.168.1.20 ansible_user=ubuntu
```
*Explanation:* Defines two groups, `webservers` and `dbservers`, with their respective IP addresses and remote users.

4.  Associate the previously created SSH Key with this inventory.
5.  Save the inventory.

### Add a Git Repository

1.  Navigate to **Repositories** > **New Repository**.
2.  Enter the Git URL of your Ansible playbooks repository.
3.  If the repository is private, select the SSH Key used for Git access.
4.  Save the repository.

### Create a Task Template

1.  Navigate to **Task Templates** > **New Template**.
2.  Name the template (e.g., "Update Web Servers").
3.  Select the **Playbook Filename** (e.g., `site.yml`).
4.  Select the **Inventory** created earlier.
5.  Select the **Repository**.
6.  Optionally, add extra variables or environment settings.
7.  Save the template.

### Execute a Playbook

1.  Go to **Task Templates**.
2.  Click **Run** next to the desired template.
3.  Monitor the real-time output in the dashboard.
4.  Review the history for success/failure status.

---

## 7. Backup, Maintenance, and Disaster Recovery

Regular backups and maintenance ensure system reliability.

### Backup Strategy

Since SQLite is used, back up the database file regularly.

```bash
cp /mnt/ceph-storage/semaphore-data/database.sqlite /opt/backups/semaphore-db-$(date +%F).sqlite
```
*Explanation:* Copies the SQLite database file to a backup location with a timestamped filename.

### Ceph Snapshot Backup

For CephFS, take snapshots for point-in-time recovery.

```bash
ceph fs subvolume snapshot create cephfs semaphore_subvol backup-$(date +%Y-%m-%d)
```
*Explanation:* Creates a snapshot of the Ceph subvolume containing Semaphore data. Replace `cephfs` and `semaphore_subvol` with your actual filesystem and subvolume names.

### Restore from Backup

To restore, stop the container, replace the database file, and restart.

```bash
docker compose down
cp /opt/backups/semaphore-db-2026-05-28.sqlite /mnt/ceph-storage/semaphore-data/database.sqlite
docker compose up -d
```
*Explanation:* Restores the database from a specific backup file.

### Update Semaphore UI

Pull the latest image and recreate the container.

```bash
docker compose pull
docker compose up -d
```
*Explanation:* Downloads the latest Semaphore UI image and restarts the container with the new version. Data persists due to volume mounts.

---

## 8. Troubleshooting Common Issues

### Permission Denied Errors

**Symptom:** Logs show `mkdir: can't create directory '/data/database.sqlite': Permission denied`.

**Solution:**
```bash
chmod 777 /opt/semaphore-ui/data
docker compose restart semaphore-ui
```
*Explanation:* Fixes directory permissions to allow the container user to write data.

### Invalid Base64 Key Errors

**Symptom:** Panic error: `access_key_encryption must be a valid base64 string`.

**Solution:**
Regenerate keys using `openssl rand -base64 32` and update `docker-compose.yml`. Ensure no extra spaces or characters are included.

### Database Migration Failures

**Symptom:** Logs show migration errors or "Inserting user failed".

**Solution:**
Delete the database file and restart to reinitialize.
```bash
rm -f /opt/semaphore-ui/data/database.sqlite
docker compose up -d
```

### PRO Feature Warnings

**Symptom:** UI displays "Secret storages are only available for PRO users".

**Explanation:** This is a standard message for advanced features not included in the open-source version. Ignore it if using standard Key Store and Inventory features, which are fully functional in the free tier.

---

## 9. Cleanup and Uninstallation

To completely remove Semaphore UI and its data.

### Stop and Remove Containers

```bash
docker compose down
```
*Explanation:* Stops and removes the Semaphore container and associated network.

### Remove Volumes

```bash
docker volume rm semaphore-ui_semaphore_tmp
```
*Explanation:* Removes the named volume for temporary data.

### Delete Persistent Data

```bash
rm -rf /opt/semaphore-ui
```
*Explanation:* Deletes the project directory, including the SQLite database and configuration files. If using Ceph, unmount and clean the Ceph directory separately.

### Remove Images

```bash
docker rmi semaphoreui/semaphore:latest
```
*Explanation:* Removes the Semaphore UI Docker image from the host to free up space.

---

## File Name Suggestion

`semaphore-ui-production-deployment-guide.md`