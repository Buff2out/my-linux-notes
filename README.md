# Здесь будут заметки ко всему

- [Создание загрузочной флешки (Live CD)](#Создание-загрузочной-флешки-Live-CD)

#### Создание загрузочной флешки Live CD  

if - откуда, of - куда. Можно таким образом создавать бэкапы, утилита мощная  
```bash
sudo dd if=/home/whoami/Downloads/nixos-minimal-24.11.717822.0c0bf9c05738-x86_64-linux.iso of=/dev/sda bs=4M status=progress oflag=sync
```

Утилитки если вдруг всё сломалось 
```
lsblk
fdisk -l
cfdisk
``` 

Пересборка grub:   
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
mkinitcpio -P
grub-mkconfig -o /boot/grub/grub.cfg
```

Глянуть id разделов
```bash
blkid /dev/nvme1n1p1
```

Troubleshooting  
```bash
fsck.fat -v /dev/nvme1n1p1  # Проверка FAT32 (EFI)
btrfstune -U random /dev/nvme1n1p2
```

Зайти в смонтированные разделы

предварительно смонтировав  
```bash
mount /dev/nvme1n1p2 /mnt           # Корень
mount /dev/nvme1n1p1 /mnt/boot/efi  # EFI

arch-chroot /mnt
```


При установке форматирование будет примерно такое  
```bash
mkfs.fat -F 32 -n BOOT /dev/sda1
mkswap -L swap /dev/sda2

sudo mkfs.btrfs -L NIXOS -f /dev/sda3 # for btrfs
```

для ext4

```bash
mkfs.ext4 -L nixos /dev/sda3
```

![alt text](./misc/image.png)

subvolumes for btrfs:  
```
mount -t btrfs /dev/sda3 /mnt
cd /mnt
btrfs subvolume create @
btrfs subvolume create @home
btrfs subvolume create @nix
btrfs subvolume create @snapshots
cd
umount /mnt
```

Умное монтирование btrfs

```
mount -o compress=zstd,subvol=@ /dev/sda3 /mnt
mkdir -p /mnt/{boot,home,nix,.snapshots}
mount -o compress=zstd,subvol=@home /dev/sda3 /mnt/home
mount -o compress=zstd,subvol=@nix /dev/sda3 /mnt/nix
mount -o compress=zstd,subvol=@snapshots /dev/sda3 /mnt/.snapshots


mount /dev/sda1 /mnt/boot
```

Простое монтирование через ext4

```bash
mount /dev/sda3 /mnt
mount --mkdir /dev/sda1 /mnt/boot
swapon /dev/sda2
mount -t efivarfs efivarfs /sys/firmware/efi/efivars
```

NixOs
```
nixos-generate-config --root /mnt
nano /mnt/etc/nixos/configuration.nix
nixos-install
```

Ещё недостающие конфиги, но это уже завтра...

```nix
{ config, pkgs, ... }: {
  # Включить Wayland и Hyprland
  programs.hyprland = {
    enable = true;
    xwayland.enable = true;  # Для совместимости с X11-приложениями
  };

  # Графические драйверы (пример для Intel/NVIDIA)
  hardware.opengl.enable = true;
  services.xserver.videoDrivers = [ "nvidia" ];  # Для NVIDIA
  # Или для AMD:
  # hardware.amdgpu.loadInInitrd = true;

  # Пользователь и сессия
  users.users.<ваш_логин> = {
    isNormalUser = true;
    extraGroups = [ "wheel" "video" "render" ];
  };

  # Автовход в TTY (если используете display manager)
  services.displayManager.autoLogin = {
    enable = true;
    user = "<ваш_логин>";
  };

  environment.systemPackages = with pkgs; [
    hyprland
    waybar
    swaylock-effects
    wofi
    kitty  # Терминал
  ];
}
```

![alt text](./misc/image4.png)

![alt text](./misc/image-1.png)

![alt text](./misc/image-2.png)

![alt text](./misc/image-3.png)

вот такая локаль  
```
ru_RU.UTF-8/UTF-8
```
отсюда взято
https://sourceware.org/git/?p=glibc.git;a=blob;f=localedata/SUPPORTED


swap-file (на Arch)

```bash
swapon --show
df -Th /swapfile
sudo rm /swapfile
sudo touch /swapfile
sudo chattr +C /swapfile
sudo fallocate -l 16G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
swapon --show
free -h
code src/me/guideall/
```