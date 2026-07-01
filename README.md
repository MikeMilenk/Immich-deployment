# Build Your Own Media Cloud with Immich

In this guide, I'll show you how to build your own private photo and video cloud using **Immich**, similar to **iCloud Photos** or **Google Photos**. We'll first deploy an Ubuntu Server VM on **Proxmox VE**, then install Immich and configure it to store all photos on a dedicated storage pool.

**Immich** is a self-hosted photo and video management system running on your own server. It allows you to:
* automatically back up photos from your phone
* store all media on your own infrastructure
* organize photos into albums
* search by metadata, location, and content
* use face and object recognition features
* access your library via web and mobile apps

In short, Immich gives you the convenience of a modern cloud photo service while keeping full control of your data on your own hardware.

# Preparing Storage for Immich Media

My server contains 2 SSDs and 2 HDDs. The SSD is used only for Proxmox VE and Linux ISO images, while the two HDDs are combined into a single **1.5 TB** ZFS storage pool called **`immich-zfs`**. This keeps the OS separate from the photo library and provides one large location for all app data.

![Server Drives](https://github.com/MikeMilenk/Immich-deployment/blob/83dcf0bc14420e3a2638382538ff980df59454ee/images/lsblk.png)

---

# Create the ZFS Storage Pool

### 1. Open the ZFS Management Page

In the Proxmox web interface, select your node (**PVE**) and navigate to:

```text
PVE → Disks → ZFS
```

Click **Create: ZFS**.

I named it **immich-zfs** for easier tracking in the future. Select the drives you want to combine. Leave all other settings at their default values. Click **Create** and wait for Proxmox to finish creating the pool.

![Create the ZFS Storage Pool](https://github.com/MikeMilenk/Immich-deployment/blob/83dcf0bc14420e3a2638382538ff980df59454ee/images/immich-zfs.png)

---

### 2. Add the Pool as Proxmox Storage

Once the pool has been created, navigate to:

```text
Datacenter → Storage
```
Configure the storage.

Click **Add**.

Click **Add → ZFS**.

I set the **ID** to `immich`. For **ZFS Pool**, select the pool created in the previous step. In my case it's **immich-zfs**. For **Content**, select **Disk Image**. Immich runs inside a VM, and virtual machines require block storage for their virtual disks. **Container** content type is intended for different purposes and isn't necessary for an Immich VM.

![Add the Pool as Proxmox Storage](https://github.com/MikeMilenk/Immich-deployment/blob/83dcf0bc14420e3a2638382538ff980df59454ee/images/immich%20disk.png)

---

### 3. Verify the Configuration

After saving, **immich-zfs** will appear in the Storage list and can now be selected when creating VMs, containers, or storing apps data. You can verify that it was created successfully by checking its status from the Proxmox terminal:

```bash
zpool status immich-zfs
```

If the pool was created correctly, the output should display the pool name, its current health status (`ONLINE`), and the disks that are part of the pool.

At this point, the SSD continues hosting Proxmox VE and will host future Linux server (where Immich Cloud will be deployed), while your Immich data will be stored on the new **immich-zfs** pool.



