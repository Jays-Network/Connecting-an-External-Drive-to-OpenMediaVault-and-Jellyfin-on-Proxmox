# Connecting an External Drive to OpenMediaVault and Jellyfin on Proxmox

This guide provides a step-by-step walkthrough on how to connect an external hard drive to a Proxmox host, pass it through to an OpenMediaVault (OMV) virtual machine, and then make it accessible to a Jellyfin container for your media library.

**Disclaimer:** ⚠️ This guide involves formatting and wiping hard drives. Please back up any important data before proceeding. We are not responsible for any data loss.

## Prerequisites

- A working Proxmox installation.
- An OpenMediaVault VM running on Proxmox.
- A Jellyfin container running on Proxmox.
- An external hard drive that you want to use for your media.

---

## Step 1: Connecting the External Drive to OpenMediaVault on Proxmox

This section covers the initial setup of connecting your external drive to the Proxmox host and passing it through to your OpenMediaVault VM.

### 1. Identify and Wipe the Drive

First, you need to identify your external drive and wipe any existing partitions.

1.  Connect your external drive to the Proxmox host.
2.  In the Proxmox web interface, navigate to your **Node > Disks**.
3.  Identify your external drive in the list.
4.  If the drive has any existing partitions, select the drive and click **Wipe Disk**.

### 2. Create a New Partition

Next, you'll create a new partition on the drive using the command line.

1.  Open the Proxmox shell for your node.
2.  Use the `lsblk` command to list your block devices and confirm the name of your external drive (e.g., `/dev/sdb`).
    ```bash
    lsblk
    ```
3.  Use `cfdisk` to create a new partition. Replace `/dev/sdb` with your drive's name.
    ```bash
    cfdisk /dev/sdb
    ```
4.  In the `cfdisk` interface:
    - Select `GPT` as the label type.
    - Select `New` and press Enter to create a new partition.
    - Select `Write`, type `yes`, and press Enter to save the changes.
    - Select `Quit` to exit.

### 3. Get the Drive's UUID

Now, you need to get the UUID (Universally Unique Identifier) of the newly created partition.

1.  In the Proxmox shell, run the `blkid` command:
    ```bash
    blkid
    ```
2.  Find your drive in the list and copy the `UUID` value. You will need this in the next step.

### 4. Pass the Drive to the OpenMediaVault VM

Finally, you will use the `qm set` command to pass the drive through to your OMV virtual machine.

1.  In the Proxmox shell, use the following command. Replace `102` with your OMV VM ID, `sata1` with the desired SATA port (if `sata0` is already in use), and the UUID with the one you copied earlier.
    ```bash
    qm set 102 -sata1 /dev/disk/by-uuid/YOUR_UUID_HERE
    ```
2.  You can verify that the drive has been passed through by going to the **Hardware** section of your OMV VM in the Proxmox web interface.

---

## Step 2: Passing the External Drive from OpenMediaVault to Jellyfin

This section explains how to configure OMV to share the drive and then how to mount that share in your Jellyfin container.

### 1. Configure OpenMediaVault

After passing the drive to the OMV VM, you need to format it and create a shared folder within the OpenMediaVault web interface.

1.  Log in to your OpenMediaVault web interface.
2.  Go to **Storage > File Systems**.
3.  Click the **+** icon and select **Create**.
4.  Select your new drive, give it a label, choose a file system (e.g., `ext4`), and click **Save**.
5.  Go to **Storage > Shared Folders**.
6.  Click **+ Add**, give the folder a name, select your new file system, and set the permissions.
7.  Go to **Services > SMB/CIFS > Shares**.
8.  Click **+ Add**, select your shared folder, and configure the SMB share settings.

### 2. Mount the OMV Share on Proxmox

Now, you will mount the SMB share you just created onto the Proxmox host.

1.  Open the Proxmox shell.
2.  Create a directory to mount the share to:
    ```bash
    mkdir /mnt/MiniPC
    ```
3.  Install the `cifs-utils` package:
    ```bash
    apt-get install cifs-utils
    ```
4.  Mount the OMV share using the following command. Replace the user, IP address, share name, and mount point with your own information.
    ```bash
    mount -t cifs -o user=SMBuser //192.168.20.202/MiniPC /mnt/MiniPC
    ```
5.  You will be prompted to enter the password for your SMB user.

### 3. Bind Mount to the Jellyfin Container

The final step is to bind mount the Proxmox mount point to your Jellyfin container.

1.  In the Proxmox shell, use the `pct set` command. Replace `203` with your Jellyfin container ID.
    ```bash
    pct set 203 -mp0 /mnt/MiniPC,mp=/sharedfolder
    ```
    This command maps the `/mnt/MiniPC` directory on the Proxmox host to the `/sharedfolder` directory inside the Jellyfin container.

2.  Reboot your Jellyfin container from the Proxmox web interface.

3.  Open the Jellyfin web interface and add a new library. When you browse for a folder, you should now see `/sharedfolder` as an option.

---

## Conclusion

You have successfully connected an external drive to your Proxmox host, passed it through to an OpenMediaVault VM, created an SMB share, and mounted it in your Jellyfin container. Your Jellyfin server can now access the media on your external drive.
