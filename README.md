Disclaimer, do not run this script on machine that you don't want to reinstall with new system.. Also, it's not fully automatized and interactive..
```
DEVICE=/dev/nvme0n1
fdisk ${DEVICE}

mkfs.fat -F 32 ${DEVICE}p1

cryptsetup luksFormat --pbkdf=pbkdf2 --pbkdf-force-iterations=2400000 ${DEVICE}p2
cryptsetup luksOpen ${DEVICE}p2 root
mkfs.btrfs /dev/mapper/root
mount /dev/mapper/root /mnt

btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@swap
btrfs subvolume create /mnt/@var_log
btrfs subvolume create /mnt/@.snapshots
btrfs filesystem mkswapfile --size 48g --uuid clear /mnt/@swap/swapfile
umount /mnt

mount --mkdir -o defaults,noatime,ssd,compress=zstd,space_cache=v2,discard=async,subvol=@ /dev/mapper/root /mnt
mount --mkdir -o defaults,noatime,ssd,compress=zstd,space_cache=v2,discard=async,subvol=@home /dev/mapper/root /mnt/home
mount --mkdir -o defaults,noatime,ssd,compress=zstd,space_cache=v2,discard=async,subvol=@swap /dev/mapper/root /mnt/swap
mount --mkdir -o defaults,noatime,ssd,compress=zstd,space_cache=v2,discard=async,subvol=@var_log /dev/mapper/root /mnt/var/log
mount --mkdir -o defaults,noatime,ssd,compress=zstd,space_cache=v2,discard=async,subvol=@.snapshots /dev/mapper/root /mnt/.snapshots
mount --mkdir -o defaults,noatime,ssd,compress=zstd,space_cache=v2,discard=async,subvolid=5 /dev/mapper/root /mnt/.btrfsroot
swapon /mnt/swap/swapfile
mount --mkdir -o defaults ${DEVICE}p1 /mnt/efi

apt install arch-install-scripts debootstrap
debootstrap questing /mnt
genfstab -U /mnt > /mnt/etc/fstab
arch-chroot /mnt
apt install ubuntu-desktop linux-generic linux-firmware grub-efi shim-signed efibootmgr initramfs-tools cryptsetup btrfs-tools sudo vim

dpkg-recofigure tzdata
dpkg-recofigure locales

UUID=`blkid -s UUID -o value ${DEVICE}p2`
echo "# ${DEVICE}p2" >> /etc/crypttab
echo "root UUID=${UUID} none luks,discard" >> /etc/crypttab

echo GRUB_ENABLE_CRYPTODISK=y >> /etc/default/grub

grub-install --target=x86_64-efi --efi-directory=/efi --boot-directory=/boot --bootloader-id=GRUB --modules="luks2 pbkdf2" --uefi --no-uefi-secure-boot /dev/mapper/root
grub-mkconfig -o /boot/grub/grub.cfg

update-initramfs -cu
```
