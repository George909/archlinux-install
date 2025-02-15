# Установка archlinux с полным шифрование диска и загрузчиком на флешке

## Перед началом усановки

Разметить флешшку: 1-ый раздел с образом, 2-ой раздел со скриптами, которые понадобятся при установке.

## Разметка диска и файловая система

Разметка флешки с загрузчиком.

```console
  Part. #     Size        Partition Type            Partition Name

----------------------------------------------------------------
            1007.0 KiB  free space
   1         100.0 MiB  EFI System                ESP
   2         512.0 MiB  Linux filesystem          Boot
   3          57.1 GiB  Linux filesystem          Storage

```

Форматируем EFI System в FAT32.

```console
mkfs.fat -F32 /dev/sdb1
```

Находим лучший алгоритм шифрования для нашего железа

```console
cryptsetup benchmark
# Tests are approximate using memory only (no storage IO).
PBKDF2-sha1      1460412 iterations per second for 256-bit key
PBKDF2-sha256    5675648 iterations per second for 256-bit key
PBKDF2-sha512    2427259 iterations per second for 256-bit key
PBKDF2-ripemd160 1084359 iterations per second for 256-bit key
PBKDF2-whirlpool  836185 iterations per second for 256-bit key
argon2i      12 iterations, 1048576 memory, 4 parallel threads (CPUs) for 256-bit key (requested 2000 ms time)
argon2id     12 iterations, 1048576 memory, 4 parallel threads (CPUs) for 256-bit key (requested 2000 ms time)
#     Algorithm |       Key |      Encryption |      Decryption
        aes-cbc        128b      1308.2 MiB/s      3809.9 MiB/s
    serpent-cbc        128b       127.4 MiB/s       900.3 MiB/s
    twofish-cbc        128b       254.8 MiB/s       553.9 MiB/s
        aes-cbc        256b       987.8 MiB/s      3512.9 MiB/s
    serpent-cbc        256b       133.3 MiB/s       901.3 MiB/s
    twofish-cbc        256b       264.1 MiB/s       556.4 MiB/s
        aes-xts        256b      5207.9 MiB/s      5230.4 MiB/s
    serpent-xts        256b       789.4 MiB/s       795.9 MiB/s
    twofish-xts        256b       499.1 MiB/s       516.1 MiB/s
        aes-xts        512b      4820.5 MiB/s      4798.6 MiB/s
    serpent-xts        512b       808.1 MiB/s       804.5 MiB/s
    twofish-xts        512b       508.4 MiB/s       520.2 MiB/s
```

Шифруем и форматируем раздел на котором будет grub (важно явно указывать luks1, т.к. grub не поддерживает luks2.

```console
cryptsetup luksFormat --type luks1 -c aes-xts -s 512 /dev/sdb2
cryptsetup open --type luks1 /dev/sdb2 cryptboot
mkfs.ext2 /dev/mapper/cryptboot
```

Форматируем последний раздел в FAT32, чтобы использовать как обычную флешку

```console
mkfs.fat -F32 /dev/sdb3
```

Монтируем расшифрованную флешку, шифруем диск под систему в luks1, хэдера и ключ сохраняем на флешку в бут раздел.

```console
mount /dev/mapper/cryptboot /mnt
truncate -s 16M /mnt/header.img
dd if=/dev/urandom of=/mnt/key.img bs=1M count1 iflag=fullblock
cryptsetup --key-file /mnt/key.img --keyfile-offset 1024 --keyfile-size=32 luksFormat --type luks1 /dev/sda --header /mnt/header.img
cryptsetup open --type luks1 --header /mnt/header.img --key-file /mnt/key.img --keyfile-offset 1024 --keyfile-size=32 /dev/sda cryptroot
unount /mnt
```

Создаем виртуальные разделы (Можно не создавать swap и тогда lvm вообще не нужен, но с lvm универсальнее), и форматируем:

```console
pvcreate /dev/mapper/cryptroot
vgcreate System /dev/mapper/cryptroot
lvcreate -L 16G System -n swap
lvcreate -l 100%FREE System -n root
mkswap /dev/mapper/System-swap
swapon -d /dev/mapper/System-swap
```

### Настраиваем файловую систему на системном диске

Форматирем диск в btrfs и создаем все необходимые субтомы

```console
mkfs.btrfs /dev/mapper/System-root
mount /dev/mapper/System-root /mnt
cd /mnt
btrfs subvolume create @
btrfs subvolume create @home
btrfs subvolume create @docker
btrfs subvolume create @snapshots
btrfs subvolume create @var-cache
btrfs subvolume create @var-log
btrfs subvolume create @var-tmp
btrfs subvolume create @vms
btrfs subvolume create @vms-snapshots
```

Монтируем btrfs

```console
umount /mnt
mount -o ssd,noatime,compress=zstd,discard=async,autodefrag,space_cache=v2,subvol=@ /dev/nvme0n1p1 /mnt
mkdir -p /mnt/{home,var/lib/docker,.snapshots,var/cache/pacman/pkg,var/log,var/tmp,var/lib/libvirt/images,var/lib/libvirt/snapshots}
mount -o ssd,noatime,compress=zstd,discard=async,autodefrag,space_cache=v2,subvol=@home /dev/nvme0n1p1 /mnt/home
mount -o ssd,noatime,compress=zstd,discard=async,autodefrag,space_cache=v2,subvol=@docker /dev/nvme0n1p1 /mnt/var/lib/docker
mount -o ssd,noatime,compress=zstd,discard=async,autodefrag,space_cache=v2,subvol=@snapshots /dev/nvme0n1p1 /mnt/.snapshots
mount -o ssd,noatime,compress=zstd,discard=async,autodefrag,space_cache=v2,subvol=@var-cache /dev/nvme0n1p1 /mnt/var/cache/pacman/pkg
mount -o ssd,noatime,compress=zstd,discard=async,autodefrag,space_cache=v2,subvol=@var-log /dev/nvme0n1p1 /mnt/var/log
mount -o ssd,noatime,compress=zstd,discard=async,autodefrag,space_cache=v2,subvol=@var-tmp /dev/nvme0n1p1 /mnt/var/tmp
mount -o ssd,noatime,compress=zstd,discard=async,autodefrag,space_cache=v2,subvol=@vms /dev/nvme0n1p1 /mnt/var/lib/libvirt/images
mount -o ssd,noatime,compress=zstd,discard=async,autodefrag,space_cache=v2,subvol=@vms-snapshots /dev/nvme0n1p1 /mnt/var/lib/libvirt/snapshots
```

Создаем бут раздел в файловой системе ОС и монтируем туда шифрованный бут на флешке

```console
mkdir /mnt/boot
mount /dev/mapper/cryptboot /mnt/boot
mkdir /mnt/boot/efi
mount /dev/sdb1 /mnt/boot/efi
```

## Установка арча

Комманда для установки арча с xfce и всем необходимым для звука и подключения к интернету, для видео устанавливается xf86-video-intel или xf86-video-amdgpu в зависимости от процессора. Установка драйверов нвидиа - отдельная тема.

```console
pacstrap /mnt base linux linux-firmware btrfs-progs efibootmgr gurb-efi-x86_64 intel-ucode vim lvm2 iwd iwctl dhcpcd ext2 xfce4 xfce4-goodies pipewire pipewire-pulse pipewire-alsa pipewire-jack wireplumber lightdm lightdm-gtk-greeter networkmanager network-manager-applet xf86-video-intel
```

Если DE не нужно то устанавливать этой коммандой

```console
pacstrap /mnt base linux linux-firmware btrfs-progs efibootmgr gurb-efi-x86_64 intel-ucode vim lvm2 iwd iwctl dhcpcd ext2
```

После установки создаем fstab

```console
genfstab -U /mnt >> /mnt/etc/fstab
```

Чрутаемся

```console
arch-chroot /mnt
```

Создаем хук для расшифровки системного раздела при загрузке. Нужно смонтировать второй раздел установочной флешки и скопировать от туда скрипты.

```console
mount /dev/sdc2 /mnt
cp /mnt/menc /etc/initcpio/hooks/menc
```

Сам скрипт выглядит так

```bash
#!/usr/bin/ash

run_hook() {
    modprobe -a -q dm-crypt >/dev/null 2>&1
    modprobe loop
    [ "${quiet}" = "y" ] && CSQUIET=">/dev/null"

    while [ ! -L '/dev/disk/by-id/usbdrive-part2' ]; do
     echo 'Waiting for USB'
     sleep 1
    done

    cryptsetup open /dev/disk/by-id/usbdrive-part2 cryptboot
    mount --mkdir /dev/mapper/cryptboot /mnt
    cryptsetup open /mnt/key.img lukskey
    cryptsetup --header /mnt/header.img --key-file=/dev/mapper/lukskey --keyfile-offset=''offset'' --keyfile-size=''size'' open /dev/disk/by-id/harddrive enc
    cryptsetup close lukskey
    umount /mnt
}
```

usbdrive-part2 - id загрузочного раздела на флешке **(!!! в двух строчкаx !!!)**, harddrive - id системного раздела. Получим этим id и запишем в конец скрипта для удобства редактирования. После подстановки не забыть удалить эти строки

```console
blkid /dev/sdb2 >> /etc/initcpio/hooks/menc
ls -l /dev/disk/by-id | grep sda >> /etc/initcpio/hooks/menc
```

Скопируем заклинание и удалим от туда help

```console
cp /usr/lib/initcpio/install/encrypt /etc/initcpio/install/customencrypthook
```

Поправим файл /etc/mkinitcpio.conf добавив туда наш хук и lvm2

```console
MODULES=(ext2)
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block customencrypthook lvm2 filesystems fsck)
```

Теперь можно пересоздать initramdisk

```console
mkinitcpio -p linux
```

Копируем файл grub с установочной флешки в /etc/default/grub и вставляем туда свой id диска с системой (по аналогии с созданием хука)

```console
cp /mnt/grub /etc/default/grub
```

```console
mkdir /boot/gurb
grub-mkconfig -o /boot/grub/grub.cfg
grub-install --target=x86_64-efi --efi-directory=/boo/efi --bootloader-id="grub"
```

## Настройка системы

Проведем базовую настройку системы

Установим таймзону

```console
ln -sf /usr/share/zoneinfo/Europe/Samara /etc/localtime
hwclock --systohc --utc
```

Раскомментировать строчку в /etc/locale.gen en_US.UTF-8 UTF-8
Установить язык для системы

```console
locale-gen
echo LANGUAGE=en_US >> /etc/locale.conf
echo LANG=en_US.UTF-8 >> /etc/locale.conf
```

Создаем пользователя и назначаем ему пароль

```consile
useradd -m -g users -G wheel,storage,power -s /bin/bash your_new_user_name
passwd your_new_user_name
```

Устанавливаем sudo

```console
pacman -S sudo
EDITOR=vim visudo
  uncomment the following line
  %wheel ALL=(ALL) ALL
```

Выходим из системы, umount диска, перезагружаемся и молимся
