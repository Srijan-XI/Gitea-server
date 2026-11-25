# Gitea Self-Hosted Git Server

Complete Technical Documentation & Administration Guide

Document Type: Server Administration & Technical Reference
Platform: Debian GNU/Linux 13 (Trixie)
Git Platform: Gitea 28.0.0
Architecture: Self-hosted, private Git server
Environment: VMware Virtual Lab
Status: Installed and operational; some future configurations remain planned.

---

1. Project Overview

1.1 Introduction

Gitea is a lightweight, self-hosted Git service that provides functionality similar to GitHub and GitLab.

It allows developers to host repositories, manage source code, collaborate on projects, track changes, and manage software development workflows without depending entirely on third-party Git hosting services.

This implementation uses a dedicated Debian 13 virtual machine running Gitea.

The primary objectives are:

- Host private Git repositories.
- Access repositories through a web interface.
- Push and pull source code using Git CLI.
- Manage multiple projects centrally.
- Maintain control over source code and repository data.
- Support development workflows across multiple machines.
- Provide a foundation for future CI/CD and automated code-review workflows.

1.2 Server Information

Parameter| Configuration
Server name| gitserver
Operating system| Debian GNU/Linux 13 (Trixie)
Architecture| x86_64
Gitea version| 28.0.0
Go version used to build| go1.27.1
Installation type| Binary installation
Binary location| "/usr/local/bin/gitea"
Service user| "git"
Service group| "git"
User home| "/home/git"
Database| SQLite
Configuration| "/etc/gitea/app.ini"
Data directory| "/var/lib/gitea"
Web interface port| TCP 3000
Git transport| SSH
Network address| "192.168.60.113"
Internal hostname| "git.home.arpa"

Important: These are the recorded configuration details. Confirm the live configuration before changing ports, authentication, or SSH settings.

---

2. Network Architecture

2.1 VMware Network

Gitea operates inside the existing VMware laboratory network.

Component| IP address
pfSense firewall| "192.168.60.101"
Main Debian server| "192.168.60.110"
NetGuardian server| "192.168.60.112"
Gitea server| "192.168.60.113"
Windows VMnet1 adapter| "192.168.60.1"

Network configuration:

- Network: "192.168.60.0/24"
- Gateway: "192.168.60.101"
- DNS server: "192.168.60.101"
- Virtual network: VMware VMnet1 Host-only

The Gitea server must remain within the existing lab network.

2.2 DNS Configuration

The internal DNS record is configured through pfSense.

Record| Value
Hostname| "git"
Domain| "home.arpa"
Fully qualified hostname| "git.home.arpa"
IP address| "192.168.60.113"

Test DNS resolution:

nslookup git.home.arpa 192.168.60.101

Expected result:

Name: git.home.arpa
Address: 192.168.60.113

Access the web interface:

http://git.home.arpa:3000

Alternatively:

http://192.168.60.113:3000

Internal DNS resolution is provided by pfSense. Public DNS resolution is not expected for this private hostname.

---

3. Server Architecture

3.1 Software Components

The Gitea deployment consists of:

1. Debian Linux – Operating system.
2. Gitea – Git hosting application.
3. SQLite – Database engine.
4. OpenSSH – SSH connectivity for Git operations, subject to the active Gitea SSH configuration.
5. systemd – Service and startup management.
6. VMware – Virtualization platform.
7. pfSense – Network firewall and internal DNS.

3.2 Architecture Diagram

                 Developer Computer
                         |
              Web Browser / Git CLI
                         |
                         |
                 VMware VMnet1
                 192.168.60.0/24
                         |
                         |
                  pfSense Firewall
                  192.168.60.101
                         |
                         |
                  Gitea Server
                  192.168.60.113
                         |
             +-----------+-----------+
             |           |           |
           Gitea       SQLite      OpenSSH
          Web UI       Database     Git SSH
           :3000       app.db      :22*
             |           |           |
             +-----------+-----------+
                         |
                  Git Repositories
                  /var/lib/gitea

*TCP 22 represents the intended SSH transport. Verify whether Git SSH is handled by system OpenSSH or Gitea's built-in SSH server.

---

4. Linux User and Directory Structure

4.1 Gitea Service Account

Gitea runs under a dedicated Linux account named "git".

Recorded configuration:

Username: git
Group: git
Home: /home/git
Shell: /bin/bash
UID: 1000
GID: 1000

Verify the account:

id git

Check its home directory:

getent passwd git

4.2 Directory Structure

The important directories are:

Directory| Purpose
"/usr/local/bin/gitea"| Gitea executable
"/etc/gitea/"| Configuration files
"/etc/gitea/app.ini"| Main configuration
"/var/lib/gitea/"| Application data
"/var/lib/gitea/data/"| Application data and SQLite database
"/var/lib/gitea/repositories/"| Git repositories
"/var/lib/gitea/log/"| Application logs
"/home/git/"| Git service account home

Some subdirectories may be created automatically depending on the configuration.

4.3 Permissions

The application data belongs to the "git" account.

Check permissions:

ls -ld /var/lib/gitea
ls -ld /etc/gitea
ls -l /usr/local/bin/gitea

Expected ownership:

/var/lib/gitea     git:git
/etc/gitea        root:git

Recommended configuration directory permissions:

sudo chown root:git /etc/gitea
sudo chmod 750 /etc/gitea

Protect the configuration file:

sudo chmod 640 /etc/gitea/app.ini

Do not recursively change ownership of system directories without checking their intended ownership first.

---

5. Gitea Installation

5.1 Update Debian

sudo apt update
sudo apt upgrade -y

Install required utilities:

sudo apt install -y \
  git \
  wget \
  curl \
  ca-certificates \
  sqlite3 \
  openssh-server

5.2 Create the Git Account

Check whether the account already exists:

id git

If it does not exist, create it:

sudo adduser \
  --system \
  --shell /bin/bash \
  --gecos 'Git Version Control' \
  --group \
  --disabled-password \
  --home /home/git \
  git

Do not recreate the existing account.

5.3 Download Gitea

The installed version is Gitea 28.0.0.

Download the appropriate binary from the official release source:

https://github.com/go-gitea/gitea/releases

The binary must be installed at:

/usr/local/bin/gitea

Set executable permissions:

sudo chmod 755 /usr/local/bin/gitea

Verify:

gitea --version

Expected version:

Gitea version 28.0.0

5.4 Create Application Directories

sudo mkdir -p /var/lib/gitea/data
sudo mkdir -p /var/lib/gitea/repositories
sudo mkdir -p /var/lib/gitea/log
sudo mkdir -p /etc/gitea

Assign ownership:

sudo chown -R git:git /var/lib/gitea
sudo chown root:git /etc/gitea

---

6. Gitea Configuration

6.1 Main Configuration File

The primary configuration file is:

/etc/gitea/app.ini

It controls:

- Application name.
- Server hostname.
- HTTP port.
- Database configuration.
- Repository storage.
- Logging.
- Authentication.
- SSH configuration.
- Security settings.

Inspect the current configuration:

sudo nano /etc/gitea/app.ini

Do not replace the existing configuration blindly.

6.2 SQLite Database

The deployment uses SQLite.

SQLite is suitable for this relatively small, single-server installation because it does not require a separate database server.

Advantages:

- Low resource consumption.
- Simple administration.
- No separate database service.
- Easy database backup.
- Suitable for small private development environments.

The database location is controlled by the "[database]" section in "app.ini".

Verify the configured path before performing database maintenance.

6.3 Important Configuration Sections

The following is a reference structure, not a replacement for the existing production configuration.

[server]
DOMAIN = git.home.arpa
HTTP_PORT = 3000
ROOT_URL = http://git.home.arpa:3000/

[database]
DB_TYPE = sqlite3

[repository]
ROOT = /var/lib/gitea/repositories

[log]
MODE = console
LEVEL = Info

Other configuration sections may be present.

Preserve existing authentication, security, and SSH settings.

---

7. systemd Service Management

7.1 Service Purpose

Gitea runs as a systemd-managed service.

This provides:

- Automatic startup after boot.
- Background application execution.
- Service monitoring.
- Restart management.
- Centralized logs.

7.2 Service Configuration

The recorded service command is:

/usr/local/bin/gitea web --config /etc/gitea/app.ini

The service runs using the "git" account.

The service unit is normally located at:

/etc/systemd/system/gitea.service

A reference service definition:

[Unit]
Description=Gitea Git Service
After=network.target

[Service]
Type=simple
User=git
Group=git
WorkingDirectory=/var/lib/gitea
ExecStart=/usr/local/bin/gitea web --config /etc/gitea/app.ini
Restart=always
RestartSec=2

[Install]
WantedBy=multi-user.target

Compare this reference with the actual installed service before making changes.

7.3 Enable Automatic Startup

sudo systemctl daemon-reload
sudo systemctl enable gitea
sudo systemctl start gitea

7.4 Service Management Commands

Check status:

sudo systemctl status gitea

Start:

sudo systemctl start gitea

Stop:

sudo systemctl stop gitea

Restart:

sudo systemctl restart gitea

Enable startup:

sudo systemctl enable gitea

Disable startup:

sudo systemctl disable gitea

---

8. Accessing Gitea

8.1 Web Interface

Open:

http://git.home.arpa:3000

Alternative:

http://192.168.60.113:3000

The web interface allows administrators to:

- Create repositories.
- Manage users.
- Configure permissions.
- View commits.
- Browse source code.
- Manage issues.
- Review pull requests.
- Configure SSH public keys.

8.2 Git CLI Access

Git can communicate with Gitea through SSH or HTTP(S).

For SSH, first check:

sudo systemctl status ssh

Check listening ports:

sudo ss -ltnp

The SSH URL depends on the configured SSH server and port.

Example:

git clone ssh://git@git.home.arpa:22/OWNER/REPOSITORY.git

Replace "OWNER" and "REPOSITORY" with actual values.

8.3 SSH Key Authentication

Generate an Ed25519 key:

ssh-keygen -t ed25519 -C "developer-key"

Display the public key:

cat ~/.ssh/id_ed25519.pub

In Gitea:

1. Log in.
2. Open account settings.
3. Navigate to SSH/GPG Keys.
4. Select Add Key.
5. Paste the public key.
6. Save.

Never upload the private key ("id_ed25519") to Gitea.

Test the SSH connection using the configured SSH hostname and port:

ssh -T git@git.home.arpa

---

9. Repository Management

9.1 Creating a Repository

1. Log in to Gitea.
2. Select the New Repository option.
3. Enter a repository name.
4. Choose private visibility.
5. Configure the README and Git ignore options if needed.
6. Create the repository.

For private projects, verify that repository visibility is set to Private.

9.2 Uploading an Existing Project

Inside the project directory:

git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin \
  ssh://git@git.home.arpa:22/OWNER/REPOSITORY.git
git push -u origin main

9.3 Cloning a Repository

git clone \
  ssh://git@git.home.arpa:22/OWNER/REPOSITORY.git

9.4 Updating a Repository

git add .
git commit -m "Update project"
git push

Retrieve remote changes:

git pull

9.5 Repository Security

Before uploading code:

- Remove passwords.
- Remove API keys.
- Exclude private certificates.
- Exclude database credentials.
- Avoid committing ".env" files.
- Review configuration files.
- Add appropriate ".gitignore" rules.

---

10. Backup and Recovery Strategy

Backups are essential because Gitea stores more than just Git source code.

A complete backup must include:

1. SQLite database.
2. Git repositories.
3. Application configuration.
4. Attachments and uploaded files.
5. Other application data.
6. Relevant custom files.

10.1 Backup Procedure

Create a backup directory:

sudo mkdir -p /var/backups/gitea

Stop Gitea to create a consistent filesystem-level backup:

sudo systemctl stop gitea

Create a compressed backup:

sudo tar -czpf \
  /var/backups/gitea/gitea-backup.tar.gz \
  /etc/gitea \
  /var/lib/gitea

Start Gitea again:

sudo systemctl start gitea

Check the backup:

sudo ls -lh /var/backups/gitea/

This backup does not include the Gitea executable or the systemd unit. Preserve those separately as part of disaster-recovery documentation.

10.2 Backup Verification

Inspect archive contents:

sudo tar -tzf \
  /var/backups/gitea/gitea-backup.tar.gz

Periodically perform a test restoration on a separate machine or isolated test instance.

A backup that has never been restored is not a verified recovery solution.

10.3 Restoration Procedure

Stop the service:

sudo systemctl stop gitea

Restore the archive:

sudo tar -xzpf \
  /var/backups/gitea/gitea-backup.tar.gz \
  -C /

Check ownership and permissions.

Start the service:

sudo systemctl start gitea

Verify:

sudo systemctl status gitea

Check repository access and database integrity after recovery.

Important: Restore onto a compatible Gitea version. Do not start an older binary against a database that has already been upgraded without checking Gitea's compatibility requirements.

---

11. Updating Gitea

Gitea must be updated carefully to prevent database or repository problems.

11.1 Before Updating

1. Check the current version.
2. Read the official release notes.
3. Confirm compatibility.
4. Back up "/etc/gitea".
5. Back up "/var/lib/gitea".
6. Preserve the current binary.

Check the version:

gitea --version

11.2 Update Procedure

Stop Gitea:

sudo systemctl stop gitea

Preserve the existing binary:

sudo cp /usr/local/bin/gitea \
  /usr/local/bin/gitea.previous

Download the new release from:

https://github.com/go-gitea/gitea/releases

Verify the downloaded release checksum or signature against the official release information.

Install the new binary:

sudo install -m 755 gitea \
  /usr/local/bin/gitea

Start Gitea:

sudo systemctl start gitea

Verify:

sudo systemctl status gitea
gitea --version

Review the logs for database migrations or other errors.

Never skip the backup step before an application upgrade.

---

12. Troubleshooting

12.1 Gitea Is Not Starting

Check:

sudo systemctl status gitea

Read logs:

sudo journalctl -u gitea -n 100 --no-pager

Common causes:

- Incorrect configuration.
- Missing directories.
- Incorrect file permissions.
- Invalid database path.
- Port conflict.
- Incorrect binary permissions.

12.2 Web Interface Is Not Accessible

Check whether Gitea is listening:

sudo ss -ltnp | grep 3000

Test locally:

curl -I http://127.0.0.1:3000

Test DNS:

nslookup git.home.arpa 192.168.60.101

Test network connectivity:

ping 192.168.60.113

If the application works locally but not remotely, inspect its bind address and the firewall rules.

12.3 Git SSH Connection Fails

Check SSH:

sudo systemctl status ssh

Check the configured port:

sudo ss -ltnp

Verify the public key is registered in the correct Gitea account.

Also verify that the SSH URL uses the correct port and hostname.

12.4 Permission Problems

Inspect:

ls -ld /var/lib/gitea
ls -ld /etc/gitea
ls -l /etc/gitea/app.ini

Check the service user:

ps aux | grep '[g]itea'

Do not resolve permission problems by applying "chmod 777".

12.5 Database Problems

Locate the configured SQLite database:

sudo grep -A10 '^\[database\]' /etc/gitea/app.ini

Use SQLite integrity checking after stopping Gitea:

sudo sqlite3 /path/to/app.db "PRAGMA integrity_check;"

The expected healthy result is:

ok

Replace the example database path with the actual configured location.

---

13. Security Recommendations

13.1 Access Control

- Use private repositories for confidential projects.
- Create separate accounts for developers.
- Apply the principle of least privilege.
- Avoid sharing administrator credentials.
- Use SSH key authentication.
- Protect the Gitea configuration file.

13.2 Firewall

Restrict access to the required services.

Port| Protocol| Purpose
3000| TCP| Gitea web interface
22| TCP| SSH, if using system OpenSSH
Custom port| TCP| Only if configured for another SSH service

Do not expose unnecessary services.

13.3 HTTPS

The current documented web access is HTTP.

HTTPS is a planned improvement.

For future HTTPS deployment:

- Configure a trusted internal certificate authority.
- Install trusted CA certificates on client machines.
- Configure the reverse proxy or Gitea TLS settings.
- Update the Gitea "ROOT_URL".
- Redirect HTTP to HTTPS where appropriate.

Do not treat the existing HTTP deployment as HTTPS-secured.

13.4 Remote Access

For remote browser and terminal Git access, a private VPN approach using Tailscale was previously considered.

The intended access model is:

- Browser: "http://TAILSCALE-IP:3000"
- Git SSH: "ssh://git@TAILSCALE-IP:22/OWNER/REPOSITORY.git"

This is a planned remote-access configuration, not a confirmed completed deployment.

Avoid exposing Gitea directly to the public internet through router port forwarding unless a separate, properly secured deployment design is implemented.

---

14. Maintenance Checklist

Daily / When Needed

- Verify that Gitea is running.
- Check repository access.
- Review application errors if something fails.

Weekly

- Verify backup creation.
- Check available disk space.
- Review system logs.
- Check for unexpected login activity.

Monthly

- Review Gitea releases.
- Review account permissions.
- Verify backup integrity.
- Test repository cloning and pushing.
- Review SSH keys.
- Review firewall rules.

Before Every Upgrade

- Read release notes.
- Create a full backup.
- Preserve the current binary.
- Verify the restoration procedure.
- Schedule a maintenance window.

---

15. Useful Administration Commands

Task| Command
Check Gitea version| "gitea --version"
Check service| "sudo systemctl status gitea"
Start service| "sudo systemctl start gitea"
Stop service| "sudo systemctl stop gitea"
Restart service| "sudo systemctl restart gitea"
Enable startup| "sudo systemctl enable gitea"
View logs| "sudo journalctl -u gitea -f"
Check port| "sudo ss -ltnp"
Check IP| "ip -br addr"
Test web service| "curl -I http://127.0.0.1:3000"
Check DNS| "nslookup git.home.arpa"
Check SSH| "sudo systemctl status ssh"
Check disk| "df -h"
Check directory size| "sudo du -sh /var/lib/gitea"

---

16. Future Improvements

The following are potential improvements, not features currently confirmed as implemented.

Feature| Purpose| Status
HTTPS| Secure browser communication| Planned
Private VPN access| Secure remote access| Planned
Automated backups| Reduce manual administration| To implement
Backup restoration testing| Validate disaster recovery| To implement
Repository mirroring| Selective external repository synchronization| Planned
CI/CD integration| Automate build and deployment| Future
Code quality scanning| Identify problems before merging| Future
Secret scanning| Detect accidentally committed credentials| Future
Centralized monitoring| Monitor server health| Future

---

17. Official Documentation and Resources

- Gitea Official Website: https://about.gitea.com/
- Gitea Documentation: https://docs.gitea.com/
- Gitea Installation: https://docs.gitea.com/installation/install-from-binary
- Gitea Configuration: https://docs.gitea.com/administration/config-cheat-sheet
- Gitea Backup and Restore: https://docs.gitea.com/administration/backup-and-restore
- Gitea Upgrade Guide: https://docs.gitea.com/administration/upgrade
- Gitea Releases: https://github.com/go-gitea/gitea/releases
- Debian Documentation: https://www.debian.org/doc/

---

18. Final Server Reference

Category| Value
Server| Gitea
Hostname| gitserver
DNS| git.home.arpa
IP address| 192.168.60.113
Operating system| Debian 13 Trixie
Application version| Gitea 28.0.0
Database| SQLite
Application binary| /usr/local/bin/gitea
Configuration| /etc/gitea/app.ini
Data| /var/lib/gitea
Service account| git
We