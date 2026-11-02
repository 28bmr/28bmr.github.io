# Arch Linux Installation Guide

This document outlines the steps I took to manually install Arch Linux in VMware Workstation. I used the official [Arch Linux Installation Guide](https://wiki.archlinux.org/title/Installation_guide). I also installed KDE Plasma as my DE and configured a few additional features for the assignment.

## 1. Pre-installation

### 1.1 Download ISO

Download ISO from [archlinux.org/download](https://archlinux.org/download).

To verify the ISO, the hash and signature of the file can be checked against the published ones with these commands:

- **SHA-256 hash:** `CertUtil -hashfile .\archlinux-2026.10.01-x86_64.iso SHA256`
- **GPG signature:** `gpg --verify archlinux-2026.10.01-x86_64.iso.sig archlinux-2026.10.01-x86_64.iso`

### 1.2 Create the VM

1. Select **Typical**.
2. Select the Arch ISO.
3. Select **Linux**.
4. Select **Other Linux 6.x kernel 64-bit** (since Arch is not listed and this is the closest).
5. Set max disk size to **50 GB** and select **Split virtual disk into multiple files** (split is easier to move).
6. Set memory to **6144 MB** (to allocate 6 GiB of RAM).
7. Set processors to **2 processors × 2 cores** (KDE Plasma will use a good amount of CPU).
8. Before booting go to **VM Settings > Options > Advanced > Firmware Type > UEFI**.
   - Faster boot times than legacy BIOS (and this ensures we always do the same thing).
9. Make sure **Secure Boot** is disabled.

### 1.3 Start the VM

1. Start the VM in VMware
2. Choose **Arch Linux install medium (x86_64, UEFI)**.

### 1.4 Once the terminal has loaded

1. Run `setfont ter-124` (for much better terminal font visibility, default is tiny).

2. Run `cat /sys/firmware/efi/fw_platform_size` to verify boot mode.
   - Should return `64`.

3. Connect to the internet:
   - Run `ip link`.
     - Should show `ens33` as an active network interface.
   - Run `ping ping.archlinux.org`.
     - **Ctrl+C** to stop the test (time was roughly 200 ms on first test).

1. Verify time by running `timedatectl` (displays system time, time zone, and sync status).

### 1.5 Disk Partitioning

1. Check the main disk name using `lsblk` (lists available disks and partitions).
   - Probably named `sda`.

2. **Partition layout**

   UEFI needs an EFI System Partition, which can be added using these commands:

   3. `fdisk /dev/sda`
   4. `g` (create a GPT partition table)
   5. `n` (create a partition)
   6. **DEFAULT** (should be partition 1)
   7. **DEFAULT**
   8. `+1G`
   9. `t`
   10. `1` (if asked, this should default to 1)
   11. `uefi`
   **NOTE:** The root partition should use the rest of the space (not doing a swap partition because we already have 6 GiB of RAM).
   12. `n` (create a partition)
   13. **DEFAULT** (should be partition 2)
   14. **DEFAULT**
   15. **DEFAULT** (should be the rest of the disk, approx 49 GB)

16. The command `p` in `fdisk` should return something like this:

```
Device      Start     End        Sectors     Size   Type
/dev/sda1   2048      2099199    2097152     1G     EFI System
/dev/sda2   2099200   104855551  102756352   49G    Linux filesystem
```

17. Use the command `w` to write the changes.

### 1.6 Formatting

1. Format the root partition with (formats with ext4):
```bash
mkfs.ext4 /dev/sda2
```
1. Format the EFI partition as FAT32 with (formats with FAT32):
```bash
mkfs.fat -F 32 /dev/sda1
```

### 1.7 Mounting

1. Mount the root volume with:
```bash
mount /dev/sda2 /mnt
```
1. Mount the EFI partition with (also creates the directory necessary):
```bash
mount --mkdir /dev/sda1 /mnt/boot
```

### 1.8 Verify

Run (displays filesystem types and mounting locations):

```bash
lsblk -f
```

(Should show sda1 and sda2 under sda, with their correct formatting and mount locations)

---

## 2. Installation

### 2.1 Install Essential Packages

For the essential packages (base Arch system and Linux kernel) , use (no need for `linux-firmware` since it's a VM):

```bash
pacstrap -K /mnt base linux
```

---

## 3. Configuration

### 3.1 Generate the Filesystem Table

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Generates the filesystem table using UUIDs.

### 3.2 Enter the Installed System

```bash
arch-chroot -S /mnt
```

Enters the Arch Linux system just installed using systemd mode.

### 3.3 Time

1. Set the timezone:
   ```bash
   ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime
   ```
2. Update the hardware clock:
   ```bash
   hwclock --systohc
   ```

### 3.4 Localization

**NOTE:** Need Nano or another text editor to edit the following file. Nano can be installed with:

```bash
pacman -S nano
```

1. Edit `/etc/locale.gen` with Nano:
   ```bash
   nano /etc/locale.gen
   ```
   - Find `#en_US.UTF-8 UTF-8`.
   - Remove the `#`.
   - Save and exit.

2. Generate the locale with:
   ```bash
   locale-gen
   ```

3. Create the config file and set `LANG`:
   ```bash
   echo "LANG=en_US.UTF-8" > /etc/locale.conf
   ```

### 3.5 Networking

1. Set hostname with:
   ```bash
   echo "arch-vm" > /etc/hostname
   ```

2. Install and enable network management:
   ```bash
   pacman -S networkmanager
   systemctl enable NetworkManager
   ```

### 3.6 Initramfs

Shouldn't have to do anything unless using LVM, system encryption, or RAID.

### 3.7 Root Password

Set the root password with:

```bash
passwd
```

### 3.8 Boot Loader

Reference: [Arch Wiki – systemd-boot](https://wiki.archlinux.org/title/Systemd-boot)

1. Install with:
   ```bash
   bootctl install
   ```

2. Configure with:
   ```bash
   nano /boot/loader/loader.conf
   ```

   Enter:
   ```conf
   default arch.conf
   timeout 4
   console-mode auto
   editor no
   ```

   Save and exit.

   **NOTE:** Configuration settings are mostly copied from the Wiki example except for `console-mode`, which is set to `auto` since the VM window may work better with this.

3. **NOTE:** I decided to go with systemd-boot because it looked like the simplest setup, however, immediately upon installing, I got a warning. In the future, maybe try another boot loader.

   `Mount point '/boot' which backs the random seed file is world accessible, which is a security hole! Random seed file '/boot/loader/.#bootctlrandom-seed6a894afd44d9d45b' is world accessible, which is a security hole!`

5. **Add the loader**

   Find the UUID of the root partition:
   ```bash
   blkid -s UUID -o value /dev/sda2
   ```

   Use the output for `ACTUAL_UUID` in the following config.

   Open:
   ```bash
   nano /boot/loader/entries/arch.conf
   ```

   Enter:
   ```conf
   title Arch Linux
   linux /vmlinuz-linux
   initrd /initramfs-linux.img
   options root=UUID=ACTUAL_UUID rw
   ```

---

## 4. Reboot

1. Type `exit` to exit `arch-chroot`.
2. Run `umount -R /mnt` to check for any busy partitions.
3. Run `poweroff` and toggle off the installation ISO in VMware (installation media can stop the reboot).
4. Power the VM back on.

---

## 5. Post-Installation

1. Log in with root credentials.

2. Ping to test internet connection:
   ```bash
   ping -c 4 archlinux.org
   ```

   **NOTE:** The internet should be tested every time you connect. I paused the project here and came back, and the internet was no longer working. Upon further investigation, this was VMware's NAT Service that was failing, not the actual VM connection. Restart VMware NAT Service and DHCP Service (in services.msc) before continuing if this happens.

---

## 6. Create User Accounts

### 6.1 Install Sudo and Zsh

Install sudo and Zsh (so we can make sudo users and have the different shell):

```bash
pacman -S sudo zsh
```

### 6.2 Add Accounts

Add the accounts to the `wheel` group since they will be sudo users, also make Zsh the default shell.

```bash
useradd -m -G wheel -s /bin/zsh brody
useradd -m -G wheel -s /bin/zsh codi
```

- `-m` creates the user's home directory.
- `-G wheel` adds the user to the `wheel` group.
- `-s /bin/zsh` makes Zsh the default shell.

### 6.3 Set Passwords

```bash
passwd brody
passwd codi
```

### 6.4 Enable Sudo for Wheel Accounts

1. Open the sudoers config with nano:
   ```bash
   EDITOR=nano visudo
   ```

2. Find:
   ```conf
   # %wheel ALL=(ALL:ALL) ALL
   ```

3. Remove the `#`:
   ```conf
   %wheel ALL=(ALL:ALL) ALL
   ```

### 6.5 Check the Accounts

Run (shows group memberships):

```bash
id brody
id codi
```

Should both have `wheel` listed.

---

## 7. Install KDE Plasma

Reference: [Arch Wiki – KDE](https://wiki.archlinux.org/title/KDE)

### 7.1 Install Plasma

```bash
pacman -S plasma
```

Stick with all defaults for now.

### 7.2 Install Essential Applications

Install the essentials before going to the DE (terminal, file manager, browser):

```bash
pacman -S konsole dolphin firefox
```

### 7.3 Install and Enable Plasma Login Manager

Install Plasma Login Manager separately (not installed by default by plasma for some reason):

```bash
pacman -S plasma-login-manager
```

**NOTE:** There are several options for login managers. This is the "default" KDE option I chose, even though it wasn't installed automatically.

Enable the login manager:

```bash
systemctl enable plasmalogin
```

### 7.4 Boot into the GUI

Set the system to boot into the GUI DE:

```bash
systemctl set-default graphical.target
```

Then reboot:

```bash
reboot
```

### 7.5 Troubleshooting

**Issue 1: KDE Plasma Black Screen**

On initial reboot, the desktop never launched after logging in. However, the search menu in KDE Plasma was working (`Alt+F2`). To fix this, I opened konsole and checked for the necessary packages. Somehow, `plasma-desktop` was missing, so I used sudo to install it:

```bash
sudo pacman -S plasma-desktop
```

**Issue 2: User Password**

During the first login to my account (`brody`), the password was not working (because I set it with num-lock off, don't do that). I reset it by running the following command while logged into the `codi` account:

```bash
sudo passwd brody
```

---

## 8. Color Coding & Aliases

### 8.1 Open Zsh Config file

```bash
nano ~/.zshrc
```

### 8.2 Enable Colors

Add these lines:

```config
autoload -Uz colors
colors
```

### 8.3 Color the Terminal Prompt

This makes the prompt show `username@hostname:currentDirectory%` colored.

```config
PROMPT='%F{cyan}%n@%m%f:%F{green}%~%f%# '
```

### 8.4 Additional Color Coding

While in `.zshrc`, we can add more automatic color coding.

**NOTE:** Not all commands have `--color=auto` as an option.

```config
alias ls='ls --color=auto'
alias grep='grep --color=auto'
alias egrep='egrep --color=auto'
alias fgrep='fgrep --color=auto'
alias dir='dir --color=auto'
alias ip='ip --color=auto'
```
### 8.5 Aliases

We can also add aliases the same way:
```config
alias ll='ls -lah --color=auto' # Long listing format
alias l.='ls -d .* --color=auto' # Show hidden files
alias mkdir='mkdir -pb' # Always make parent directories
alias h='history' # History shortcut 
alias path='echo -e ${PATH//:/\\n}' # Path shortcut 
alias ping='ping -c 4' # Ping 4 packets instead of continuous
alias fastping='ping -c 100 -s.2' # No 1 sec intervals (100 packets)
alias ..='cd ..' # Go up one directory
alias ...='cd ../..' # Go up two directories
alias update='sudo pacman -Sye' # Updates installed packages
alias install='sudo pacman -S' # Install packages
alias moaa='nano ~/.zshrc' # Mother of all aliases (the alias for editing aliases)
alias zshrc='nano ~/.zshrc' # Does the same as moaa (easy to remember) 
```

### 8.6 Reload the Configuration

```bash
source ~/.zshrc
```

Reloads the configuration so the changes take effect.

### 8.7 Copying Config

**NOTE:** This only edits the configuration file for the current user. However, you can copy the file using these commands (make sure to reset the ownership to the correct user):

```bash
sudo cp /home/brody/.zshrc /home/codi/.zshrc
sudo chown codi:codi /home/codi/.zshrc
```

---

## 9. Install SSH

### 9.1 Install OpenSSH

```bash
sudo pacman -S openssh
```

**NOTE:** Could technically use the `install` alias instead.

### 9.2 Enable SSH

```bash
sudo systemctl enable --now sshd
```

Enables the SSH server at boot and starts it now.

### 9.3 Find the VM's IP Address

```bash
ip -br addr
```

### 9.4 Test SSH Connection

Connect from PowerShell on host Windows machine:

```powershell
ssh brody@XXX.XXX.XXX.XXX
```

Replace `XXX.XXX.XXX.XXX` with the VM's IP address.

**NOTE:** Connecting from a machine that isn't the actual host windows machine requires additional configuration in VMware.
