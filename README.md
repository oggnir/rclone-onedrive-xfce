# Mount Microsoft OneDrive in Fedora XFCE with rclone

A practical guide to accessing Microsoft OneDrive directly from Fedora XFCE using [rclone](https://rclone.org/) and Thunar.

The approach described here mounts OneDrive as a virtual filesystem rather than downloading the entire OneDrive to the local computer.

## Architecture

```text
Thunar
   │
   ▼
~/OneDrive
   │
   ▼
rclone mount
   │
   ▼
FUSE
   │
   ▼
Internet
   │
   ▼
Microsoft OneDrive
```

This is a **cloud mount**, not a complete local synchronisation.

Files remain primarily on OneDrive. rclone may use local caching to support normal read/write operations.

---

## 1. Install rclone

On Fedora:

```bash
sudo -v
curl https://rclone.org/install.sh | sudo bash
```

Check the installation:

```bash
rclone version
```

---

## 2. Configure Microsoft OneDrive

Start the rclone configuration interface:

```bash
rclone config
```

Create a new remote:

```text
n
```

Choose:

```text
Microsoft OneDrive
```

Give the remote a name, for example:

```text
OneDrive
```

For the following options, the exact prompts may vary slightly depending on the rclone version:

* `client_id` → leave blank
* `client_secret` → leave blank
* Region → Microsoft Cloud Global
* `tenant` → leave blank
* Advanced config → `n`
* Browser authentication → `y`

Sign in using your own Microsoft account.

If rclone asks you to select the type of OneDrive account, choose the option appropriate for your Microsoft 365 / OneDrive environment.

Confirm the detected drive when rclone displays it.

When configuration is complete:

```text
q
```

to quit the configuration interface.

### Important

Each user should authenticate using their **own Microsoft account**.

Do not share:

* passwords
* OAuth tokens
* `rclone.conf`
* authentication URLs containing sensitive information

---

## 3. Verify the OneDrive connection

List the folders at the top level:

```bash
rclone lsd "OneDrive:"
```

If the connection works, rclone should display the directories available in the remote OneDrive.

Check the available storage:

```bash
rclone about "OneDrive:"
```

---

## 4. Create a local mount point

Create a directory that will represent the mounted OneDrive:

```bash
mkdir -p ~/OneDrive
```

The directory itself does not contain a local copy of the entire OneDrive.

Once mounted, it becomes the interface through which rclone accesses the cloud storage.

---

## 5. Test the mount manually

Run:

```bash
rclone mount "OneDrive:" ~/OneDrive --vfs-cache-mode writes
```

Keep this terminal open while testing.

Open Thunar and navigate to:

```text
Home → OneDrive
```

The remote OneDrive should now appear like a normal directory.

### Test reading and writing

Create a small test file:

```bash
echo "rclone is working." > ~/OneDrive/rclone-test.txt
```

Read it:

```bash
cat ~/OneDrive/rclone-test.txt
```

The file should also become visible through the OneDrive web interface.

After testing, remove it:

```bash
rm ~/OneDrive/rclone-test.txt
```

Stop the temporary mount with:

```text
Ctrl+C
```

---

## 6. Create an automatic systemd user service

The manual mount is useful for testing, but it is inconvenient to start manually every time.

Create the systemd user-service directory:

```bash
mkdir -p ~/.config/systemd/user
```

Create the service:

```bash
nano ~/.config/systemd/user/rclone-onedrive.service
```

Use:

```ini
[Unit]
Description=Rclone OneDrive Mount
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/rclone mount "OneDrive:" %h/OneDrive \
    --vfs-cache-mode writes \
    --dir-cache-time 1m \
    --poll-interval 1m
ExecStop=/usr/bin/fusermount3 -u %h/OneDrive
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
```

Save the file:

```text
Ctrl+O
Enter
Ctrl+X
```

---

## 7. Enable and start the service

Reload systemd:

```bash
systemctl --user daemon-reload
```

Enable the service:

```bash
systemctl --user enable rclone-onedrive
```

Start it:

```bash
systemctl --user start rclone-onedrive
```

Check its status:

```bash
systemctl --user status rclone-onedrive
```

A working service should show:

```text
Active: active (running)
```

Check whether it is enabled for future logins:

```bash
systemctl --user is-enabled rclone-onedrive
```

Expected:

```text
enabled
```

---

## 8. Normal daily use

Once the service is running:

1. Open Thunar.
2. Open `~/OneDrive`.
3. Work with files normally.

You can:

* read files
* create files
* edit files
* rename files
* create directories
* move files
* delete files

The operations are performed against the remote OneDrive.

### Important mental model

```text
~/OneDrive
```

looks like a normal local directory, but it is actually a **mounted cloud filesystem**.

It is therefore different from a traditional synchronization client.

---

## 9. Local storage and caching

The following command:

```bash
du -sh ~/OneDrive
```

should **not** be interpreted as "how much space OneDrive is consuming on my SSD."

`~/OneDrive` represents the mounted remote filesystem.

With:

```text
--vfs-cache-mode writes
```

rclone can use local storage for caching and temporary file operations.

The entire OneDrive is **not automatically downloaded** to the computer.

To inspect rclone's configured paths:

```bash
rclone config paths
```

Depending on the rclone configuration and operating system, this can help identify cache/configuration locations.

---

## 10. Useful commands

### Test the remote

```bash
rclone lsd "OneDrive:"
```

### Check remote storage

```bash
rclone about "OneDrive:"
```

### Check the systemd service

```bash
systemctl --user status rclone-onedrive
```

### Restart the mount

```bash
systemctl --user restart rclone-onedrive
```

### Stop the mount

```bash
systemctl --user stop rclone-onedrive
```

### View recent logs

```bash
journalctl --user -u rclone-onedrive --no-pager -n 50
```

### Follow logs live

```bash
journalctl --user -u rclone-onedrive -f
```

### Check the mount

```bash
findmnt ~/OneDrive
```

### Unmount manually

```bash
fusermount3 -u ~/OneDrive
```

### Find the rclone configuration file

```bash
rclone config file
```

### Find rclone configuration/cache paths

```bash
rclone config paths
```

---

## 11. Troubleshooting

### OneDrive is not visible in Thunar

Check the service:

```bash
systemctl --user status rclone-onedrive
```

Then test the remote independently:

```bash
rclone lsd "OneDrive:"
```

If the remote works but the mount does not, inspect the service log:

```bash
journalctl --user -u rclone-onedrive --no-pager -n 50
```

---

### The mount appears stuck

Try:

```bash
fusermount3 -u ~/OneDrive
```

Then:

```bash
systemctl --user restart rclone-onedrive
```

---

### Check whether the service starts automatically

```bash
systemctl --user is-enabled rclone-onedrive
```

Expected:

```text
enabled
```

---

## 12. Offline use

This setup is **not an offline copy of OneDrive**.

Normal access to remote files requires network connectivity.

If a file needs to remain available independently of OneDrive or the network, keep a genuine local copy outside the mount, for example:

```text
~/Documents/
```

---

## 13. Security and privacy

Treat the mounted directory as the actual cloud storage.

For example:

```bash
rm ~/OneDrive/important-file.pdf
```

can delete the actual remote file.

Be especially careful with destructive commands such as:

```bash
rm -rf ~/OneDrive/...
```

### Never publish

Do not upload any of the following to a public GitHub repository:

* `rclone.conf`
* passwords
* OAuth tokens
* authentication credentials
* private keys
* taxpayer information
* confidential institutional documents
* private OneDrive files
* internal screenshots
* private SharePoint URLs
* personal account identifiers
* confidential folder structures

Each user should configure their own rclone remote and authenticate with their own account.

---

## 14. Why use rclone instead of downloading everything?

A traditional synchronisation approach may maintain a substantial local copy of the cloud storage.

With an rclone mount:

```text
Local computer
    │
    │ virtual filesystem
    ▼
~/OneDrive
    │
    ▼
rclone
    │
    ▼
OneDrive
```

The computer can interact with remote files without requiring the entire cloud storage to occupy the local SSD.

This can be particularly useful on computers with limited local storage.

---

## 15. Example for other users

The exact remote name is a personal choice.

For example, one user might configure:

```text
OneDrive
```

while another might use:

```text
WorkDrive
```

If the remote is named `WorkDrive`, the commands become:

```bash
rclone lsd "WorkDrive:"
```

and:

```bash
rclone mount "WorkDrive:" ~/OneDrive --vfs-cache-mode writes
```

The important relationship is:

```text
rclone remote name → local mount point
```

---

## 16. Summary

The complete setup is:

```text
Microsoft OneDrive
        │
        │ Internet
        ▼
      rclone
        │
        │ FUSE mount
        ▼
   ~/OneDrive
        │
        ▼
      Thunar
```

The result is a lightweight way to access OneDrive from Fedora XFCE while keeping the cloud storage primarily in the cloud rather than downloading the entire repository to the local computer.

**Key principle:**

> Document the method publicly; keep credentials, personal information, and institutional data private.
