# Home Lab Disaster Recovery

## Overview

Complete disaster recovery and backup solution for your home lab infrastructure using git-based configuration management with profile support.

**Last Updated:** December 2025  
**System:** Profile-based config backup with git-crypt encryption  
**Servers Covered:** 3 (NAS, DC, ML)

---

## 📚 Documentation Structure

### Core System Documentation
- **PROFILE_ARCHITECTURE.md** - How the profile system works
- **GIT_BACKUP_SETUP_GUIDE.md** - Complete setup instructions
- **GIT_BACKUP_QUICK_REFERENCE.md** - Daily usage cheatsheet

### Server-Specific Plans
- **NAS_Disaster_Recovery_Plan_v2.md** - Network Attached Storage
- **DC_Disaster_Recovery_Plan_v1.md** - Domain Controller
- **ML_GPU_Disaster_Recovery_Plan_v1.md** - ML/GPU Server
- **ML_Testing_Guide.md** - Testing procedures (start here!)

---

## Introduction

This backup system uses Git to version control server configurations with encryption support via git-crypt. It's designed to:

- Track all configuration changes over time
- Encrypt sensitive data at rest
- Support multiple servers in one repository
- Enable easy disaster recovery
- Facilitate configuration auditing and rollback

### What Gets Backed Up

- **System configs**: `/etc` directory (selective)
- **Custom scripts**: `/usr/local/bin` scripts
- **Systemd services**: Custom unit files
- **Docker configs**: docker-compose.yml and .env files
- **Cron jobs**: System and user crontabs
- **Documentation**: System information and metadata

### What's NOT Backed Up

- User data and databases
- Log files (structure preserved, content excluded)
- Large binary files (ISOs, disk images)
- Runtime data and caches

---

## Installation

### Prerequisites

```bash
# Install required packages
sudo apt update
sudo apt install -y git git-crypt gnupg

# Optional but recommended
sudo apt install -y vim tree
```

### Download Scripts

```bash
# Create scripts directory
sudo mkdir -p /opt/scripts
cd /opt/scripts

# Download the backup scripts
sudo curl -O https://your-repo/git-config-backup.sh
sudo curl -O https://your-repo/restore-from-git.sh

# Make them executable
sudo chmod +x git-config-backup.sh restore-from-git.sh

# Optional: Create symbolic links
sudo ln -s /opt/scripts/git-config-backup.sh /usr/local/bin/config-backup
sudo ln -s /opt/scripts/restore-from-git.sh /usr/local/bin/config-restore
```

---

## Initial Setup

### Step 1: Initialize Repository

```bash
# Initialize the repository structure
sudo /opt/scripts/git-config-backup.sh --init

# This creates:
# - /opt/homelab-configs/
# - Git repository structure
# - Pre-commit hooks
# - Documentation templates
```

### Step 2: Configure Git

```bash
cd /opt/homelab-configs

# Set git user (for commits)
git config user.name "System Backup"
git config user.email "backup@yourdomain.com"

# Optional: Set default branch name
git config init.defaultBranch main
```

### Step 3: Initialize git-crypt (Encryption)

```bash
cd /opt/homelab-configs

# Initialize git-crypt
git-crypt init

# Export the symmetric key (STORE THIS SECURELY!)
git-crypt export-key ~/git-crypt-key

# Move key to secure location (encrypted USB, password manager, etc.)
# Example: Copy to encrypted USB drive
sudo cp ~/git-crypt-key /media/secure-usb/homelab-git-crypt-key
sudo chmod 600 /media/secure-usb/homelab-git-crypt-key

# Remove local copy after securing it
shred -u ~/git-crypt-key
```

**⚠️ CRITICAL**: The git-crypt key is needed to decrypt your backups. Without it, encrypted files are unrecoverable. Store it in multiple secure locations:
- Encrypted USB drive (off-site)
- Password manager (encrypted notes)
- Secure cloud storage (encrypted)
- Physical safe (printed and secured)

### Step 4: Optional - Add GPG Key

For team access, you can use GPG keys instead of the symmetric key:

```bash
# Generate GPG key if you don't have one
gpg --full-generate-key

# Add your GPG key to git-crypt
cd /opt/homelab-configs
git-crypt add-gpg-user YOUR_GPG_KEY_ID

# Add team member's GPG key
git-crypt add-gpg-user TEAM_MEMBER_GPG_KEY_ID
```

### Step 5: Configure Git Remote (Optional)

If you want to push backups to a remote git server:

```bash
cd /opt/homelab-configs

# Add remote repository
git remote add origin git@github.com:youruser/homelab-configs.git
# OR
git remote add origin https://gitea.yourdomain.com/youruser/homelab-configs.git

# Set upstream
git branch --set-upstream-to=origin/main main

# Test push
git push -u origin main
```

**Security Note**: If using a cloud git service (GitHub, GitLab, etc.):
- Sensitive files are encrypted by git-crypt
- Only encrypted data leaves your network
- Private repository is still recommended
- Consider self-hosted git (Gitea, GitLab) for maximum control

### Step 6: First Backup

```bash
# Run first backup
sudo /opt/scripts/git-config-backup.sh --backup

# Verify backup was created
cd /opt/homelab-configs
git log
tree -L 2

# Check encrypted files are actually encrypted
cat nas/etc/samba/smb.conf
# Should show binary garbage if encrypted correctly
```

---

## Usage

### Basic Backup Commands

```bash
# Backup current server
sudo config-backup --backup

# Backup and push to remote
sudo config-backup --push

# Backup specific server (if running centrally)
sudo config-backup --backup --server web-server

# Use custom repository location
sudo config-backup --backup --repo /mnt/storage/configs
```

### View Backup History

```bash
cd /opt/homelab-configs

# View all commits
git log --oneline

# View commits for specific server
git log --oneline -- nas/

# Show what changed in last backup
git show HEAD

# Show changes to specific file
git log -p -- nas/etc/samba/smb.conf

# Compare two backups
git diff HEAD~5 HEAD -- nas/etc/samba/smb.conf
```

### Restore Configurations

```bash
# Dry run (see what would be restored)
sudo config-restore --dry-run

# Restore all configurations
sudo config-restore

# Restore specific category
sudo config-restore --category docker
sudo config-restore --category etc
sudo config-restore --category systemd

# Restore from specific server backup
sudo config-restore --server nas

# Force restore without prompts
sudo config-restore --force
```

### Restore Specific File

```bash
# Navigate to repository
cd /opt/homelab-configs

# Copy specific file
sudo cp nas/etc/samba/smb.conf /etc/samba/smb.conf
sudo chmod 644 /etc/samba/smb.conf

# Restart service
sudo systemctl restart smbd
```

---

## Security Considerations

### Understanding Encryption

**git-crypt encrypts files at rest in the repository.**

- Files matching patterns in `.gitattributes` are encrypted
- Encryption happens automatically on `git add`
- Decryption happens automatically when repository is unlocked
- Unencrypted files are never stored in git history

### What Should Be Encrypted

**Always encrypt:**
- SSH private keys (if you choose to back them up)
- SSL/TLS certificates and private keys
- Database passwords
- API keys and tokens
- Docker `.env` files with secrets
- Samba configuration (contains LDAP passwords)
- SSSD configuration (contains LDAP credentials)
- Any file with "password", "secret", "key" in the name

**Don't need encryption:**
- Network configuration files (no secrets)
- Systemd unit files (no secrets)
- Public SSL certificates
- Cron job schedules
- Package lists
- System information files

### Verifying Encryption

```bash
cd /opt/homelab-configs

# Lock the repository (encrypt everything)
git-crypt lock

# Try to view encrypted file (should see binary data)
cat nas/etc/samba/smb.conf
# Output: binary garbage = encrypted correctly ✓

# Unlock repository
git-crypt unlock /path/to/git-crypt-key

# Now file should be readable
cat nas/etc/samba/smb.conf
# Output: readable configuration ✓
```

### Files to NEVER Commit

Even with encryption, some files should NEVER be in git:

- `/etc/shadow` - User password hashes (use `.gitignore`)
- Database dumps (too large, changes too frequently)
- Large binary files (use separate backup system)
- Temporary files and caches

These are already excluded in the default `.gitignore`.

### Access Control

```bash
# Restrict repository access
sudo chown -R root:root /opt/homelab-configs
sudo chmod 750 /opt/homelab-configs

# Only specific users can access
sudo usermod -aG root your-admin-user

# Protect the git-crypt key
chmod 600 /path/to/git-crypt-key
```

---

## Multi-Server Setup

### Adding a New Server

**On the new server:**

```bash
# Install git and git-crypt
sudo apt install git git-crypt

# Clone existing repository
sudo git clone https://your-git-server/homelab-configs.git /opt/homelab-configs
cd /opt/homelab-configs

# Unlock with your git-crypt key
sudo git-crypt unlock /path/to/git-crypt-key

# Copy the backup script
sudo cp scripts/git-config-backup.sh /opt/scripts/
sudo chmod +x /opt/scripts/git-config-backup.sh

# Run first backup for this server
sudo /opt/scripts/git-config-backup.sh --backup --server new-server-name

# Commit and push
cd /opt/homelab-configs
git push origin main
```

### Central Backup Server Approach

Set up one server to backup all others via SSH:

```bash
# On central backup server
cat > /opt/scripts/backup-all-servers.sh << 'EOF'
#!/bin/bash

SERVERS=("nas" "web-server" "media-server" "docker-host")

for server in "${SERVERS[@]}"; do
    echo "Backing up: $server"
    ssh root@$server '/opt/scripts/git-config-backup.sh --backup'
done

# Pull all changes to central repo
cd /opt/homelab-configs
git pull

# Push to remote
git push origin main
EOF

chmod +x /opt/scripts/backup-all-servers.sh
```

### Repository Structure

```
/opt/homelab-configs/
├── nas/                          # NAS server configs
│   ├── etc/
│   ├── docker/
│   └── docs/
├── web-server/                   # Web server configs
│   ├── etc/
│   └── docs/
├── docker-host/                  # Docker host configs
├── scripts/                      # Shared scripts
│   ├── git-config-backup.sh
│   └── restore-from-git.sh
├── .gitattributes               # Encryption rules
├── .gitignore                   # Exclusion rules
└── README.md                    # Documentation
```

---

## Restoration Procedures

### Disaster Recovery Scenario

**Complete server loss with data drives intact:**

```bash
# 1. Install fresh Ubuntu Server

# 2. Install git and git-crypt
sudo apt update
sudo apt install git git-crypt

# 3. Clone configuration repository
sudo git clone https://your-git-server/homelab-configs.git /opt/homelab-configs

# 4. Unlock repository
cd /opt/homelab-configs
sudo git-crypt unlock /path/to/git-crypt-key

# 5. Install restoration script
sudo cp scripts/restore-from-git.sh /opt/scripts/
sudo chmod +x /opt/scripts/restore-from-git.sh

# 6. Dry run to see what will be restored
sudo /opt/scripts/restore-from-git.sh --dry-run

# 7. Perform restoration
sudo /opt/scripts/restore-from-git.sh --server nas

# 8. Review and adjust configs
sudo nano /etc/fstab  # Verify UUIDs match your disks
sudo nano /etc/network/interfaces  # Verify network config

# 9. Restart services
sudo systemctl daemon-reload
sudo systemctl restart sssd smbd nmbd nfs-kernel-server

# 10. Test functionality
testparm -s
showmount -e
docker ps
```

### Rollback to Previous Configuration

```bash
cd /opt/homelab-configs

# View history
git log --oneline -- nas/etc/samba/smb.conf

# Checkout file from specific commit
git checkout abc123 -- nas/etc/samba/smb.conf

# Copy to system
sudo cp nas/etc/samba/smb.conf /etc/samba/smb.conf
sudo systemctl restart smbd

# If happy with rollback, commit it
git add nas/etc/samba/smb.conf
git commit -m "Rollback samba config to working version"
```

### Compare Current vs Backup

```bash
# Compare current system file with backup
diff /etc/samba/smb.conf /opt/homelab-configs/nas/etc/samba/smb.conf

# Or use git
cd /opt/homelab-configs
git diff -- nas/etc/samba/smb.conf
```

---

## Automation

### Daily Backup via Cron

```bash
# Edit root crontab
sudo crontab -e

# Add daily backup at 2 AM
0 2 * * * /opt/scripts/git-config-backup.sh --push >> /var/log/config-backup.log 2>&1

# Or weekly backup every Sunday at 3 AM
0 3 * * 0 /opt/scripts/git-config-backup.sh --push >> /var/log/config-backup.log 2>&1
```

### Systemd Timer (Alternative to Cron)

```bash
# Create timer unit
sudo nano /etc/systemd/system/config-backup.timer

[Unit]
Description=Configuration Backup Timer
Requires=config-backup.service

[Timer]
OnCalendar=daily
OnCalendar=02:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
# Create service unit
sudo nano /etc/systemd/system/config-backup.service

[Unit]
Description=Configuration Backup Service
After=network.target

[Service]
Type=oneshot
ExecStart=/opt/scripts/git-config-backup.sh --push
StandardOutput=journal
StandardError=journal
```

```bash
# Enable and start timer
sudo systemctl daemon-reload
sudo systemctl enable config-backup.timer
sudo systemctl start config-backup.timer

# Check status
sudo systemctl list-timers
sudo systemctl status config-backup.timer
```

### Backup on Configuration Change

Monitor config files and trigger backup:

```bash
# Install inotify tools
sudo apt install inotify-tools

# Create monitoring script
sudo nano /opt/scripts/watch-configs.sh

#!/bin/bash
inotifywait -m -r -e modify,create,delete \
  /etc/samba \
  /etc/sssd \
  /etc/ssh |
while read path action file; do
    echo "$(date): $action on $path$file"
    sleep 60  # Debounce
    /opt/scripts/git-config-backup.sh --backup
done
```

---

## Best Practices

### 1. Regular Testing

```bash
# Monthly: Test restoration in VM
# - Clone repository
# - Unlock with key
# - Restore configurations
# - Verify services start

# Quarterly: Full disaster recovery drill
# - Fresh Ubuntu install in VM
# - Complete restoration procedure
# - Document time and issues
```

### 2. Backup Key Management

**Store git-crypt key in multiple locations:**
- Primary: Encrypted USB drive (off-site)
- Secondary: Password manager
- Tertiary: Secure cloud storage
- Physical: Printed on paper in safe

**Test key access:**
```bash
# Monthly test
git-crypt unlock /backup/location/git-crypt-key
```

### 3. Commit Messages

Use descriptive commit messages:

```bash
# Good
"Updated Samba config to enable audit logging"
"Added fail2ban jail for SSH port 2222"
"Restored known-good SSSD config from 2024-01-15"

# Bad
"changes"
"update"
"fix"
```

### 4. Pre-Change Backups

Before major changes:

```bash
# Backup before change
sudo config-backup --backup

# Make changes
sudo nano /etc/samba/smb.conf

# Test changes
testparm -s

# If good, backup again with descriptive message
sudo config-backup --backup
cd /opt/homelab-configs
git commit --amend -m "Enabled guest access on media share"
```

### 5. Regular Repository Maintenance

```bash
# Monthly: Review repository size
cd /opt/homelab-configs
du -sh .git

# If too large, consider git gc
git gc --aggressive --prune=now

# Remove old branches
git branch -d old-branch-name

# Review what's taking space
git count-objects -vH
```

### 6. Documentation

Update server documentation when making changes:

```bash
cd /opt/homelab-configs/nas/docs

# Document change
cat >> changes.md << EOF
## $(date +%Y-%m-%d) - Samba Guest Access
- Enabled guest access on media share
- Removed password requirement for readonly access
- Updated firewall rules to allow SMB from media network
EOF

git add nas/docs/changes.md
git commit -m "Documented Samba guest access change"
```

---

## Troubleshooting

### Repository Not Found

```bash
# Check repository location
ls -la /opt/homelab-configs/.git

# If missing, initialize
sudo /opt/scripts/git-config-backup.sh --init
```

### Cannot Decrypt Files

```bash
# Check if repository is locked
cd /opt/homelab-configs
git-crypt status

# Unlock with key
git-crypt unlock /path/to/git-crypt-key

# If key is lost, encrypted files are unrecoverable
# You must restore from unencrypted backup
```

### Pre-commit Hook Blocking Commit

```bash
# Review the changes
git diff --cached

# If you're sure there are no secrets:
git commit --no-verify -m "Your message"

# Better: Fix the issue and commit properly
```

### Merge Conflicts

```bash
# If pulling from remote causes conflicts
git pull origin main

# Resolve conflicts in files
git status  # Shows conflicted files

# Edit files to resolve
nano nas/etc/samba/smb.conf

# Mark as resolved
git add nas/etc/samba/smb.conf

# Complete merge
git commit
```

### Large Repository Size

```bash
# Check what's taking space
cd /opt/homelab-configs
git rev-list --objects --all |
  git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' |
  sed -n 's/^blob //p' |
  sort --numeric-sort --key=2 |
  cut -c 1-12,41- |
  $(command -v gnumfmt || echo numfmt) --field=2 --to=iec-i --suffix=B --padding=7 --round=nearest |
  tail -20

# Remove large file from history (careful!)
git filter-branch --tree-filter 'rm -f path/to/large/file' HEAD

# Cleanup
git gc --aggressive --prune=now
```

### Permission Denied

```bash
# Backup script needs root
sudo /opt/scripts/git-config-backup.sh --backup

# Repository permissions
sudo chown -R root:root /opt/homelab-configs
sudo chmod -R 750 /opt/homelab-configs
```

### Git-crypt Not Encrypting

```bash
# Verify .gitattributes exists
cat /opt/homelab-configs/.gitattributes

# Check encryption status
cd /opt/homelab-configs
git-crypt status

# Re-encrypt specific file
git rm --cached nas/etc/samba/smb.conf
git add nas/etc/samba/smb.conf
git commit -m "Re-encrypt smb.conf"
```

---

## Additional Resources

### Useful Commands Reference

```bash
# View encrypted files in unlocked repo
git-crypt status

# Export new key
git-crypt export-key /new/location/key

# Add GPG user
git-crypt add-gpg-user USER_GPG_ID

# View git-crypt users
git-crypt list-gpg-users

# Force unlock
git-crypt unlock --force /path/to/key

# View commit that introduced file
git log --follow -- path/to/file

# Search git history for pattern
git log -S "search term" --source --all

# Show files in commit
git show --pretty="" --name-only COMMIT_HASH
```

### Recommended Tools

- **lazygit**: Terminal UI for git
- **tig**: Text-mode interface for git
- **diff-so-fancy**: Better diff output
- **git-crypt**: Transparent file encryption

### Related Documentation

- Git Documentation: https://git-scm.com/doc
- git-crypt: https://github.com/AGWA/git-crypt
- GPG: https://gnupg.org/documentation/

---

## Support and Contributing

### Getting Help

1. Check troubleshooting section above
2. Review git and git-crypt documentation
3. Check `/var/log/config-backup.log` for errors
4. Review git commit history for similar issues

### Improving the Scripts

To modify the backup script:

```bash
# Edit script
sudo nano /opt/scripts/git-config-backup.sh

# Test changes
sudo /opt/scripts/git-config-backup.sh --backup

# Commit improved script
cd /opt/homelab-configs
cp /opt/scripts/git-config-backup.sh scripts/
git add scripts/git-config-backup.sh
git commit -m "Improved backup script: added feature X"
git push
```

# Attribution

This project was built using AI-assisted development with Claude. I told it what to build and the AI did a lot of typing. Commits with AI contributions are marked appropriately because I'm not going to pretend otherwise.

---

**Version:** 1.0  
**Last Updated:** December 2025  
**Author:** regul8or with help of AI Assistant
