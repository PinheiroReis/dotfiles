# Installing Arch Linux with Full Disk Encryption, BTRFS, Zram, GRUB, and Systemd-based Initramfs

## Includes Guides To:

- **LUKS encryption** (included)
- **BTRFS filesystem with subvolumes** (included)
- **UEFI mode** (included)
- **Systemd-based initramfs** (included)
- **Zram implementation** (included)
- **Autologin with `greetd`** (included)

If you're only interested in installing Linux and not setting up dual boot with Windows, feel free to skip the Windows-related sections.

## Prepare the System

Before we dive into the installation process, let's ensure that your system is ready:

- **Data Backup:** Make sure you've backed up all your important data. We're about to make significant changes, and it's always wise to have a safety net.

- **UEFI Mode:** In your system's firmware settings (BIOS/UEFI), enable the UEFI boot mode. Note that this option may be unavailable on very old hardware.

- **Secure Boot:** You must disable Secure Boot in your UEFI settings to allow the unsigned Arch Linux installation media to boot.


## Prepare the USB Drive

- **Ventoy Installation:** Start by installing [Ventoy](https://github.com/ventoy/Ventoy) on your USB drive. Ventoy is a versatile tool that allows you to easily create a multi-boot USB drive.
- **Download Arch ISO:** Head to [Arch Linux's official website](https://www.archlinux.org/download/) and download the Arch ISO image. Copy it to your USB drive.

## Disk and Filesystem Structure

The installation process will create the following physical partition scheme on your drive:

| Partition          | Size    | Mount Point | Filesystem | Notes                      |
| ------------------ | ------- | ----------- | ---------- | -------------------------- |
| `/dev/<efi-disk>`  | +512M   | `/boot/efi` | FAT32      | EFI System Partition       |
| `/dev/<boot-disk>` | +1GB    | `/boot`     | ext4       | Unencrypted boot partition |

Inside the LUKS container (`/dev/mapper/cryptroot`), we will set up a BTRFS filesystem with the following subvolume layout:

| BTRFS Subvolume | Mount Point             | Purpose                     |
| --------------- | ----------------------- | --------------------------- |
| `@`             | `/`                     | The root filesystem         |
| `@home`         | `/home`                 | User home directories       |
| `@log`          | `/var/log`              | System log files            |
| `@pkg`          | `/var/cache/pacman/pkg` | Pacman package cache        |
| `@.snapshots`   | `/.snapshots`           | For storing BTRFS snapshots |

Please be aware that these names should be substituted with the actual device paths relevant to your system configuration:

| Device         | In this Doc        | Examples                      |
|:-------------- | ------------------ |:----------------------------- |
| Disk Device    | `/dev/<your-disk>` | `/dev/sda`, `/dev/nvme0n1`    |
| EFI Partition  | `/dev/<efi-disk>`  | `/dev/sda5`, `/dev/nvme0n1p1` |
| Boot Partition | `/dev/<boot-disk>` | `/dev/sda6`, `/dev/nvme0n1p2` |
| LUKS Partition | `/dev/<luks-disk>` | `/dev/sda7`, `/dev/nvme0n1p3` |

## Install Arch Linux

1. Connect the USB drive and boot from the Arch Linux ISO.

2. Set your keyboard layout:
   
   ```shell
   loadkeys <keyboard-layout>
   ```

3. Set pacman configs, where "number" could be what you want, but not too high:
   
   ```shell
   vim /etc/pacman.conf
   
   # Uncomment and modify:
   # ParallelDownloads = <number>
   ```

4. Make sure the system is booted in UEFI mode. The following command should display the directory contents without error:
   
   ```shell
   ls /sys/firmware/efi/efivars
   ```

5. Connect to the internet. A wired connection is preferred since it's easier to connect. [More info](https://wiki.archlinux.org/index.php/Installation_guide#Connect_to_the_internet)
   
   **Note on Device Names:** Before partitioning, identify your disk's name. It will likely be `/dev/sda` or `/dev/vda` for SATA drives (including SSDs and HDDs) or `/dev/nvme0n1` for NVMe drives. Use `lsblk` to list block devices and find the correct name for your system. The guide will use `<your-disk>` as a placeholder.

6. Run `fdisk` and follow until step 11 to create Linux partitions:
   
   ```shell
   fdisk /dev/<your-disk>
   ```

7. Create an empty GPT partition table using the `g` command. (**WARNING:** This will erase the entire disk.)
   
       Command (m for help): g
       Created a new GPT disklabel (GUID: ...).

8. Create the EFI partition (`/dev/<efi-disk>`):
   
       Command (m for help): n
       Partition number: <Press Enter>
       First sector: <Press Enter>
       Last sector, +/-sectors or +/-size{K,M,G,T,P}: +512M
       
       Command (m for help): t
       Partition type or alias (type L to list all): uefi

9. Create the Boot partition (`/dev/<boot-disk>`):
   
       Command (m for help): n
       Partition number: <Press Enter>
       First sector: <Press Enter>
       Last sector, +/-sectors or +/-size{K,M,G,T,P}: +1GB
       
       Command (m for help): t
       Partition type or alias (type L to list all): linux

10. Create the LUKS partition (`/dev/<luks-disk>`):
    
        Command (m for help): n
        Partition number: <Press Enter>
        First sector: <Press Enter>
        Last sector, +/-sectors or +/-size{K,M,G,T,P}: <Press Enter>
        
        Command (m for help): t
        Partition type or alias (type L to list all): linux

11. Print the partition table using the `p` command and check that everything is OK:
    
        Command (m for help): p

12. Write changes to the disk using the `w` command. (Make sure you know what you're doing before running this command).
    
        Command (m for help): w

13. Format the EFI and Boot Partitions:
    
    ```shell
    mkfs.fat -F 32 /dev/<efi-disk>
    mkfs.ext4 /dev/<boot-disk>
    ```

14. Set up the encrypted partition. This will contain your BTRFS filesystem. Let's call the mapped device `cryptroot` for clarity.
    
    ```shell
    cryptsetup --use-urandom luksFormat /dev/<luks-disk>
    cryptsetup open /dev/<luks-disk> cryptroot
    ```

15. Format the encrypted partition with BTRFS. The `-L` flag sets a label for the filesystem.
    
    ```shell
    mkfs.btrfs -L Arch /dev/mapper/cryptroot
    ```

16. Create BTRFS Subvolumes.
    
    First, mount the encrypted BTRFS volume to a temporary directory:
    
    ```shell
    mount /dev/mapper/cryptroot /mnt
    ```
    
    Use the `btrfs` command to create the subvolumes. The `@` prefix is a common convention to distinguish them from regular directories.
    
    ```shell
    btrfs subvolume create /mnt/@
    btrfs subvolume create /mnt/@home
    btrfs subvolume create /mnt/@log
    btrfs subvolume create /mnt/@pkg
    btrfs subvolume create /mnt/@.snapshots
    ```
    
    Now that the subvolumes are created, unmount the top-level volume:
    
    ```shell
    umount /mnt
    ```

17. Mount the Core Filesystems.
    
    Now we will mount our newly created subvolumes, along with the boot partitions, to their final destinations under `/mnt`.
    
    **Note on Mount Options:**
    
    - `compress=zstd`: Enables transparent compression.
    - `noatime`: Improves performance by not writing file access times.
    - `ssd`: Use this if you are installing on an NVMe or a SATA SSD. **Omit this option for traditional Hard Disk Drives (HDDs).**
    
    Mount the root subvolume (`@`) to `/mnt`. The following command is for an SSD/NVMe:
    
    ```shell
    mount -o compress=zstd,ssd,noatime,subvol=@ /dev/mapper/cryptroot /mnt
    ```
    
    Next, mount the boot and EFI partitions:
    
    ```shell
    mount --mkdir /dev/<boot-disk> /mnt/boot
    mount --mkdir /dev/<efi-disk> /mnt/boot/efi
    ```

18. Mount the Remaining BTRFS Subvolumes.
    
    First, create the necessary directories for the subvolume mount points:
    
    ```shell
    mkdir -p /mnt/{home,var/log,var/cache/pacman/pkg,.snapshots}
    ```
    
    Now, mount the remaining subvolumes. Remember to omit the `ssd` option if you are on an HDD.
    
    ```shell
    mount -o compress=zstd,ssd,noatime,subvol=@home /dev/mapper/cryptroot /mnt/home
    mount -o compress=zstd,ssd,noatime,subvol=@log /dev/mapper/cryptroot /mnt/var/log
    mount -o compress=zstd,ssd,noatime,subvol=@pkg /dev/mapper/cryptroot /mnt/var/cache/pacman/pkg
    mount -o compress=zstd,ssd,noatime,subvol=@.snapshots /dev/mapper/cryptroot /mnt/.snapshots
    ```

19. Install the base system. We will also install microcode (for CPU bug fixes) and some useful packages like `git`, `vim`, and `sudo`. **Choose the correct microcode package for your CPU**.
    
    ```shell
    # For AMD CPUs:
    pacstrap -K /mnt base base-devel linux linux-firmware amd-ucode btrfs-progs mesa plymouth openssh git vim sudo
    
    # For Intel CPUs:
    pacstrap -K /mnt base base-devel linux linux-firmware intel-ucode btrfs-progs mesa plymouth openssh git vim sudo
    ```

20. Generate `/etc/fstab`. This file can be used to define how disk partitions, various other block devices, or remote filesystems should be mounted into the filesystem:
    
    ```shell
    genfstab -U /mnt > /mnt/etc/fstab
    
    # Check with
    cat /mnt/etc/fstab
    ```

21. Enter the new system:
    
    ```shell
    arch-chroot /mnt /bin/bash
    ```

22. Execute `step 3` operation.

23. Set TimeZone:
    
    ```shell
    # See available timezones:
    ls /usr/share/zoneinfo/
    
    # Set timezone (you may should use other):
    ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
    ```

24. Run hwclock(8) to generate `/etc/adjtime`:
    
    ```shell
    hwclock --systohc
    ```

25. Set Locale:
    
    ```shell
    vim /etc/locale.gen 
    # uncomment en_US.UTF-8 UTF-8 or another
    
    locale-gen
    
    echo LANG=en_US.UTF-8 > /etc/locale.conf
    ```

26. Set hostname:
    
    ```shell
    echo YourHostName > /etc/hostname
    ```

27. Create a user:
    
    ```shell
    useradd -m -G wheel --shell /bin/bash YourUserName
    
    passwd YourUserName
    
    visudo
    # Uncomment %wheel ALL=(ALL) ALL
    ```

28. Make keyboard config persistent:
    
    ```shell
    vim /etc/vconsole.conf
    
    # KEYMAP=<keyboard-layout>
    ```

29. Configure `mkinitcpio` with modules needed to create the systemd-based initramfs image:
    
    ```shell
    vim /etc/mkinitcpio.conf
    
    # HOOKS=(base systemd plymouth autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt filesystems fsck)
    ```

30. Recreate the initramfs image:
    
    ```shell
    mkinitcpio -P
    ```

31. Setup GRUB:
    
    ```shell
    pacman -S grub efibootmgr
    
    grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
    ```
    
    In `/etc/default/grub` edit the line `GRUB_CMDLINE_LINUX`. This tells GRUB to unlock the encrypted partition and specifies the root filesystem. Replace `<luks-disk-UUID-code>` with your LUKS partition's UUID (you can find this with `lsblk -f` or `blkid`).
    
        GRUB_CMDLINE_LINUX="rd.luks.name=<luks-disk-UUID-code>=cryptroot root=/dev/mapper/cryptroot rootflags=subvol=@"
    
    Furthermore, for customization, add configs for boot screen:
    
        GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
    
    Now generate the main GRUB configuration file:
    
    ```shell
    grub-mkconfig -o /boot/grub/grub.cfg
    ```

32. Install `networkmanager` package and enable `NetworkManager` service to ensure you have Internet connectivity after rebooting:
    
    ```shell
    pacman -S networkmanager
    systemctl enable NetworkManager
    ```

33. Exit new system and unmount all filesystems:
    
    ```shell
    exit
    umount -R /mnt
    ```

34. Arch is now installed 🎉. Reboot:
    
    ```shell
    reboot
    ```

35. Open BIOS settings and set `GRUB` as first boot priority. Save and exit BIOS settings. After booting the system, you should see the GRUB menu.

36. Reboot again and log in to Arch Linux with your username and password.

37. Check internet connectivity:
    
    ```shell
    ping google.com
    ```

38. Reboot!

## Post Installation

### Setup greetd for autologin

1. Install dependencies:
   
   ```shell
   pacman -Syu greetd
   ```

2. Config session and user:
   
   ```shell
   vim /etc/greetd/config.toml
   ```
   
   Put this:
   
   ```
   [terminal]
   vt = 1
   
   [initial_session]
   user="YourUserName"
   command="Hyprland"
   
   [default_session]
   command = "agreety --cmd /bin/sh"
   user = "greeter"
   ```
   
   

3. Enable `greetd`:
   
   ```shell
   # Don't use "enable --now"
   sudo systemctl enable greetd
   ```

4. Reboot and test.

### Setup `pacman` mirros list and update with `reflector`

1. Install dependencies:
   
   ```shell
    pacman -Syu reflector
   ```

2. Enable `multilib`:
   
    ```
    # uncoment [multilib] section in /etc/pacman.conf
    ```

3. Add config for auto-runs:
   
    ```
    # /etc/xdg/reflector/reflector.conf
    --country 'Brazil','United States' # choose yours
    --protocol https
    --age 12
    --latest 200
    --fastest 10
    --sort rate
    --save /etc/pacman.d/mirrorlist
    ```

4. Enable to auto-run:
   
   ```shell
    systemctl enable --now reflector.service
    systemctl enable --now reflector.timer
   ```

### Basic setup SSH

1. Download dependencies:
   
   ```shell
    pacman -Syu openssh
   ```

2. Add some keys:
   
   ```shell
    ssh-keygen -t ed25519 -C "your@email.com"
   ```

3. Add some config, for GitHub for example (add the public key on GitHub to auto auth):
   
    ```
    # ~/.ssh/config
    Host github.com
       HostName github.com
       User git
       Port 22
       IdentityFile ~/.ssh/id_ed25519_github
       ForwardAgent yes
    ```

### Create Encrypted External Drive

1. Create the partition:
   
   ```
   Command (m for help): n
   Partition number: <Press Enter>
   First sector: <Press Enter>
   Last sector, +/-sectors or +/-size{K,M,G,T,P}: <Press Enter>
   
   Command (m for help): t
   Partition type or alias (type L to list all): linux
   ```

2. Use cryptsetup to encrypt device:
   
   ```shell
   cryptsetup --use-urandom luksFormat /dev/<backup-disk-partition>
   ```

3. Open:
   
   ```shell
   cryptsetup open /dev/<backup-disk-partition> <YourBackupName>
   ```

4. Make the filesystem:
   
   ```shell
   mkfs.ext4 /dev/mapper/<YourBackupName>
   ```

**Optional, just for automation:**

1. Create keyfile:
   
   ```shell
   openssl genrsa -out <path/to/key> 4096
   ```

2. Add key to encrypted device:
   
   ```shell
   cryptsetup luksAddKey /dev/<external-disk> <path/to/key>
   ```

3. Add device to /etc/crypttab for autodecrypt it:
   
   ```shell
   vim /etc/crypttab
   
   # <device-name>       UUID=<device-UUID-code>      <path/to/key>    luks,<options>
   
   # Example don't using keyfile
   # BACKUP      UUID=738c6426-3ef5-48d5-a837-b437c722802f       -       luks
   
   # Example using
   # BACKUP      UUID=73481cae-1b80-400c-bef3-4f4a2b2a9a1e       /root/backup-key        luks
   ```

4. Add the external drive to /etc/fstab to automount (sometimes useless):
   
   ```shell
   # To help you with information about mounted drive (don't simply overwrite fstab)
   genfstab -U /
   
   vim /etc/fstab
   
   # UUID=<device-UUID-code>     <path/to/mount> <type> <options>  <dump>  <fsck>
   
   # For example
   # UUID=8d90233f-36ff-434d-bc5a-de6d596719f1       /run/timeshift/backup   ext4            rw,relatime     0 2
   ```

### Zram Implementation (Recommended)

1. Install the `zram-generator` package:
   
   ```shell
   pacman -Syu zram-generator
   ```

2. Configure zram by creating a configuration file. This example allocates 50% of your RAM memory (or the min of 4096MiB):
   
   ```shell
   # /etc/systemd/zram-generator.conf
   [zram0]
   zram-size = ram / 2
   compression-algorithm = zstd
   ```

3. Use  `sytemctl` to enable the `zram-generator`:
   
   ```shell
   systemctl daemeon-reload
   
   # The number after "zram" may be other 
   systemctl start systemd-zram-setup@zram0 
   ```

4. Reboot, verify the zram device is active:
   
   ```shell
   swapon --show
   ```

### Setting up Snapper

The BTRFS layout in this guide was designed to work correctly with Snapper, a powerful tool for filesystem snapshots. The key is that we created a dedicated subvolume (`@.snapshots`) for storing snapshots and mounted it at `/.snapshots`. This prevents nested snapshots (snapshots inside of other snapshots), which can be problematic.

Here is the correct procedure to set up Snapper after the system is installed and you have rebooted into it.

1. Install dependencies
  
    ```shell
    pacman -Syu snapper snap-pac grub-btrfs inotify-tools
    ```
    
2. Create the Snapper Configuration

This is a critical step with a non-obvious workaround. The `snapper` command expects to create the `/.snapshots` directory itself, which conflicts with the subvolume we already have mounted there. Here is how to handle it correctly:

-  Umount the `@.snapshots` subvolume that we created during installation:

    ```shell
    umount /.snapshots
    ```

- Next, remove the now-empty mountpoint directory:

    ```shell
    rmdir /.snapshots
    ```

- Now, run the snapper command to create a configuration for your root filesystem (`/`). Snapper will automatically create a new `/.snapshots` directory.

    ```shell
    snapper -c root create-config /
    ```

- Delete the plain directory snapper just made:

    ```shell
    rmdir /.snapshots
    ```

- Re-mount all filesystems listed in your `/etc/fstab`, which will include our original `/.snapshots` mount:

    ```shell
    mount --mkdir -a
    ```

- Finally, verify that your `@.snapshots` subvolume is correctly mounted again:

    ```shell
    findmnt --target /.snapshots
    #--> It should show /dev/mapper/cryptroot[/@.snapshots] mounted on /.snapshots
    ```

3. Add initial configs

    ```shell
    #/etc/snapper/configs/config
    TIMELINE_MIN_AGE="1800"
    TIMELINE_LIMIT_HOURLY="5"
    TIMELINE_LIMIT_DAILY="5"
    TIMELINE_LIMIT_WEEKLY="3"
    TIMELINE_LIMIT_MONTHLY="3"
    TIMELINE_LIMIT_YEARLY="0"
    ```

4. Enable Automatic Snapshots and Cleanup

    ```shell
    systemctl enable --now snapper-timeline.timer
    systemctl enable --now snapper-boot.timer
    systemctl enable --now snapper-cleanup.timer
    systemctl enable --now grub-btrfsd.service
    ```

5. Update grub-btrfs.cfg:

    ```shell
    /etc/grub.d/41_snapshots-btrfs
    ```

6. Just for test, run it:

    ```shell
    grub-mkconfig -o /boot/grub/grub.cfg
    ```

### Notes

#### Backup LUKS Headers

It is important to make a backup of LUKS header so that you can access your data in case of emergency (if your LUKS header somehow gets damaged).

- Create a backup file:

    ```shell
    sudo cryptsetup luksHeaderBackup /dev/<luks-disk> --header-backup-file luks-header-backup-$(date -I)
    ```

- Store the backup file in a safe place, such as a USB drive. If something bad happens, you can restore the backup header:

    ```shell
    sudo cryptsetup luksHeaderRestore /dev/<luks-disk> --header-backup-file /path/to/backup_header_file
    ```

## References

- **Arch Wiki Main Guides:**
  
  - [Installation guide](https://wiki.archlinux.org/title/Installation_guide)
  - [Btrfs](https://wiki.archlinux.org/title/Btrfs)
  - [Snapper](https://wiki.archlinux.org/title/Snapper)
  - [dm-crypt/Encrypting an entire system](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system)
  - [GRUB](https://wiki.archlinux.org/title/GRUB)
  - [Zram](https://wiki.archlinux.org/title/Zram)
  - [greetd](https://wiki.archlinux.org/title/Greetd)

- **Original Inspirations:**
  
  - https://gist.github.com/mattiaslundberg/8620837