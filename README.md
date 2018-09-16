# Linux in RAM: debirf way

Do you want to have absolute disk speed nowadays, don’t you? Let's revise how it could be done with versioning and automation in 2018.

> От модератора: нам в Песочницу пришла статья на английском языке. Мы её прочитали и и в качестве пятничного эксперимента решили опубликовать. Не судите строго, всем peace and happy Friday! Let's come together! Короче, фром зе боттом оф ауа хартс.

<cut />

## Changelog:

- The language of the article is corrected in English
- sample repository introduced at [github](https://github.com/egeneralov/debirf-example.git)
- Added test params
- added asciicast

## You must know before running this tutorial:


- linux essential
- the main difference between bash and sh (debirf written on sh, so I recommend to follow the style)
- how to format usb key (in any way)
- what drivers are needed for your hardware (or you can debug it via dmesg|lspci)
- how to gain your task purposes via the scripting

### you can replace:

- usb letter from `/dev/sdb` to any else (`/dev/sdd`)
- working directory from `/root/Projects/debirf/` to your choice (`/home/username/Documents/debirf`)
- mountpoint from `/media/root/8B46-1189` to `/media/username/myflashdrive`

## Steps to preparing

They are (optional) enable non-free components for basic installation. In addition, I think, you will need the non-free repository.

- open line number 107 of file `/usr/bin/debirf` like `nano +107 $(which debirf)`
- find function `create_debootstrap`
- find line like `local OPTS="`
- add `--components main,contrib,non-free` in brackets

### Up to start:

Let's imagine, our flash drive:

- must to be fast, so it is recommended to use 8+ class, or booting will take enough time
- /dev/sdb
- formated
- mounted at `/media/root/8B46-1189`.
- our working directory `/root/Projects/debirf/`

### Install debirf

```bash
apt-get install -yq debirf mtools genisoimage
```

- mtools needed for create iso via debirf (not working, but needed)
- genisoimage needed for create real working iso (optional)


### Prepare debirf working directory

```bash
mkdir -p /root/Projects/debirf
tar xzf /usr/share/doc/debirf/example-profiles/rescue.tgz -C /root/Projects/debirf
cd /root/Projects/debirf/rescue
```

### And configure /root/Projects/debirf/rescue/debirf.conf

```bash
DEBIRF_LABEL="debirf-rescue"
DEBIRF_SUITE=stretch
DEBIRF_DISTRO=debian
DEBIRF_MIRROR=http://ftp.ru.debian.org/debian/
```

## Create LiR

- Run `debirf make .` and go away. It need many time, at minimal 15 minutes on top hardware.
- Run `debirf makeiso .` for create not working iso (needed for grub.cfg file)


## Test it

- Install qemu
  - for linux: `apt-get install -yq qemu`
  - for macos: `brew install qemu`
- decide which resources will be allocated for VM
  - `-smp 1` 1 real kernel
  - `-m 1G` 1G memory
- additional
  - `-nographic` will launch VM in current terminal window
  - `--enable-kvm` enables hardware accelation
  - `-kernel vmlinuz-*` permit directly pass kernel
  - `-initrd *.cgz` direct access to .cgz file with initramfs
  - `-append` allows bypass kernel params, here are the parameters for running without a graphical shell

The command to start the virtual machine:

    qemu-system-x86_64 --enable-kvm -kernel vmlinuz-* -initrd *.cgz -append "console=tty0 console=ttyS0,115200n8" -m 1G -smp 1 -net nic,vlan=0 -net user -nographic

## Test sample

[![asciicast](https://asciinema.org/a/YvChsRn942rrbEOaTVR1z1tXX.png)](https://asciinema.org/a/YvChsRn942rrbEOaTVR1z1tXX)

### Install grub to flash drive and copy LiR on it

I recommend you use bios legacy boot and package grub-pc. Not tested with UEFI, but must work. Next lines will be do:

- create mount point (on GUI-powered systems enabled auto-mount it isn't needed)
- mount usb key to mount point (on GUI-powered systems enabled auto-mount it isn't needed)
- install grub
- copy grub file
- copy initramfs (system)
- copy vmlinuz (kernel)
- unmount usb key
- remove mount point

```bash
mkdir -p /media/root/8B46-1189
mount /dev/sdb1 /media/root/8B46-1189
grub-install --boot-directory=/media/root/8B46-1189/boot /dev/sdb
cp /root/Projects/debirf/rescue/iso/boot/grub/grub.cfg /media/root/8B46-1189/boot/grub/
cp /root/Projects/debirf/rescue/*.cgz /media/root/8B46-1189
cp /root/Projects/debirf/rescue/vmlinuz-* /media/root/8B46-1189
umount /media/root/8B46-1189
rm -rf /media/root/8B46-1189
```


### Create bootable iso (optional)

- download isolinux.bin
- create isolinux config file
- create iso

```bash
mkdir -p rescue/iso/isolinux/
wget -O rescue/iso/isolinux/isolinux.bin 'http://mirror.yandex.ru/centos/7/os/x86_64/isolinux/isolinux.bin'

cat << EOF > rescue/iso/isolinux/isolinux.cfg
TIMEOUT 5
DEFAULT lir

LABEL lir
  LINUX /vmlinuz-4.9.0-7-amd64
  INITRD /debirf-rescue_stretch_4.9.0-7-amd64.cgz
EOF

genisoimage -b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -l -input-charset default -V LiR -A "Linux in RAM" -o rescue/rescue.iso rescue/iso/
```


## Check result

- Install QEMU hypervisor `apt-get install -yq qemu`
- run `qemu-system-x86_64 -enable-kvm -m 512 -hda /dev/sdb`
- if previous command fails, remove `-enable-kvm`
- VM will be started, booted from usb key
- you must see two menu items, difference in end: console and serial. Select first entry.
- in ~minute you will see many lines. after it ends - press enter to see welcome message
- login: root, no password

## Customize it: Create custom debirf module

Module - executable sh script for running during LiR creating

- create module file
- the order of file’s names is important. In file `network` the resolving file will be changed to the system-resolved, and you can’t work with the network;
- lines from 1 to 3 must be present, line 3 must present file name
- if you want install package - use construction `#DEBIRF_PACKAGE>+`
- if you want work with rootfs - use `$DEBIRF_ROOT`
- if you want to run command in fakeroot - use `debirf_exec`

### Sample:

```bash
cat <<< EOF > rescue/modules/mi
#!/bin/sh -e

# debirf module: mi
# prepare to run on mi notebook
#
# This script were written by
# Eduard Generalov <eduard@generalov.net>
#
# They are Copyright 2018, and published under the MIT,

#DEBIRF_PACKAGE>+firmware-iwlwifi
#DEBIRF_PACKAGE>+firmware-misc-nonfree
#DEBIRF_PACKAGE>+wpasupplicant

echo 'iwlwifi' >> $DEBIRF_ROOT/etc/modules

cat << EOF > $DEBIRF_ROOT/etc/wpa_supplicant/wpa_supplicant-wlp1s0.conf
ctrl_interface=/run/wpa_supplicant
update_config=1
network={
        ssid="WiFi_SSID"
        psk="WIFIPASSWORD"
}
EOF

cat << EOF > $DEBIRF_ROOT/etc/systemd/network/wireless.network
[Match]
Name=wlp1s0
[Network]
DHCP=ipv4
[DHCP]
RouteMetric=20
EOF

```

and replace line with `resolved` in file rescue/modules/network with `debirf_exec systemctl enable wpa_supplicant@wlp1s0.service systemd-networkd.service systemd-resolved.service`


### Bonus: lxc on LiR

module rescue/modules/lxc

```bash
#!/bin/sh -e

# debirf module: lxc
# prepare lxc
#
# This script were written by
# Eduard Generalov <eduard@generalov.net>
#
# They are Copyright 2018, and published under the MIT,

#DEBIRF_PACKAGE>+lxc

mkdir -p $DEBIRF_ROOT/root/.ssh/
ssh-keygen -b 2048 -t rsa -f $DEBIRF_ROOT/root/.ssh/id_rsa -q -N ""
cp $DEBIRF_ROOT/root/.ssh/id_rsa $DEBIRF_ROOT/root/.ssh/authorized_keys
chmod 400 $DEBIRF_ROOT/root/.ssh/authorized_keys

debirf_exec systemctl enable lxc-net


cat << EOF > $DEBIRF_ROOT/etc/lxc/default.conf
lxc.network.type = veth
lxc.network.link = lxc
lxc.network.name = eth0
lxc.network.flags = up
lxc.network.hwaddr = 00:FF:AA:FF:xx:xx

lxc.mount.entry=/var/cache/apt var/cache/apt none bind,rw 0 0
lxc.mount.entry = /root/.ssh/ root/.ssh none bind,create=dir 0 0
EOF

cat << EOF > $DEBIRF_ROOT/etc/default/lxc-net 
USE_LXC_BRIDGE="true"
LXC_BRIDGE="lxc"
LXC_ADDR="10.0.3.1"
LXC_NETMASK="255.255.255.0"
LXC_NETWORK="10.0.3.0/24"
LXC_DHCP_RANGE="10.0.3.2,10.0.3.254"
LXC_DHCP_MAX="253"
LXC_DHCP_CONFILE=""
LXC_DOMAIN="lxc"
EOF
```
