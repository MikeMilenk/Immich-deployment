# Build Your Own Media Cloud with Immich

In this guide, I'll show you how to build your own private photo and video cloud using **Immich**, similar to **iCloud Photos** or **Google Photos**. We'll first deploy an Ubuntu Server VM on **Proxmox VE**, then install Immich and configure it to store all photos on a dedicated storage pool.

My server contains 2 SSDs and 2 HDDs. The SSD is used only for Proxmox VE and Linux ISO images, while the two HDDs are combined into a single **1.5 TB** ZFS storage pool called **`immich-zfs`**. This keeps the OS separate from the photo library and provides one large location for all app data.

---

# Create the ZFS Storage Pool

### 1. Open the ZFS Management Page

In the Proxmox web interface, select your node (**PVE**) and navigate to:

```text
PVE → Disks → ZFS
```

Click **Create: ZFS**.

I named it **immich-zfs** for easier tracking in the future. Select the drives you want to combine. Leave all other settings at their default values. Click **Create** and wait for Proxmox to finish creating the pool.

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

---

### 3. Verify the Configuration

After saving, **immich-zfs** will appear in the Storage list and can now be selected when creating VMs, containers, or storing apps data.

At this point, the SSD continues hosting Proxmox VE and ISO images, while Immich data will be stored on the new **immich-zfs** pool.
