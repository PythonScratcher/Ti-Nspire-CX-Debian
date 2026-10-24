
# Debian 9 Stretch on the TI-Nspire CX



you will need:

-   TI-Nspire CX (the normal cx or cx cas, NOT the cx II its different hardware and wont work)
-   ndless on the calculator
-   a usb drive (debian goes on this)
-   a usb otg cable/adapter for the calcs mini usb port
-   a **powered** usb hub if you want to plug in anything thats not just a flash drive
-   a linux computer or VM (debian/ubuntu/raspberry pi os)
-   from this repo:
    -   `linuxloader2.tns`
    -   `zImage.tns` (in /linux folder)
    -   `nspire.dtb.tns` (in /linux folder)
    -   `start_usb.ll2.tns`(in /linux folder)

the usb drive will have the debian fs

dont mix the `zImage.tns` and `nspire.dtb.tns` from here with ones from somewhere else, use both from this repo or it might not boot

## 1. prepare the usb

you should probably use a debian based computer (i used my rpi)

first install the packages needed:

```
sudo apt update
sudo apt install debootstrap qemu-user-static binfmt-support
```

plug in the usb drive and find its device name:

```
lsblk
```

you should see something similar to:

```
sda
├─sda1
└─sda2
sdb
└─sdb1
```

find which one is your usb drive. (usually its the one that wasnt there before) **this will wipe everything on it** so back it up if you need to

for this guide i will use:

```
/dev/sdb
```

unmount it:

```
sudo umount /dev/sdb*
```

if the usb doesnt already have one partition make one with:

```
sudo fdisk /dev/sdb
```

in fdisk press `o` (new partition table) then `n` (new partition, just press enter for all the defaults so it uses the whole drive) then `w` to save and quit

then make it ext4:

```
sudo mkfs.ext4 -L nspire /dev/sdb1
```

it has to be the first partition (shows up as `/dev/sda1` on the calc) and it has to be ext2/3/4. fat/exfat/ntfs wont work for this


## 2. mount the usb

make a mount point:

```
sudo mkdir -p /mnt/nspire
```

then mount the usb:

```
sudo mount /dev/sdb1 /mnt/nspire
```

check it mounted:

```
df -h /mnt/nspire
```

## 3. install debian 9 stretch

use `debootstrap` to make a minimal arm debian install (this will probably take ages)

```
sudo debootstrap --arch=armel --foreign stretch /mnt/nspire http://archive.debian.org/debian/
```

the nspire needs the **32 bit arm `armel`** userspace, not armhf or arm64 or amd64. the nspires cpu (arm926) doesnt have hardware floating point so armhf stuff just crashes with "Illegal instruction"

now copy the arm emulator into the new system (you need this on a normal x86 pc, doesnt hurt on a pi):

```
sudo cp /usr/bin/qemu-arm-static /mnt/nspire/usr/bin/
```

run the second stage of the install inside the new system:

```
sudo chroot /mnt/nspire /debootstrap/debootstrap --second-stage
```

when thats done debian is installed on the usb.

## 4. set up the basic debian system

mount the system folders (apt doesnt work properly in the chroot without these) and then go into the filesystem:

```
for d in proc sys dev dev/pts; do sudo mount --bind /$d /mnt/nspire/$d; done
sudo cp /etc/resolv.conf /mnt/nspire/etc/resolv.conf
sudo chroot /mnt/nspire /bin/bash
```

set the hostname:

```
echo nspire > /etc/hostname
```

make `/etc/hosts` (might not be needed but i saw it on a forum ages ago and i dont know if it breaks something if you dont):

```
cat > /etc/hosts <<'EOF'
127.0.0.1       localhost
127.0.1.1       nspire
EOF
```

set the root password:

```
passwd
```

make `/etc/fstab` so the drive mounts read/write and uses a swap file (without this it can come up read only which is really annoying):

```
cat > /etc/fstab <<'EOF'
/dev/sda1   /           ext4   defaults,noatime   0 1
/swapfile   none        swap   sw                 0 0
EOF
```

make the swap file (the nspire only has like 54mb of ram you can actualy use so you NEED swap):

```
dd if=/dev/zero of=/swapfile bs=1M count=256
chmod 600 /swapfile
mkswap /swapfile
```

set up apt:

```
cat > /etc/apt/sources.list <<'EOF'
deb http://archive.debian.org/debian stretch main
EOF
echo 'Acquire::Check-Valid-Until "false";' > /etc/apt/apt.conf.d/99archive
```

(stretch-updates doesnt exist on the archive anymore so dont add it, and the Check-Valid-Until line stops apt complaining that the archive is too old)

**install everything you want NOW from the computer.** apt normally causes the calculator to kill the process because its out of memory (OOM) when you run it on the calculator itself, even with swap

then:

```
apt update
apt install bash coreutils procps iproute2 net-tools nano isc-dhcp-client dropbear-run
```

make sure dropbear is allowed to start on boot:

```
sed -i 's/^NO_START=1/NO_START=0/' /etc/default/dropbear
```

`dropbear-run` is the ssh server. its alot lighter than openssh which matters on the calculator, and it lets root log in with a password straight away so theres nothing to set up. (dont install the `dropbear` package itself, that one pulls in initramfs stuff you dont need)

### networking (usb ethernet adapters / phone tethering)

the kernel in this repo has most of the common usb ethernet drivers built in (realtek rtl8152/8153/8156, asix ax88772/ax88179, smsc/microchip, davicom, corechip, cdc-ecm/ncm, rndis and iphone tethering).

you cant really set this up from the computer because you dont know what the adapter will be called yet. debian names it from the adapters mac address so it'll be something like `enxc84d44296e4a` (phones doing usb tethering usually show up as `usb0` instead). so do this part after the first boot (step 10). for now just leave the loopback in `/etc/network/interfaces`:

```
cat > /etc/network/interfaces <<'EOF'
auto lo
iface lo inet loopback
EOF
```

### optionally a gui

lightweight desktop (xorg + fluxbox + xterm), install it from the computer its WAY too slow on the calculator:

```
apt install --no-install-recommends -y xserver-xorg-core xserver-xorg-video-fbdev xserver-xorg-input-evdev xinit fluxbox xterm
mkdir -p /etc/X11/xorg.conf.d
cat > /etc/X11/xorg.conf.d/10-nspire.conf <<'EOF'
Section "Device"
    Identifier "nspire-fb"
    Driver "fbdev"
    Option "fbdev" "/dev/fb0"
EndSection
EOF
echo "exec startfluxbox" > /root/.xinitrc
```

i used fluxbox cause its way smaller than jwm on stretch. other light apps that work ok: `xfe` (file manager and `xfw` text editor), `dillo` (web browser), `htop`, `mc`, `neofetch`

exit the chroot and unmount the system folders:

```
exit
for d in dev/pts dev sys proc; do sudo umount /mnt/nspire/$d; done
```

## 5. unmount the usb

remove the emulator (the calc doesnt need it) and make sure everything has been written:

```
sudo rm /mnt/nspire/usr/bin/qemu-arm-static
sync
```

then:

```
sudo umount /mnt/nspire
```

the debian usb is now ready.

----------

## 6. put linuxloader2 on the nspire

your files should be like this on the calculator (send them with ti nspire computer link or tilp, they have to keep the `.tns` on the end):

```
ndless/
    linuxloader2.tns
linux/
    zImage.tns
    nspire.dtb.tns
    start_usb.ll2.tns
```

put `linuxloader2.tns` in the `ndless` folder.

make the `linux` folder and put these three files in it:

```
zImage.tns
nspire.dtb.tns
start_usb.ll2.tns
```

`start_usb.ll2.tns` is just a text file that tells linuxloader2 what to do. it should have this in it:

```
kernel linux/zImage.tns
dtb linux/nspire.dtb.tns
cmdline root=/dev/sda1 rootdelay=10 rw
boot
```

-   `rootdelay=10` waits for the usb to be found before it trys to boot from it
-   `rw` mounts it read/write


## 7. configure ndless

linuxloader2 needs to be registered as an extension so `.ll2.tns` scripts can be opened.

open:

```
/ documents / ndless / ndless.cfg.tns
```

and add:

```
ext.ll2=linuxloader2
```


## 8. connect the usb drive

connect the usb drive to the nspire with the otg adapter.

the nspires usb gives out very little power, so a basic flash drive is the only thing that really works on its own. a powered usb hub **WILL** be needed for bigger stuff (ethernet adapters, keyboards, mice.) plug the flash drive into the hub too if you are using one

## 9. boot debian

open:

```
linux/start_usb.ll2.tns
```

linuxloader2 should load:

```
zImage.tns
nspire.dtb.tns
```

and boot the kernel. after about 10 seconds the kernel should find the debian filesystem on:

```
/dev/sda1
```

and start debian. first boot is slow (a few mins is normal). log in as `root` with the password you set

## 10. check debian is running

once you get a shell:

```
uname -a
```

then:

```
cat /etc/debian_version
```

you should get a debian 9 stretch version.

check the architecture:

```
uname -m
```

it should say `armv5tejl`

check swap is on:

```
free -m
```

### set up the network

plug in your ethernet adapter (through the powered hub) and find out what its called:

```
ip link
```

ignore `lo`, `tunl0` and `sit0`. yours is the other one, something like `enxc84d44296e4a` or `usb0`. for the rest of this im using `enxc84d44296e4a`, change it to yours

test it:

```
ip link set enxc84d44296e4a up
dhclient enxc84d44296e4a
ping -c 3 debian.org
```

if that works make it connect on every boot:

```
cat >> /etc/network/interfaces <<'EOF'

allow-hotplug enxc84d44296e4a
iface enxc84d44296e4a inet dhcp
EOF
```

the name stays the same as long as you use the same adapter. if you change adapter you have to do this again with the new name

get the calcs ip with `ip addr`, then you can ssh in from your computer (`ssh root@<the calculators ip>`) which is way easier than typing on the keypad lol

## 11. using the calculator

-   **keypad**: works as a keyboard. `ctrl` is bottom left, `ctrl` + `c` stops stuff
-   **touchpad**: moves the mouse in the gui. pressing the pad is left click, pressing in the **bottom right corner** is right click. you can change the speed and how big the right click bit is without rebooting:
    ```
    echo 10 > /sys/module/nspire_touchpad/parameters/speed
    echo 35 > /sys/module/nspire_touchpad/parameters/right_zone
    ```
-   **gui**: `startx` (exit from the right click menu in fluxbox)
-   **clock**: https needs the right time. if `date` is wrong set it with `date -s "2026-10-24 17:00"` or install `ntpdate` from the computer
-   **turning it off**: do `poweroff`, wait till the screen stops changing, then press the reset button on the back to go back to the normal calculator os

## 12. optional: overclocking

the cx normally runs at 132mhz. the ndless program **nover** (its on ticalc.org) can make it go to like 220-240mhz before you start linux:

1.  put `nover.tns` in `ndless/startup/`
2.  reboot the calc. nover looks for the fastest speed that works (the calc might freeze while its doing this, just press reset and let it carry on)
3.  boot linux like normal, linux keeps the faster speed

check with `cat /proc/cpuinfo`, on this cpu bogomips is about **half** the mhz (around 66 normally, 115ish when overclocked)

to undo it take `nover.tns` out of `ndless/startup/` and reset.

## 13. troubleshooting

-   **`Kernel panic - VFS: Unable to mount root fs on unknown-block(0,0)`**: the usb wasnt found. check the otg adapter, use a powered hub, try a diffrent flash drive and make sure `rootdelay=10` is in the cmdline
-   **kernel panic but the drive was found (you see `sda` in the messages)**: the first partition isnt ext4 or debian isnt on it, redo step 1-3
-   **`Read-only file system`**: `/etc/fstab` is missing or `rw` isnt in the cmdline. quick fix: `mount -o remount,rw /`
-   **network adapter not found (`ip link` only shows `lo`, `tunl0`, `sit0`)**: use a powered hub. check `dmesg | tail -30`, if the adapter shows up but nothing uses it then its a chip the kernel doesnt have a driver for
-   **adapter is there but no ip**: do `ip link set <name> up` then `dhclient <name>`. if ping to an ip works but names dont: `echo "nameserver 1.1.1.1" > /etc/resolv.conf`
-   **network worked before but not after changing adapter**: the new adapter has a different name, redo "set up the network" in step 10
-   **stuff gets `Killed`**: out of memory. make sure swap is on (`free -m` should show about 255mb swap, if not `swapon /swapfile`). install stuff from the computer (step 4) not with apt on the calc
-   **https / websites give certificate errors**: the calculators clock is wrong, see step 11
-   **traffic going over ipv6 when it shouldnt (like going around a vpn)**: turn ipv6 off with `sysctl -w net.ipv6.conf.all.disable_ipv6=1` (put `net.ipv6.conf.all.disable_ipv6=1` in `/etc/sysctl.conf` to make it stay)
-   **"Text file busy" when replacing a program**: its still running, stop it first or write it to a new file and `mv` it over the old one

## 14. optional: windscribe cli (vpn)

this is the real windscribe cli app (v2.25.5), cross compiled for the nspire. its mostly a proof of concept, its really slow on the calculator but it does actualy log in and connect.

you need `windscribe-nspire.tar.gz` from this repo and a windscribe account (free works)

**do this on the computer** while the usb is still mounted and you are in the chroot (step 4, before you `exit`). unpacking it on the calculator takes ages.

copy the file onto the usb first (from a normal terminal on the computer, not the chroot):

```
sudo cp windscribe-nspire.tar.gz /mnt/nspire/root/
```

then in the chroot:

```
tar -xzf /root/windscribe-nspire.tar.gz -C /
rm /root/windscribe-nspire.tar.gz
groupadd windscribe
useradd -r -g windscribe -s /bin/false windscribe
useradd -m -s /bin/bash nspire
passwd nspire
usermod -aG windscribe nspire
chgrp windscribe /opt/windscribe/Windscribe
chmod 2755 /opt/windscribe/Windscribe
ln -sf /opt/windscribe/windscribe-cli /usr/bin/windscribe-cli
mkdir -p /var/log/windscribe
systemctl enable windscribe-helper
echo "net.ipv6.conf.all.disable_ipv6=1" >> /etc/sysctl.conf
```

- the `nspire` user is needed because windscribe wont run as root (only its helper runs as root, thats what the systemd service is)
- the `chgrp` and `chmod 2755` bits are important, without them the app cant talk to its helper
- ipv6 gets turned off because otherwise ipv6 traffic goes around the vpn (see "what doesnt work" below)

### using it

on the calculator (or over ssh), log in as `nspire` not root:

```
su - nspire
windscribe login
```

`windscribe` (not `windscribe-cli`) is a launcher that starts the windscribe engine in the background first if its not already running, then runs the cli. starting the engine takes 1-3 mins. you might get a captcha made of `#` symbols, just type the numbers.

then connect:

```
windscribe connect best udp
```

the first connect took me about **10 minutes**, just leave it.

**DONT press `ctrl` + `c` after it says `Connected`**, that kills the vpn connection. the command just stays there while you are connected, so leave it running and use another tty (or ssh in) for everything else. in the other tty / ssh session check your ip:

```
windscribe status
wget -4 -qO- icanhazip.com
```

it should be a windscribe ip not yours. disconnect with `windscribe disconnect` (from the other tty)

### what doesnt work

- **only openvpn (`udp` or `tcp`)**. wireguard needs `IP_MULTIPLE_TABLES` (policy routing) in the kernel and this kernel doesnt have it
- **no kill switch / firewall**. the kernel has no netfilter so the firewall is just faked, turning it on does nothing
- **dns doesnt go through the vpn** because stretch doesnt have the `resolvconf` thing windscribes dns script uses
- **ipv6 leaks** if you dont turn ipv6 off (its turned off in the install above)
- its **slow**. everything (login, getting the server list, connecting) takes minutes. turn on swap and overclock if you can

### windscribe troubleshooting

- **`Could not start application` / `Failed to connect to bus`**: you ran `windscribe-cli` before the engine was running. use `windscribe` instead which starts it for you
- **`Error: this application should not run as root`**: do `su - nspire` first
- **`Could not log in`**: the calculator is too slow and requests time out. check the clock is right (`date`), then restart the engine (`pkill -9 -x Windscribe`) and try `windscribe login` again. dont turn on `ignoresslerrors`, it wont help and its not safe
- **stuck on "Logging in" forever**: restart the engine with `pkill -9 -x Windscribe` and log in again
- **ip doesnt change after connecting**: you probably pressed `ctrl` + `c` on the connect command, that disconnects it. connect again and leave it running, use another tty or ssh. also make sure ipv6 is off and you used `wget -4`
- **logs**: `~/.local/share/Windscribe/Windscribe2/client.log` (app) and `/var/log/windscribe/helper.log` (helper)
