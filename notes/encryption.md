That's a very detailed and well-structured installation guide\! To incorporate **full disk encryption** (specifically for your root and home partitions) using **LUKS** while keeping **systemd-boot**, you need to make modifications to three key areas:

1.  **Partitioning and Formatting** (Encrypt the partitions).
2.  **Base Installation** (Install necessary tools).
3.  **Initial RAM Disk (Initramfs)** and **Bootloader Configuration** (Add `encrypt` hook and update boot entries).

Here are the specific changes to your steps:

## 🔒 Encrypting Root and Home Partitions with LUKS

I'll outline the modified steps, focusing on your current partitions:

- `/dev/nvme0n1p1`: EFI System Partition (unencrypted)
- `/dev/nvme0n1p2`: Root partition (to be encrypted)
- `/dev/nvme0n1p3`: Home partition (to be encrypted)

---

### 1\. Modified Step 4: Formatting and Encryption

Instead of directly formatting with `mkfs.ext4`, you first encrypt the partitions using **LUKS** (Linux Unified Key Setup) and then format the resulting decrypted devices.

| Action   | Old Command                      | New Command(s)                               | Notes                                                                                                             |
| :------- | :------------------------------- | :------------------------------------------- | :---------------------------------------------------------------------------------------------------------------- |
| **EFI**  | `$ mkfs.fat -F32 /dev/nvme0n1p1` | **No Change**                                | Unencrypted, as it holds the bootloader and kernels.                                                              |
| **Root** | `$ mkfs.ext4 /dev/nvme0n1p2`     | `$ cryptsetup luksFormat /dev/nvme0n1p2`     | Encrypts the partition. You will be asked to set a **passphrase**.                                                |
| **Root** | (N/A)                            | `$ cryptsetup open /dev/nvme0n1p2 cryptroot` | Opens the encrypted partition and maps it to a device named `cryptroot` in `/dev/mapper/`.                        |
| **Root** | (N/A)                            | `$ mkfs.ext4 /dev/mapper/cryptroot`          | Formats the decrypted volume.                                                                                     |
| **Home** | `$ mkfs.ext4 /dev/nvme0n1p3`     | `$ cryptsetup luksFormat /dev/nvme0n1p3`     | Encrypts the partition. **Recommendation**: Use a different passphrase than root, or handle key management later. |
| **Home** | (N/A)                            | `$ cryptsetup open /dev/nvme0n1p3 crypthome` | Opens the encrypted partition and maps it to a device named `crypthome`.                                          |
| **Home** | (N/A)                            | `$ mkfs.ext4 /dev/mapper/crypthome`          | Formats the decrypted volume.                                                                                     |

---

### 2\. Modified Step 5: Mounting Disks

You now mount the decrypted devices (`/dev/mapper/cryptroot` and `/dev/mapper/crypthome`), not the raw partitions.

```bash
# 1. Mount the decrypted root volume
$ mount /dev/mapper/cryptroot /mnt

# 2. Mount the EFI partition (No change)
$ mkdir -p /mnt/boot/
$ mount /dev/nvme0n1p1 /mnt/boot/

# 3. Mount the decrypted home volume
$ mkdir -p /mnt/home
$ mount /dev/mapper/crypthome /mnt/home
```

---

### 3\. Modified Step 6: Base Installation

You need the `cryptsetup` tool for managing LUKS volumes.

```bash
# for systemdboot,
$ pacstrap /mnt linux linux-lts linux-zen linux-headers linux-firmware base base-devel efibootmgr networkmanager neovim mtools dosfstools **cryptsetup** intel-ucode intel-media-driver vulkan-intel intel-gmmlib sof-firmware
```

---

### 4\. New Step: Configure Initramfs ⚙️

This is crucial. The initial RAM disk environment (`initramfs`) must contain the tools to decrypt and mount the root partition before the full system loads. This is done by adding the `encrypt` hook.

Run this **after Step 8 (`arch-chroot /mnt`)**:

```bash
$ nvim /etc/mkinitcpio.conf
```

Find the `HOOKS` line and insert `encrypt` **after** `block` and **before** `filesystems`:

```conf
HOOKS="base udev autodetect modconf block **encrypt** filesystems keyboard fsck"
```

Then, regenerate the initramfs images for all installed kernels:

```bash
$ mkinitcpio -P
```

---

### 5\. New Step: Configure `crypttab` (Optional but Recommended)

You can use the `/etc/crypttab` file to manage the decryption of your non-root encrypted partitions (like `/home`) using the kernel's boot process. This simplifies the `/etc/fstab`.

Get the **UUIDs** of the encrypted partitions (`/dev/nvme0n1p2` and `/dev/nvme0n1p3`):

```bash
$ blkid /dev/nvme0n1p2
$ blkid /dev/nvme0n1p3
```

Create and edit `/etc/crypttab`:

```bash
$ nvim /etc/crypttab
```

Add an entry for your home partition. You usually _don't_ need an entry for the root partition here, as it's handled by the bootloader's kernel command line and the `encrypt` hook.

```conf
# <name>        <device>                                    <password/keyfile>    <options>
crypthome       UUID=<UUID of /dev/nvme0n1p3>               none                  luks
```

Replace `<UUID of /dev/nvme0n1p3>` with the actual UUID from the `blkid` output (e.g., `UUID=a1b2c3d4-e5f6-7890-a1b2-c3d4e5f67890`).

---

### 6\. Modified Step 7: Generate fstab

The `genfstab` command should automatically pick up the **decrypted device names** (`/dev/mapper/cryptroot` and `/dev/mapper/crypthome`), which is what you want.

**Ensure `genfstab` is run AFTER all partitions are mounted.**

```bash
$ genfstab -U /mnt >> /mnt/etc/fstab
$ cat /mnt/etc/fstab
```

The output should show:

- `/dev/mapper/cryptroot` mounted on `/`
- `/dev/mapper/crypthome` mounted on `/home`
- `/dev/nvme0n1p1` mounted on `/boot`

---

### 7\. Modified Step 15: Setting up Bootloader (systemd-boot)

You must modify your boot entries to tell the kernel where the encrypted root partition is and that it needs to be decrypted.

In all your `/boot/loader/entries/*.conf` files (e.g., `arch.conf`, `arch-lts.conf`, etc.), change the `options` line.

**Find the UUID of the encrypted ROOT partition (`/dev/nvme0n1p2`):**

```bash
$ blkid /dev/nvme0n1p2
```

Use the **UUID of the raw encrypted partition** (`/dev/nvme0n1p2`), **not** the UUID of the filesystems inside the decrypted volume.

**Change the `options` line in ALL kernel entries (arch, lts, zen, etc.):**

| Old Options                                       | New Options                                                                                                      |
| :------------------------------------------------ | :--------------------------------------------------------------------------------------------------------------- |
| `options root=/dev/nvme0n1p2 rw quiet loglevel=3` | `options **cryptdevice=UUID=<UUID of /dev/nvme0n1p2>:cryptroot** root=/dev/mapper/cryptroot rw quiet loglevel=3` |

Example for `arch.conf`:

```conf
# arch.conf
title	Arch Linux
linux	/vmlinuz-linux
initrd	/intel-ucode.img
initrd	/initramfs-linux.img
options	cryptdevice=UUID=<UUID of /dev/nvme0n1p2>:cryptroot root=/dev/mapper/cryptroot rw quiet loglevel=3
```

Replace `<UUID of /dev/nvme0n1p2>` with the actual UUID.

---

### 8\. Final Step 16: Unmounting

Before rebooting, ensure you **close the LUKS volumes** in addition to unmounting.

```bash
$ exit

$ umount -R /mnt
# Unmounts /mnt/boot, /mnt/home, and /mnt

$ cryptsetup close crypthome
$ cryptsetup close cryptroot

$ reboot
```

The system will now prompt you for the root passphrase during boot (handled by the `encrypt` hook in `initramfs`), and then it will proceed to boot normally, automatically mounting your encrypted `/home` via `crypttab` and `/etc/fstab`.

Would you like me to elaborate on any of these specific steps, such as key file usage for more complex setups?

---

Choosing to reinstall your OS while keeping your LUKS-encrypted partitions is absolutely possible and is one of the key benefits of using separate partitions for `/` and `/home`.

The process is essentially the same as your initial install, but with two critical differences: **You skip the formatting step for the LUKS partitions, and you skip the partitioning step entirely.**

Here are the modified steps for a smooth OS reinstall (reusing `/dev/nvme0n1p2` for root and `/dev/nvme0n1p3` for home):

---

## 🔁 Reinstalling Arch on Existing LUKS Partitions

### 1\. Decrypt and Open Volumes 🔑

Do **NOT** run `cryptsetup luksFormat` on the partitions you want to keep (`/dev/nvme0n1p2` and `/dev/nvme0n1p3`), or you will destroy your data.

1.  **Unlock the Root Partition:**

    ```bash
    $ cryptsetup open /dev/nvme0n1p2 cryptroot
    ```

    (Enter your LUKS passphrase for root)

2.  **Unlock the Home Partition:**

    ```bash
    $ cryptsetup open /dev/nvme0n1p3 crypthome
    ```

    (Enter your LUKS passphrase for home)

### 2\. Format the Root Filesystem Only ⚠️

Since you are reinstalling the OS, you must **format the root partition's filesystem** to wipe the old OS files. You will **NOT** format the `/home` filesystem to preserve your data.

1.  **Format the decrypted root device:**

    ```bash
    $ mkfs.ext4 /dev/mapper/cryptroot
    ```

    (This step erases the old root files, but your `/home` data is safe.)

2.  **Do NOT run `mkfs.ext4 /dev/mapper/crypthome`**

### 3\. Mount Partitions 💾

You will mount the existing partitions, including the EFI partition and the decrypted LUKS volumes.

```bash
# 1. Mount the newly formatted (decrypted) root
$ mount /dev/mapper/cryptroot /mnt

# 2. Mount the unencrypted EFI partition (no change)
$ mkdir -p /mnt/boot/
$ mount /dev/nvme0n1p1 /mnt/boot/

# 3. Mount the preserved (decrypted) home partition
$ mkdir -p /mnt/home
$ mount /dev/mapper/crypthome /mnt/home
```

### 4\. Continue with Steps 6-15 (Standard Install)

You can now continue your standard installation process, as the environment is set up just like a fresh install:

- **Step 6: Base Installation:** Re-run `pacstrap` (this installs new kernel images and the base OS files into the freshly formatted `/dev/mapper/cryptroot`).
- **Step 7: Generate fstab:** Re-run `genfstab` to generate a fresh file using the current UUIDs of the decrypted mapper devices.
- **Step 8: Chroot** and follow all configuration steps.
- **Initramfs:** This is crucial—remember to **re-add the `encrypt` hook** to `/etc/mkinitcpio.conf` and **re-run `mkinitcpio -P`**.
- **Step 15: systemd-boot:** Re-run `bootctl --path=/boot install`. You **must** then **recreate your kernel entry files** (`arch.conf`, etc.) in `/boot/loader/entries/` and ensure the `options` line still points to the correct **UUID of the raw encrypted root partition** (`/dev/nvme0n1p2`).

### 5\. Post-Reboot Cleanup (User ID)

When you create your user account in the new install (Step 14), make sure you use the **exact same username** as before. When you log in, your user ID (UID) will match the ownership of the files in the preserved `/home` partition, and everything will work seamlessly.

If you have any trouble finding the original UUIDs or want a specific command check, let me know\!
