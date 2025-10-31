# TrueNAS → Proxmox → Unprivileged LXC → Docker: Persistent NFS Storage Guide

> A step-by-step guide to mounting a TrueNAS NFS share on Proxmox, binding it into an **unprivileged LXC container**, and configuring Docker access while preserving **UID/GID consistency (user 1000)** across all layers.

![Proxmox](https://img.shields.io/badge/Proxmox-7%2B-orange?logo=proxmox)
![TrueNAS](https://img.shields.io/badge/TrueNAS-NFS-blue?logo=truenas)
![Docker](https://img.shields.io/badge/Docker-Compatible-2496ED?logo=docker)
![License](https://img.shields.io/badge/license-MIT-green)

---

## 🧩 Overview

This guide walks through how to mount a **TrueNAS NFS share** to a **Proxmox host**, bind it into an **unprivileged LXC container**, and then allow **Docker** within that container to use the share directly.  
The focus is on maintaining **consistent file ownership and permissions** (UID/GID = 1000) between TrueNAS, Proxmox, and Docker — enabling seamless shared storage access without permission mismatches or root escalation issues.

---

## 💭 Personal

My goal with this setup was to build a single, efficient system that could serve as both a NAS and an application host, without needing extra hardware or drives just for apps.

While TrueNAS is excellent for storage and data protection, running it bare metal comes with a major limitation:  
you cannot install or run apps directly on the boot drive or pool. To add Docker, containers, or other applications, you would need a separate physical drive or pool dedicated to those apps.

Instead of going that route, I chose to run TrueNAS as a virtual machine inside Proxmox.  
This lets me:
- Use the Proxmox host for applications, containers, and services such as Docker, Sonarr, and Radarr  
- Keep TrueNAS as a dedicated NAS focused purely on storage and file sharing  
- Take advantage of drive passthrough so TrueNAS still has full control over the disks  
- Treat TrueNAS as a plug-and-play NAS appliance that I can migrate or restore easily

In this configuration, Proxmox handles the apps and workloads, while TrueNAS provides networked NFS storage.  
This keeps everything reliable, modular, and flexible.

---

## 🖥️ System Configuration

This setup was built and tested on the following hardware and virtualization configuration.  
Your specs don’t need to match exactly, but they can help provide context for performance and behavior.

| Component | Details |
|------------|----------|
| **CPU (current)** | Intel Core i3-12100 |
| **Planned upgrade** | Intel Core i7-12700K *(guide may or may not be updated post-upgrade)* |
| **OS and Apps drive** | PNY CS3140 4 TB NVMe SSD |
| **Storage drives** | 3 × WD Red Plus 8 TB HDDs |
| **Host platform** | Proxmox VE |
| **TrueNAS deployment** | Running as a **VM** (not an LXC) on the same Proxmox host |
| **Drive configuration** | HDDs are **passed through** directly to the TrueNAS VM |
| **Shared storage** | TrueNAS NFS share exported to the Proxmox host and bind-mounted into an unprivileged LXC running Docker |
| **NFS export settings** | The NFS share uses **mapall** to the user corresponding to **UID 1000**, ensuring consistent ownership and permissions across TrueNAS, Proxmox, LXC, and Docker layers. |

> 🧠 Note: Since TrueNAS runs as a VM with direct disk passthrough, all NFS performance and permission consistency depend on proper passthrough configuration and NFS export settings (especially `mapall`).

---

## 🗄️ Step 1: Add the NFS Share to Proxmox

1. In the **Proxmox Web UI**, navigate to:  
   **Datacenter → Storage → Add → NFS**

2. Fill out the required NFS details:
   - **ID:** A friendly name for your storage (e.g., `truenas-nfs`)
   - **Server:** The IP address or hostname of your TrueNAS server
   - **Export:** The exported NFS path (e.g., `/mnt/pool1/docker-share`)
   - **Content:** Select what the storage will be used for (e.g., `Disk image`, `Container`, `ISO image`, or `VZDump backup file`)
   - **Nodes:** Choose the Proxmox node(s) that should have access
   - **Options:** 
     - Check **Enable**
     - (Optional) Enable **Backup**, **ISO**, or **Container** depending on your use case

3. Click **Add** to finalize and mount the NFS share in Proxmox.

✅ **Tip:** Verify the NFS share by going to **Datacenter → Storage**, selecting your NFS entry, and checking the **Status** tab. It should show **Active**.

---

## 📁 Step 2: Create a Local Mount Point and Connect NFS via /etc/fstab

Once your NFS share is added to Proxmox, create a local directory where it will be mounted.  
This directory will act as the access point for the NFS share on the host.

```bash
mkdir -p /mnt/<mount-point>
```

Now, edit your `/etc/fstab` file to automatically mount the NFS share on boot:

```bash
nano /etc/fstab
```

Add the following line (replace placeholders with your own details):

```bash
<server-ip>:/mnt/<pool>/<dataset>   /mnt/<mount-point>   nfs4  rw,noatime,_netdev,x-systemd.automount,retry=5,timeo=14  0  0
```

### ⚙️ Explanation

| Parameter | Description |
|------------|-------------|
| `<server-ip>:/mnt/<pool>/<dataset>` | The **server IP** and **exported path** of your TrueNAS NFS share |
| `/mnt/<mount-point>` | The **local mount point** on your Proxmox host |
| `nfs4` | Specifies the **NFS version** being used |
| `rw` | Mounts the share with **read/write** access |
| `noatime` | Prevents the system from updating file access timestamps — improves performance |
| `_netdev` | Ensures the system waits for the **network to be available** before mounting |
| `x-systemd.automount` | Allows **on-demand mounting** instead of at boot, reducing boot delays |
| `retry=5` | Retries mounting up to **5 times** if the connection fails |
| `timeo=14` | Sets the **timeout** (in tenths of a second) for NFS requests |
| `0 0` | Disables **dump** and **fsck** for this mount (not needed for NFS) |

✅ **Pro tip:** After saving `/etc/fstab`, test the mount immediately without rebooting:

```bash
mount -a
```

---

## 🧩 Step 3: Bind-Mount the Host NFS into the Unprivileged LXC

Use `pct set` to attach the host mount (created in Step 2) into your container.

```bash
# Replace placeholders:
# <ctid>           = your LXC container ID (e.g., 200)
# <host-mount>     = the host path where NFS is mounted (e.g., /mnt/media-nas)
# <container-path> = the path inside the container where you want it mounted (e.g., /mnt/nas)
pct set <ctid> --mp0 <host-mount>,mp=<container-path>
```

Reboot or restart the container to ensure the mount is active:

```bash
pct reboot <ctid>
# or
pct stop <ctid> && pct start <ctid>
```

---

## 🔐 Step 4: (Advanced) ID Mapping to Preserve UID/GID 1000 for Docker

> ⚠️ **Warning:** Modifying user namespace mappings on unprivileged containers can weaken isolation if misconfigured. Proceed only if you understand the implications and have backups.

### 4.1 Allow the host to map UID/GID 1000

Edit these files **on the Proxmox host**:

```bash
sudo nano /etc/subuid
sudo nano /etc/subgid
```

Add or adjust entries:

```text
# /etc/subuid
# comment out what you originally had so you can go back and restore
root:100000:65536
root:1000:1
```
```text
# /etc/subgid
# comment out what you originally had so you can go back and restore
root:100000:65536
root:1000:1
```

### 4.2 Add custom ID maps to the container config

Edit `/etc/pve/lxc/<ctid>.conf`:

```text
lxc.idmap: u 0 100000 1000
lxc.idmap: u 1000 1000 1
lxc.idmap: u 1001 101001 64535

lxc.idmap: g 0 100000 1000
lxc.idmap: g 1000 1000 1
lxc.idmap: g 1001 101001 64535
```

Restart the container:

```bash
pct restart <ctid>
```

---

## ⚠️ Step 5: Pre-Create Docker Bind Mount Directories (Avoid Permission Conflicts)

Before running Docker containers that map volumes to your NFS share, **make sure the directories exist** on the mounted share.  
Docker tends to **create missing folders as root-mapped 100000**, not as user 1000, which can cause permission issues.

### 🧩 Why This Matters

If Docker creates a directory itself, it inherits shifted UID mappings, resulting in folders owned by `100000:100000` on the host.

### ✅ The Fix

Create the full folder structure **manually** before starting Docker, and set ownership to user 1000.

```bash
mkdir -p /mnt/nas/configs/sonarr
mkdir -p /mnt/nas/media
chown -R 1000:1000 /mnt/nas/configs
chown -R 1000:1000 /mnt/nas/media
```

Example Docker Compose snippet:

```yaml
volumes:
  - /mnt/nas/configs/sonarr:/config
  - /mnt/nas/media:/media
```

As long as these directories exist and are owned by UID/GID 1000 **before** Docker initializes them,  
permissions will remain consistent across the NFS share.

---

### ✅ Verification

Inside the container:

```bash
id
stat -c "%u:%g %n" <container-path>
touch <container-path>/testfile && ls -l <container-path>/testfile
```

On the host, confirm ownership appears as `1000:1000`.

> 💡 If ownership doesn’t match, re-check `/etc/subuid`, `/etc/subgid`, and `lxc.idmap` for overlaps or typos.  
> Also ensure your NFS export isn’t applying `all_squash` or `root_squash`.
