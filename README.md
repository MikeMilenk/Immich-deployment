# Build Your Own Media Cloud with Immich

> In this guide, I'll show you how to build your own private photo and video cloud using **Immich**, similar to **iCloud Photos** or **Google Photos**. Here is the **[official Immich deployment guide](https://docs.immich.app/overview/quick-start)**. My guide provides a more detailed setup with additional configuration for Proxmox, ZFS storage, dedicated HDD storage, Ubuntu Server, and Tailscale remote access.

You need first [deploy an Ubuntu Server VM on **Proxmox VE**](https://github.com/MikeMilenk/Deploying-Linux-Server.git), then install Immich and configure it to store all photos on a dedicated storage pool.

**Immich** is a self-hosted photo and video management system running on your own server. It allows you to:
* automatically back up photos from your phone
* store all media on your own infrastructure
* organize photos into albums
* search by metadata, location, and content
* use face and object recognition features
* access your library via web and mobile apps

In short, Immich gives you the convenience of a modern cloud photo service while keeping full control of your data on your own hardware.

# Preparing Storage for Immich Media

My server contains 2 SSDs and 2 HDDs. The SSD is used only for Proxmox VE and Linux ISO images, while the two HDDs will be combined into a single **1.5 TB** ZFS storage pool and will be called as **`immich-zfs`**. This keeps the OS separate from the photo library and provides one large location for all app data. On the screenshot below, you can see my disks. In the next steps, I will combine these HDDs for Immich.

![Server Drives](https://github.com/MikeMilenk/Immich-deployment/blob/83dcf0bc14420e3a2638382538ff980df59454ee/images/lsblk.png)

---

# 1. Create the ZFS Storage Pool

## 1.1 Create the ZFS Pool in Proxmox

In the Proxmox web interface, select your node (**PVE**) and navigate to the **ZFS Management Page**:
```text
PVE → Disks → ZFS
```

Click `Create: ZFS`.

I named it **`immich-zfs`** for easier tracking in the future. Select the drives you want to combine. Leave all other settings at their default values. Click **Create** and wait for Proxmox to finish creating the pool.

![Create the ZFS Storage Pool](https://github.com/MikeMilenk/Immich-deployment/blob/83dcf0bc14420e3a2638382538ff980df59454ee/images/immich-zfs.png)

---

## 1.2 Add the ZFS Pool as Proxmox Storage

Once the pool has been created, navigate to:

```text
Datacenter → Storage
```
Add the storage: `Add → ZFS`.

- I set the **ID** to **`immich`**.
- For **ZFS Pool**, select the pool created in the previous step. In my case it's **`immich-zfs`**.
- For **Content**, select **`Disk Image`**.
 
*`NOTE: Immich runs inside a VM, and virtual machines require block storage for their virtual disks. "Container" content type is intended for different purposes and isn't necessary for an Immich VM.`*

![Add the Pool as Proxmox Storage](https://github.com/MikeMilenk/Immich-deployment/blob/83dcf0bc14420e3a2638382538ff980df59454ee/images/immich%20disk.png)

---

## 1.3 Verify the ZFS Pool Configuration

After saving, **`immich`** will appear in the Proxmox Storage list and can be used to store virtual disks for VMs. You can verify that it was created successfully by checking its status from the Proxmox terminal:

```bash
zpool status
```

If the pool was created correctly, the output should display the pool name, its current health status (`ONLINE`), and the disks that are part of the pool.

At this point, the SSD continues hosting Proxmox VE and will host future Linux server (where Immich Cloud will be deployed), while your Immich data will be stored on the new **immich-zfs** pool.

![zfs pool status](https://github.com/MikeMilenk/Immich-deployment/blob/b9b092946b3f18329b03df9a5c8b804e5b498ac7/images/zfs%20pools%20status.png)

---

# 2. Prepare the Ubuntu Server
After creating and configuring the **ZFS pool** in **Proxmox**, we move to our Ubuntu Server VM.

This is where we will install **Docker** and **Immich**. The setup will look like this:
```text
Proxmox → ZFS Pool → Ubuntu Server VM → Docker → Immich
```

---

## 2.1 Prepare the combined HDD storage for Immich
As I indicated previously, I want Immich media to be stored on separate HDDs instead of the SSD where the Ubuntu VM is located.

### 2.1.1 Add the combined HDD storage to Ubuntu Server VM

In Proxmox, go to:
`VM → Hardware → Add → Hard Disk`
![Adding HDD Storage](https://github.com/MikeMilenk/Immich-deployment/blob/7690750d53db603162fafd5731209bb54c9d61b9/images/Add%20HDD.png)

> Bus: `SCSI: 1`
> Storage: `immich` — our existing Proxmox ZFS storage backed by the `immich-zfs` pool.
> Disk size (GiB): `1300`

*`NOTE: I have 1.45 TB of available storage, which is approximately 1350 GiB. I will allocate 1300 GiB and leave approximately 50 GiB of free space to avoid filling the storage completely and to leave some room for normal ZFS operations and future growth.`*

Click `Add`.

![Configuring HDD Storage](https://github.com/MikeMilenk/Immich-deployment/blob/7690750d53db603162fafd5731209bb54c9d61b9/images/Add%20HDD%20Storage.png)

This attaches a virtual hard disk backed by our ZFS pool to the Ubuntu VM. The disk will then appear inside Ubuntu and can be partitioned, formatted, and mounted for Immich storage.

Run `lsblk` to verify that the new disk is detected. In my case, `sda` is the main disk containing the Ubuntu system, while `sdb` is the newly added disk with no partitions yet. We will create a partition on `sdb` next.

![detected disks](https://github.com/MikeMilenk/Immich-deployment/blob/93c59fefd70985038fca32a85729ffbc9954c372/images/lsblk.png)

### 2.1.2 Create a Partition
```bash
sudo fdisk /dev/sdb
```
Then select:
```text
n       ← create new partition
Enter   ← default partition number
Enter   ← default first sector
Enter   ← use all available space
w       ← write changes
```

### 2.1.3 Format the Partition
```bash
sudo mkfs.ext4 /dev/sdb1
```
This creates an `ext4` filesystem so Ubuntu can store files on it.

### 2.1.4 Mount the Disk
In my case, I created and named the new directory **`immich-storage`** to make it easier to identify and track later.
```bash
sudo mkdir -p /mnt/immich-storage
sudo mount /dev/sdb1 /mnt/immich-storage
```
This makes the disk accessible through `/mnt/immich-storage`.

### 2.1.5 Configure Permanent Mounting
To prevent the disk from becoming unavailable after future reboots, configure automatic mounting using `/etc/fstab`.

First, retrieve the disk `UUID`:

```dash
sudo blkid /dev/sdb1
```
![UUID](https://github.com/MikeMilenk/Immich-deployment/blob/1cf84d0e13bf50ce4dccb47583d23de1738b4a61/images/UUID.png)

Then edit `/etc/fstab`:
```bash
sudo nano /etc/fstab
```
Add: `UUID=YOUR-DISK-UUID /mnt/immich-storage ext4 defaults 0 2`

Replace `YOUR-DISK-UUID` with the `UUID` returned by `blkid`.
![fstab edit](https://github.com/MikeMilenk/Immich-deployment/blob/1cf84d0e13bf50ce4dccb47583d23de1738b4a61/images/fstab%20edit.png)

Skipping this step may cause Immich to lose access to its storage after a reboot or power outage.
For more information on what can happen and how to recover Immich, see my [Immich Storage Recovery Guide](https://github.com/MikeMilenk/Immich-Storage-Recovery-After-Power-Outage.git).

---

## 2.2 Install and Configure Immich
We need to download and keep the `docker-compose.yml` and `.env` files in the newly created **Immich** directory. In my case, the Immich directory is `immich-app`.

### 2.2.1 Create the Immich Directory and navigate into it
Create a directory of your choice (I named it as `immich-app`)
```bash
mkdir ./immich-app
cd ./immich-app
```

### 2.2.2 Download the required files
Download the `docker-compose.yml` file and the `example.env` template provided by **Immich**. The `example.env` file will be saved as `.env`:

- **Get `docker-compose.yml` file:**
```bash
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
```
- **Get `example.env` file**
```bash
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
```
### 2.2.3 Populate the .env file with custom values
Default environmental variable content:
```bash
# You can find documentation for all the supported env variables at https://docs.immich.app/install/environment-variables

# The location where your uploaded files are stored
UPLOAD_LOCATION=./library

# The location where your database files are stored. Network shares are not supported for the database
DB_DATA_LOCATION=./postgres

# To set a timezone, uncomment the next line and change Etc/UTC to a TZ identifier from this list: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones#List
# TZ=Etc/UTC

# The Immich version to use. You can pin this to a specific version like "v2.1.0"
IMMICH_VERSION=v3

# Connection secret for postgres. You should change it to a random password
# Please use only the characters `A-Za-z0-9`, without special characters or spaces
DB_PASSWORD=postgres

# The values below this line do not need to be changed
###################################################################################
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
```

- **`UPLOAD_LOCATION`** — set this to the new storage location.
In my case it's `UPLOAD_LOCATION=/mnt/image_storage/library`
- **`DB_DATA_LOCATION`** — leave the default.
- **`TZ`** — optional. You can set your timezone by uncommenting the `TZ=` line
- **`IMMICH_VERSION`** — leave the default value from the downloaded `.env` file.
- **`DB_PASSWORD`** — you should change the default *"postgres"* password to a random password.
- **`DB_USERNAME`** and **`DB_DATABASE_NAME`** can be left unchanged.

---

## 2.3 Start the containers
From the created directory, where the `docker-compose.yml` and `.env` files are located (`immich-app` in my case), run the following command to start **Immich** in the background:

```bash
docker compose up -d
```
This will start all required Immich containers as background services.

![Start Docker](https://github.com/MikeMilenk/Immich-deployment/blob/9c14ca013e8441521f381e5e7ab2d5ddf2b660bf/images/Docker%20up.png)

---

# 3. Remote Access to Immich
At this point, you have 2 options for accessing your **Immich server**.

## 3.1 Option 1 — Local Network
You can access Immich using the local IP address of your Ubuntu Server while connected to your home network.

### 3.1.1 Access Immich via Web Interface
Simply enter the local IP address of your Ubuntu Server followed by port 2283 in your web browser:
```text
http://192.168.x.x:2283
```
Click **Getting Started** button. On the first login, Immich will prompt you to create the administrator account.

![Immich Web Interface Initial Admin Registration](https://github.com/MikeMilenk/Immich-deployment/blob/9c14ca013e8441521f381e5e7ab2d5ddf2b660bf/images/WEB%20Initial%20admin%20registration.webp)

### 3.1.2 Access Immich from the Mobile App

Once you have created the administrator account, you can access your Immich server using the official mobile app.

The mobile app can be downloaded from the following places:
- [Apple App Store](https://apps.apple.com/us/app/immich/id1613945652)
- [Google Play Store](https://play.google.com/store/apps/details?id=app.alextran.immich)
- [GitHub Releases (APK)](https://github.com/immich-app/immich/releases)
- Obtainium: You can get your Obtainium config link from the [Utilities page of your Immich server](https://my.immich.app/utilities).
- [F-Droid](https://app.futo.org/fdroid/repo/)

When signing in, enter the same **server address** you used in the web browser in the **`Server Endpoint URL`** field:

```text
http://192.168.x.x:2283
```

![Immich Mobile App](https://github.com/MikeMilenk/Immich-deployment/blob/9c14ca013e8441521f381e5e7ab2d5ddf2b660bf/images/Mobile%20app%20-%20Initial%20Login.jpg)

Sign in using your Immich account credentials. You can now access your existing photo library from your phone and upload new photos directly from your device.


## 3.2 Option 2 — Remote Access via VPN
If you want to access your **Immich server** from outside your home network, you can use a VPN. For this setup, I use `Tailscale`, a VPN based on WireGuard.

To access Immich remotely, Tailscale needs to be installed and configured on both:
- the device you want to use to access Immich remotely (such as a phone or computer)
- the Ubuntu Server running Immich

For instructions on installing Tailscale on Ubuntu Server, follow this guide:
> **[Install Tailscale on Ubuntu Server](https://github.com/MikeMilenk/Installing-Tailscale-on-Proxmox.git)**

Once both devices are connected to the same Tailscale network, you can access Immich using the Tailscale IP address of your Ubuntu Server:
```text
http://100.x.x.x:2283
```

