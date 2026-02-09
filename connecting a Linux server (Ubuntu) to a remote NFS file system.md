Abdul, this guide is about **connecting a Linux server (Ubuntu) to a remote NFS file system** so that it behaves like a local disk.

This is a **core DevOps skill** because NFS is commonly used with:

* Amazon EFS
* On-prem NAS
* Shared application storage
* Kubernetes Persistent Volumes

I will explain this **from zero**, in a structured and practical way:

---

# 1. What You Are Trying to Achieve

You want:

```
Remote NFS Storage  →  Appears as local folder on Ubuntu
```

After mounting:

```
/mnt/shared
   ├── file1
   ├── logs/
   └── uploads/
```

Applications read/write as if it were local disk.

---

# 2. Core Concepts (Must Understand)

### 2.1 NFS (Network File System)

NFS allows a server to **export** a directory and other machines to **mount** it.

Think:

```
Remote disk over network
```

---

### 2.2 Client vs Server

* NFS Server → hosts files
* NFS Client → mounts files

Your Ubuntu machine is acting as a **client**.

---

### 2.3 Temporary vs Persistent Mount

| Type             | Survives Reboot? |
| ---------------- | ---------------- |
| mount command    | No               |
| /etc/fstab entry | Yes              |

---

# 3. Prerequisites Checklist

Before mounting:

* NFS server reachable
* Port 2049 open
* DNS resolves
* You have root or sudo access

Test:

```bash
ping <File-system-DNS-name>
```

If ping fails → fix networking first.

---

# 4. Step 1 — Install NFS Client Tools

```bash
sudo apt update
sudo apt install -y nfs-common
```

### What this does

Installs NFS drivers and utilities.

Verify:

```bash
dpkg -l | grep nfs-common
```

---

# 5. Step 2 — Create Mount Directory

Choose a directory:

```bash
sudo mkdir -p /mnt/nfs
```

This will be your mount path.

---

# 6. Step 3 — Temporary Mount (Test First)

```bash
sudo mount -t nfs4 \
-o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport \
<File-system-DNS-name>:/ /mnt/nfs
```

---

## Explanation of Options

| Option      | Meaning                       |
| ----------- | ----------------------------- |
| nfsvers=4.1 | Use NFS v4.1                  |
| rsize/wsize | Large I/O buffers             |
| hard        | Retry forever if server fails |
| timeo=600   | Wait time                     |
| retrans=2   | Retry count                   |
| noresvport  | Use non-reserved port         |

These are **AWS EFS recommended options**.

---

## Verify Mount

```bash
df -h | grep nfs
```

or

```bash
mount | grep nfs
```

Create test file:

```bash
touch /mnt/nfs/test.txt
ls /mnt/nfs
```

If file appears → working.

---

# 7. Step 4 — Persistent Mount (Auto on Reboot)

Open fstab:

```bash
sudo nano /etc/fstab
```

Add line:

```bash
<File-system-DNS-name>:/ /mnt/nfs nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport,_netdev 0 0
```

Save and exit.

---

## Test fstab Entry (Important)

```bash
sudo umount /mnt/nfs
sudo mount -a
```

If no error → safe.

Reboot test:

```bash
sudo reboot
```

After reboot:

```bash
df -h | grep nfs
```

---

# 8. Why `_netdev` Is Critical

Prevents boot hang.

System waits for network before mounting.

Without it → server may freeze during boot.

---

# 9. Common Problems & Fixes

---

### Mount hangs

Cause:

* Port 2049 blocked

Fix:

Open firewall / security group.

---

### Permission denied

Cause:

* NFS server export permissions

Fix:

Check server-side export rules.

---

### Read-only mount

Cause:

* Export configured as ro

Fix:

Change server export to rw.

---

# 10. Performance Best Practices

* Use NFS 4.1
* Place clients and NFS in same AZ (for AWS)
* Avoid heavy random writes
* Use local disk for temp/cache

---

# 11. Real-World Example

Web servers share uploads:

```
Web1 → /mnt/uploads
Web2 → /mnt/uploads
Web3 → /mnt/uploads
```

All use same NFS.

No file sync required.

---

# 12. Security Best Practices

* Restrict NFS access by IP
* Use private networks
* Encrypt in transit (for EFS: TLS)
* Avoid public exposure

---

# 13. Mental Model

Think:

```
USB drive plugged over network
```

Same behavior, remote location.

---

# 14. Official References

* Ubuntu NFS Client
  [https://help.ubuntu.com/community/SettingUpNFSHowTo](https://help.ubuntu.com/community/SettingUpNFSHowTo)

* Amazon EFS Mounting
  [https://docs.aws.amazon.com/efs/latest/ug/mounting-fs.html](https://docs.aws.amazon.com/efs/latest/ug/mounting-fs.html)

* NFS v4 Manual
  [https://man7.org/linux/man-pages/man5/nfs.5.html](https://man7.org/linux/man-pages/man5/nfs.5.html)

---

If you want next, I can teach:

* Setting up NFS server
* Using NFS with Docker volumes
* Using NFS in Kubernetes
* Performance tuning & troubleshooting

Tell me which direction you want to go.
