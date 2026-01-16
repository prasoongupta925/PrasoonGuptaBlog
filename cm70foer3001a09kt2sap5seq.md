---
title: "Building a Self‑Managed File Share System Using NFS Across Cloud Providers"
seoTitle: "NFS File Sharing Across Clouds"
seoDescription: "Learn to build a self-managed file share system using NFS across multiple cloud providers for full control and cost-efficiency"
datePublished: Tue Feb 11 2025 12:03:39 GMT+0000 (Coordinated Universal Time)
cuid: cm70foer3001a09kt2sap5seq
slug: building-a-selfmanaged-file-share-system-using-nfs-across-cloud-providers
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1739274770113/79ee1aa0-0ac8-4b25-945e-a51331b54299.png

---

Have you ever wanted to fully control your file sharing system on Linux? In this guide, we'll show you how to set up a strong, self-managed file sharing system using NFS across AWS, Azure, Digital Ocean, or any cloud provider. If you prefer managed solutions, you can also explore AWS EFS, Azure File Share, or GCP Cloud Store.

## **Why a Self‑Managed File Share?**

Self-managing your file shares means:

* **Full Control:** Customize exports, mount options, and security settings to fit your needs perfectly.
    
* **Flexibility:** Easily integrate with your infrastructure across different cloud providers.
    
* **Cost-Effectiveness:** Use standard Linux tools without **paying** extra for **managed services**.
    

> **Pro Tip:** If you want a solution that requires less management, consider using managed services like **AWS EFS**, **Azure File Share**, or **GCP Cloud Store**.

# Setting Up the NFS Server

This section explains how to install and set up the NFS server on your Linux machine.  
  
**1\. Install the NFS Kernel Server**  
Open your terminal and run:

```plaintext
sudo apt update
sudo apt install nfs-kernel-server
sudo systemctl status nfs-kernel-server
sudo systemctl start nfs-kernel-server
sudo systemctl restart nfs-kernel-server
```

Then, enable the service to start automatically on boot:

```plaintext
sudo systemctl enable nfs-kernel-server
```

### 2\. Configure the Shared Directories

Edit the `/etc/exports` file to define which directories to share and which networks have access. For example, to share your WordPress content and uploads directory with the private cloud network (`189.31.0.0/16`):

```plaintext
sudo nano /etc/exports

# Add the following lines:

/var/www/html/wp-content 189.31.0.0/16(rw,sync,no_root_squash,no_subtree_check) 
/mnt/data/html/wp-content/uploads 189.31.0.0/16(rw,sync,no_root_squash,no_subtree_check)
```

Apply the changes with:

```plaintext
sudo exportfs -arv
```

You should see output confirming that the directories are now exported.

```plaintext
root@ip-189-31-23-91:~# sudo exportfs -arv
exporting 189.31.0.0/16:/var/www/html/wp-content
exporting 189.31.0.0/16:/mnt/data/html/wp-content/uploads
```

# Configuring the NFS Client

After the server is ready, configure the client machine to mount the shared directories.

### 1\. Install NFS Client Components

On your client system, install the NFS utilities:

```plaintext
sudo apt install nfs-common
```

### 2\. Manage Firewall Rules

For providers like Digital Ocean (or any system using UFW), check your firewall status:

```plaintext
sudo ufw status
sudo ufw app list
```

Allow NFS traffic from your server’s IP or network. For example, on Digital Ocean you might run:

```plaintext
sudo ufw allow from <server_ip_or_network> to any port nfs
```

> > **Note:** On AWS, Azure, or GCP, set up your security groups or network security groups (NSG) to allow these ports:
> > 
> > * **2049/TCP:** NFS
> >     
> > * **111/TCP:** Port mapper
> >     
> > * **662/TCP:** NFS Lock Manager
> >     
> > * **20048/TCP:** Mount daemon
> >     
> > * **32803/TCP:** Alternate mount daemon3. Mount the NFS Shares
> >     

**Before mounting, ensure the NFS client service is running correctly:**

```plaintext
sudo rm /lib/systemd/system/nfs-common.service
sudo systemctl daemon-reload
sudo systemctl start nfs-common
sudo systemctl restart nfs-common
sudo systemctl enable nfs-common
```

Mount the shares manually (adjust IPs and mount points as needed):

```plaintext
sudo mount 189.31.23.91:/mnt/data/html/wp-content/uploads /mnt/data/html/wp-content/uploads
sudo mount 189.31.23.91:/var/www/html/wp-content /var/www/html/wp-content
```

Verify that the shares are mounted:

```plaintext
mount | grep nfs
```

### 4\. Set Up Auto‑Mount via `/etc/fstab`

To have the shares mount automatically at boot, add these entries to `/etc/fstab`:

```plaintext
189.31.23.91:/mnt/data/html/wp-content/uploads /mnt/data/html/wp-content/uploads nfs rw,auto,nofail,noatime,nolock,intr,tcp,actimeo=1800 0 0
189.31.23.91:/var/www/html/wp-content  /var/www/html/wp-content  nfs rw,auto,nofail,noatime,nolock,intr,tcp,actimeo=1800 0 0
```

Test the configuration by remounting all filesystems:

```plaintext
sudo mount -a
```

## Automating Mounts with a Script

To make sure your NFS shares are always mounted, even after reboots, use an auto-mount script. Create a file at `/var/www/nfs_automount`[`v1.sh`](http://v1.sh) with the following content:

```plaintext
#!/bin/bash
 
# Variables
NFS_SERVER="189.31.23.91"
SHARE_PATH1="/var/www/html/wp-content"
SHARE_PATH2="/mnt/data/html/wp-content/uploads"
MOUNT_POINT1="/var/www/html/wp-content"
MOUNT_POINT2="/mnt/data/html/wp-content/uploads"
FSTAB_FILE="/etc/fstab"
 
# Create mount point for uploads if it doesn't exist
if [ ! -d "$MOUNT_POINT2" ]; then
    echo "Creating directory $MOUNT_POINT2"
    sudo mkdir -p "$MOUNT_POINT2"
fi
 
# Mount the uploads share first
echo "Mounting $NFS_SERVER:$SHARE_PATH2 to $MOUNT_POINT2..."
sudo mount $NFS_SERVER:$SHARE_PATH2 $MOUNT_POINT2
 
if mountpoint -q "$MOUNT_POINT2"; then
    echo "$MOUNT_POINT2 mounted successfully."
else
    echo "Failed to mount $MOUNT_POINT2."
    exit 1
fi
 
# Create mount point for content if it doesn't exist
if [ ! -d "$MOUNT_POINT1" ]; then
    echo "Creating directory $MOUNT_POINT1"
    sudo mkdir -p "$MOUNT_POINT1"
fi
 
# Mount the content share
echo "Mounting $NFS_SERVER:$SHARE_PATH1 to $MOUNT_POINT1..."
sudo mount $NFS_SERVER:$SHARE_PATH1 $MOUNT_POINT1
 
if mountpoint -q "$MOUNT_POINT1"; then
    echo "$MOUNT_POINT1 mounted successfully."
else
    echo "Failed to mount $MOUNT_POINT1."
    exit 1
fi
 
# Add to /etc/fstab if missing
grep -q "$SHARE_PATH2" "$FSTAB_FILE"
if [ $? -ne 0 ]; then
    echo "Adding $NFS_SERVER:$SHARE_PATH2 to /etc/fstab"
    echo "$NFS_SERVER:$SHARE_PATH2 $MOUNT_POINT2 nfs rw,auto,nofail,noatime,nolock,intr,tcp,actimeo=1800 0 0" | sudo tee -a "$FSTAB_FILE"
fi
 
grep -q "$SHARE_PATH1" "$FSTAB_FILE"
if [ $? -ne 0 ]; then
    echo "Adding $NFS_SERVER:$SHARE_PATH1 to /etc/fstab"
    echo "$NFS_SERVER:$SHARE_PATH1 $MOUNT_POINT1 nfs rw,auto,nofail,noatime,nolock,intr,tcp,actimeo=1800 0 0" | sudo tee -a "$FSTAB_FILE"
fi
 
# Reload systemd to apply fstab changes
echo "Reloading systemd to apply changes in /etc/fstab..."
sudo systemctl daemon-reload
 
if [ $? -eq 0 ]; then
    echo "Systemd reloaded successfully."
else
    echo "Failed to reload systemd."
    exit 1
fi
 
echo "All operations completed successfully."
```

Make the script executable:

```plaintext
sudo chmod +x /var/www/nfs_automount_v1.sh
```

Then add an entry to your Crontab to run it at reboot:

```plaintext
@reboot /bin/bash /var/www/nfs_automount_v1.sh
```

---

# Final Thoughts

With this guide, you now possess a comprehensive, self-managed file-sharing system using NFS. You have learned how to:

* **Install and configure the NFS server and client across various cloud providers.**
    
* **Set up UFW (or equivalent firewall rules) to ensure secure NFS traffic**.
    
* **Automate mounting using both** `/etc/fstab` **and a custom script.**
    

While a self-managed solution provides maximum control and flexibility, managed options such as **AWS EFS**, **Azure File Share**, and **GCP Cloud Store** are also available if you prefer a more hands-off approach.

*Wishing you success in your file-sharing endeavors, and may your file systems always mount seamlessly!*

---

*Note: Always adjust IP addresses, network ranges, and file paths to fit your specific environment and security needs.*