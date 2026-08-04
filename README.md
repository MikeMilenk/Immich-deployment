# Build Your Own Media Cloud with Immich

In this guide, I'll show you how to build your own private photo and video cloud using **Immich**, similar to **iCloud Photos** or **Google Photos**. You need first [deploy an Ubuntu Server VM on **Proxmox VE**](https://github.com/MikeMilenk/Immich-deployment), then install Immich and configure it to store all photos on a dedicated storage pool.

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

# Create the ZFS Storage Pool

### 1. Open the ZFS Management Page

In the Proxmox web interface, select your node (**PVE**) and navigate to:

```text
PVE → Disks → ZFS
```

Click **Create: ZFS**.

I named it **`immich-zfs`** for easier tracking in the future. Select the drives you want to combine. Leave all other settings at their default values. Click **Create** and wait for Proxmox to finish creating the pool.

![Create the ZFS Storage Pool](https://github.com/MikeMilenk/Immich-deployment/blob/83dcf0bc14420e3a2638382538ff980df59454ee/images/immich-zfs.png)

---

### 2. Add the Pool as Proxmox Storage

Once the pool has been created, navigate to:

```text
Datacenter → Storage
```
Configure the storage:

> Click **Add**.

> Click **Add → ZFS**.

I set the **ID** to `immich`. For **ZFS Pool**, select the pool created in the previous step. In my case it's **`immich-zfs`**. For **Content**, select **`Disk Image`**. Immich runs inside a VM, and virtual machines require block storage for their virtual disks. **`Container`** content type is intended for different purposes and isn't necessary for an Immich VM.

![Add the Pool as Proxmox Storage](https://github.com/MikeMilenk/Immich-deployment/blob/83dcf0bc14420e3a2638382538ff980df59454ee/images/immich%20disk.png)

---

### 3. Verify the Configuration

After saving, **`immich-zfs`** will appear in the Storage list and can now be selected when creating VMs, containers, or storing apps data. You can verify that it was created successfully by checking its status from the Proxmox terminal:

```bash
zpool status
```

If the pool was created correctly, the output should display the pool name, its current health status (`ONLINE`), and the disks that are part of the pool.

At this point, the SSD continues hosting Proxmox VE and will host future Linux server (where Immich Cloud will be deployed), while your Immich data will be stored on the new **immich-zfs** pool.

![zfs pool status](https://github.com/MikeMilenk/Immich-deployment/blob/b9b092946b3f18329b03df9a5c8b804e5b498ac7/images/zfs%20pools%20status.png)

# Set up the server
After creating and configuring the **ZFS pool** in **Proxmox**, we move to our Ubuntu Server VM.

This is where we will install **Docker** and **Immich**. The setup will look like this:
`Proxmox → ZFS Pool → Ubuntu Server VM → Docker → Immich`

### 1. Prepare the combined HDD storage for Immich
As I indicated previously, I want Immich media to be stored on separate HDDs instead of the SSD where the Ubuntu VM is located. Let's add the HDD storage to the Ubuntu VM. In Proxmox, go to:
`VM → Hardware → Add → Hard Disk`
> Bus: `SCSI 1`
> Storage: `immich` — our existing Proxmox ZFS storage backed by the `immich-zfs` pool.
> Disk size (GiB): `1300`

`*NOTE: I have **1.45 TB** of available storage, which is approximately **1350 GiB**. I will allocate **1300 GiB** and leave approximately **50 GiB** of free space to avoid filling the storage completely and to leave some room for normal ZFS operations and future growth.*`

Click `Add`.

This attaches a virtual hard disk backed by our ZFS pool to the Ubuntu VM. The disk will then appear inside Ubuntu and can be partitioned, formatted, and mounted for Immich storage.

### 2.Create the Immich Directory and navigate into it
Create a directory of your choice (I named it as `immich-app`)
```bash
mkdir ./immich-app
cd ./immich-app
```

### 3.Download the required files
We need to keep the `docker-compose.yml` and `.env` files in the `immich-app` directory.
Download the `docker-compose.yml` file and the `example.env` template provided by **Immich**. The `example.env` file will be saved as `.env`:

> **Get `docker-compose.yml` file:**
```bash
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
```
> **Get `example.env` file**
```bash
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
```
### 4.Populate the .env file with custom values
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

