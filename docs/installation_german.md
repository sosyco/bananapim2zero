# Installation Archlinux 

Banana Pi BPI-M2 ZERO

## Keine Haftung
Ich beschreibe hier mein Vorgehen auf Grund meiner persoenlichen Erfahrung. Nachahmung ohne Garantie auf Funktion und keinerlei Haftung.

## Allgemeine Quellen
- http://www.banana-pi.org/downloadall.html
- https://wiki.archlinux.org/index.php/Banana_Pi#Compile_and_copy_U-Boot_bootloader


## Voraussetzungen:
- PC mit Archlinux
- SD-Karten-Leser
- Micro-SD-Karte ab 4GB
- Zugang zum Internet
- Monitor mit HDMI-Eingang 
- Netzwerkanschluss via Kabel zum Internet mit dhcp und Verbindung zum PC
- USB-Tastatur für Ersteinrichtung

## Vorbereitungen auf dem PC (Terminal-Fenster/Console)

```Bash
# Vorbereiten des Systems
sudo pacman --needed -S uboot-tools wget swig arm-none-eabi-gcc dtc git
# Finden der SD-Karte
# meisst /dev/sd... 
# oder 
# /dev/mmcblk... !!!dann im folgenden die angehaengten 1/2 durch p1/p2 austauschen!!!
sudo fdisk -l
export target_disc=/dev/...

# Löschen der SD-Karte
sudo dd if=/dev/zero of=${target_disc} bs=1M count=8

# Partitionieren der SD-Karte
sudo parted -s ${target_disc} mklabel msdos
sudo parted -a optimal -- ${target_disc} mkpart primary 2048s 150M
sudo parted -a optimal -- ${target_disc} mkpart primary 150M 100%

# Formatieren der Partitionen
sudo mkfs.ext4 -O ^metadata_csum,^64bit ${target_disc}1
sudo mkfs.ext4 -O ^metadata_csum,^64bit ${target_disc}2
export target_uuid1=$(sudo blkid ${target_disc}1 | awk '{print $2}')
export target_uuid2=$(sudo blkid ${target_disc}2 | awk '{print $2}')
export target_partuuid=$(sudo blkid ${target_disc}2 | awk '{print $4}')

# Installieren der Images
mkdir -p BananaM2Zero
cd BananaM2Zero
mkdir -p root
mkdir -p boot
sudo mount ${target_disc}1 boot
sudo mount ${target_disc}2 root
wget http://archlinuxarm.org/os/ArchLinuxARM-armv7-latest.tar.gz
sudo bsdtar -xpf ArchLinuxARM-armv7-latest.tar.gz -C root
sudo mv root/boot/* boot/.
echo "${target_uuid2}       /               ext4            rw,relatime     0 1" | sudo tee -a root/etc/fstab
echo "${target_uuid1}       /boot           ext4            rw,relatime     0 2" | sudo tee -a root/etc/fstab

# Bootsystem installieren
cat <<EOF > boot.cmd.sd-card
part uuid \${devtype} \${devnum}:\${bootpart} uuid
setenv bootargs console=\${console} root=${target_partuuid} rw rootwait

if load \${devtype} \${devnum}:\${bootpart} \${kernel_addr_r} zImage; then
  if load \${devtype} \${devnum}:\${bootpart} \${fdt_addr_r} dtbs/\${fdtfile}; then
    if load \${devtype} \${devnum}:\${bootpart} \${ramdisk_addr_r} initramfs-linux.img; then
      bootz \${kernel_addr_r} \${ramdisk_addr_r}:\${filesize} \${fdt_addr_r};
    else
      bootz \${kernel_addr_r} - \${fdt_addr_r};
    fi;
  fi;
fi

if load \${devtype} \${devnum}:\${bootpart} 0x48000000 uImage; then
  if load \${devtype} \${devnum}:\${bootpart} 0x43000000 script.bin; then
    setenv bootm_boot_mode sec;
    bootm 0x48000000;
  fi;
fi
EOF

sudo mkimage -A arm -O linux -T script -C none -a 0 -e 0 -n "BananaM2Zero boot script" -d boot.cmd.sd-card boot/boot.scr
sudo umount boot
sudo umount root

# uboot installieren
git clone git://git.denx.de/u-boot.git
cd u-boot
make -j4 ARCH=arm CROSS_COMPILE=arm-none-eabi- bananapi_m2_zero_defconfig
make -j4 ARCH=arm CROSS_COMPILE=arm-none-eabi-
sudo dd if=u-boot-sunxi-with-spl.bin of=${target_disc} bs=1024 seek=8 && sync

```

## Booten des BananaM2Zero
- SSD in den BananaPi stecken
- HDMI, Netzwerk, Tastatur anschliessen
- USB-Stromversorgung verbinden

## Nach dem Booten
Mit user root Passwort root einloggen.

Danach folgende Befehle auf der Console ausführen:
```bash
loadkeys de-latin1
passwd root
pacman-key --init
pacman-key --populate archlinuxarm
pacman -ySu sudo uboot-tools dialog wpa_supplicant
useradd -m -g users -G wheel -s /bin/bash $username$
passwd $username$
userdel alarm
systemctl enable sshd.service
# ipadresse fuer eth0 merken
ip a
reboot
```

Danach über ssh einloggen
```bash
ssh $user_name$@$ipadresse$
sudo su
loadkeys de-latin1
echo "de_DE.UTF-8 UTF-8" >> /etc/locale.gen
echo "de_DE ISO-8859-1" >> /etc/locale.gen
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
echo "en_US ISO-8859-1" >> /etc/locale.gen
locale-gen
echo "LANG=de_DE.UTF-8" >> /etc/locale.conf
echo "LANGUAGE=de_DE" >> /etc/locale.conf
echo "LC_ALL=C" >> /etc/locale.conf
echo "KEYMAP=de-latin1" >> /etc/vconsole.conf
echo "%wheel ALL=(ALL) ALL" >> /etc/sudoers
exit
reboot
```

## Festplatte/SSD als rootfs
kein SATA-Anschluss 

## Zugriff auf WLAN
konnte das Wlan-Device nicht aktivieren
```bash
sudo wifi-menu
```

## Installation von python
```bash
sudo pacman --needed -S python python-pip
```

### Installation GPIO-Bibliotheken
Quelle: https://github.com/LeMaker/RPi.GPIO_BP
```bash
sudo pacman --needed -S gpio-utils python-periphery wiringx-git git base-devel
git clone https://github.com/LeMaker/RPi.GPIO_BP -b bananapi
python setup.py install
sudo python setup.py install

# test: changes #17 between high and low every second
cat <<EOF > test.py
import RPi.GPIO as GPIO
from time import sleep
GPIO.setmode(GPIO.BCM)
GPIO.setup(17, GPIO.OUT)
while True:
    GPIO.output(17, GPIO.HIGH)
    sleep(1)
    GPIO.output(17, GPIO.LOW)
    sleep(1)
EOF

sudo python test.py
```

# ToDos
- Freigabe von /dev/mem (gpio) fuer User
