# 在 TI-Nspire CX 上运行 Debian 9 Stretch

[English](README.md) | 简体中文

你需要准备：

-   TI-Nspire CX（普通的 CX 或 CX CAS，**不是** CX II，CX II 的硬件不一样，用不了）
-   计算器上装好 ndless
-   一个 U 盘（Debian 装在这上面）
-   一根适配计算器 mini USB 口的 USB OTG 线/转接头
-   如果要接 U 盘以外的东西，需要一个**带电源的** USB 集线器
-   一台 Linux 电脑或虚拟机（Debian/Ubuntu/Raspberry Pi OS）
-   本仓库中的以下文件：
    -   `linuxloader2.tns`
    -   `zImage.tns`（在 /linux 文件夹里）
    -   `nspire.dtb.tns`（在 /linux 文件夹里）
    -   `start_usb.ll2.tns`（在 /linux 文件夹里）

U 盘上放的是 Debian 的文件系统。

不要把这里的 `zImage.tns` 和 `nspire.dtb.tns` 跟别处下载的混用，两个都要用本仓库的，否则可能无法启动。

## 1. 准备 U 盘

最好用一台基于 Debian 的电脑（我用的是树莓派）。

先安装需要的软件包：

```
sudo apt update
sudo apt install debootstrap qemu-user-static binfmt-support
```

插上 U 盘，找到它的设备名：

```
lsblk
```

你应该会看到类似这样的输出：

```
sda
├─sda1
└─sda2
sdb
└─sdb1
```

找出哪个是你的 U 盘（通常就是插上之前没有的那个）。**这会清空 U 盘上的所有数据**，有需要的话先备份。

本教程中使用：

```
/dev/sdb
```

先卸载它：

```
sudo umount /dev/sdb*
```

如果 U 盘上还没有一个分区，用下面的命令建一个：

```
sudo fdisk /dev/sdb
```

在 fdisk 里按 `o`（新建分区表），然后按 `n`（新建分区，一路回车用默认值，这样会占满整个 U 盘），最后按 `w` 保存并退出。

然后格式化成 ext4：

```
sudo mkfs.ext4 -L nspire /dev/sdb1
```

它必须是第一个分区（在计算器上显示为 `/dev/sda1`），并且必须是 ext2/3/4 格式。FAT/exFAT/NTFS 都不行。


## 2. 挂载 U 盘

创建挂载点：

```
sudo mkdir -p /mnt/nspire
```

然后挂载 U 盘：

```
sudo mount /dev/sdb1 /mnt/nspire
```

检查是否挂载成功：

```
df -h /mnt/nspire
```

## 3. 安装 Debian 9 Stretch

用 `debootstrap` 制作一个最小化的 ARM Debian 系统（这一步可能要很久）：

```
sudo debootstrap --arch=armel --foreign stretch /mnt/nspire http://archive.debian.org/debian/
```

Nspire 需要的是 **32 位 ARM `armel`** 用户空间，不是 armhf、arm64 或 amd64。Nspire 的 CPU（ARM926）没有硬件浮点，所以 armhf 的程序会直接报 "Illegal instruction" 崩溃。

然后把 ARM 模拟器复制到新系统里（在普通 x86 电脑上必须这样做，在树莓派上做了也没坏处）：

```
sudo cp /usr/bin/qemu-arm-static /mnt/nspire/usr/bin/
```

在新系统里运行安装的第二阶段：

```
sudo chroot /mnt/nspire /debootstrap/debootstrap --second-stage
```

完成后 Debian 就已经装到 U 盘上了。

## 4. 配置基础 Debian 系统

挂载系统目录（没有这些，apt 在 chroot 里没法正常工作），然后进入文件系统：

```
for d in proc sys dev dev/pts; do sudo mount --bind /$d /mnt/nspire/$d; done
sudo cp /etc/resolv.conf /mnt/nspire/etc/resolv.conf
sudo chroot /mnt/nspire /bin/bash
```

设置主机名：

```
echo nspire > /etc/hostname
```

创建 `/etc/hosts`（可能不是必须的，但我很久以前在论坛上看到过，不确定不加会不会出问题）：

```
cat > /etc/hosts <<'EOF'
127.0.0.1       localhost
127.0.1.1       nspire
EOF
```

设置 root 密码：

```
passwd
```

创建 `/etc/fstab`，让磁盘以读写方式挂载并启用交换文件（没有它，系统可能会以只读方式启动，非常烦人）：

```
cat > /etc/fstab <<'EOF'
/dev/sda1   /           ext4   defaults,noatime   0 1
/swapfile   none        swap   sw                 0 0
EOF
```

创建交换文件（Nspire 实际可用的内存只有大约 54MB，所以**必须**有 swap）：

```
dd if=/dev/zero of=/swapfile bs=1M count=256
chmod 600 /swapfile
mkswap /swapfile
```

配置 apt：

```
cat > /etc/apt/sources.list <<'EOF'
deb http://archive.debian.org/debian stretch main
EOF
echo 'Acquire::Check-Valid-Until "false";' > /etc/apt/apt.conf.d/99archive
```

（存档里已经没有 stretch-updates 了，所以别加；Check-Valid-Until 那一行是为了让 apt 不再抱怨存档太旧。）

**现在就在电脑上把你想要的东西全部装好。** 在计算器上直接运行 apt 通常会因为内存不足（OOM）被系统杀掉，即使开了 swap 也一样。

然后：

```
apt update
apt install bash coreutils procps iproute2 net-tools nano isc-dhcp-client dropbear-run
```

确保 dropbear 允许开机启动：

```
sed -i 's/^NO_START=1/NO_START=0/' /etc/default/dropbear
```

`dropbear-run` 是 SSH 服务器。它比 OpenSSH 轻量得多，这在计算器上很重要，而且它默认允许 root 用密码直接登录，不需要额外配置。（不要安装 `dropbear` 这个包本身，它会拉进来一堆用不到的 initramfs 相关的东西。）

### 网络（USB 网卡 / 手机 USB 共享网络）

本仓库的内核已内置大多数常见的 USB 网卡驱动（Realtek RTL8152/8153/8156、ASIX AX88772/AX88179、SMSC/Microchip、Davicom、CoreChip、CDC-ECM/NCM、RNDIS 以及 iPhone 共享网络）。

这部分没法在电脑上提前配置，因为你还不知道网卡会叫什么名字。Debian 会根据网卡的 MAC 地址命名，比如 `enxc84d44296e4a`（手机 USB 共享网络通常显示为 `usb0`）。所以这一步要在第一次启动之后做（第 10 步）。现在先在 `/etc/network/interfaces` 里只保留回环接口：

```
cat > /etc/network/interfaces <<'EOF'
auto lo
iface lo inet loopback
EOF
```

### 可选：图形界面

轻量桌面（xorg + fluxbox + xterm），要在电脑上装，在计算器上装**太慢了**：

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

我用 fluxbox 是因为在 stretch 上它比 jwm 小得多。其他能正常用的轻量程序：`xfe`（文件管理器，附带 `xfw` 文本编辑器）、`dillo`（网页浏览器）、`htop`、`mc`、`neofetch`。

退出 chroot 并卸载系统目录：

```
exit
for d in dev/pts dev sys proc; do sudo umount /mnt/nspire/$d; done
```

## 5. 卸载 U 盘

删除模拟器（计算器用不到它），并确保所有数据都已写入：

```
sudo rm /mnt/nspire/usr/bin/qemu-arm-static
sync
```

然后：

```
sudo umount /mnt/nspire
```

Debian U 盘就做好了。

----------

## 6. 把 linuxloader2 放到 Nspire 上

计算器上的文件结构应该是这样的（用 TI-Nspire Computer Link 或 TiLP 传输，文件名末尾必须保留 `.tns`）：

```
ndless/
    linuxloader2.tns
linux/
    zImage.tns
    nspire.dtb.tns
    start_usb.ll2.tns
```

把 `linuxloader2.tns` 放进 `ndless` 文件夹。

新建 `linux` 文件夹，把下面三个文件放进去：

```
zImage.tns
nspire.dtb.tns
start_usb.ll2.tns
```

`start_usb.ll2.tns` 只是一个告诉 linuxloader2 该做什么的文本文件，内容如下：

```
kernel linux/zImage.tns
dtb linux/nspire.dtb.tns
cmdline root=/dev/sda1 rootdelay=10 rw
boot
```

-   `rootdelay=10`：等待 U 盘被识别后再尝试从它启动
-   `rw`：以读写方式挂载


## 7. 配置 ndless

需要把 linuxloader2 注册为扩展，这样才能打开 `.ll2.tns` 脚本。

打开：

```
/ documents / ndless / ndless.cfg.tns
```

添加：

```
ext.ll2=linuxloader2
```


## 8. 连接 U 盘

用 OTG 转接头把 U 盘接到 Nspire 上。

Nspire 的 USB 口供电非常弱，单独接的话基本只有普通 U 盘能用。接更大的设备（网卡、键盘、鼠标）**必须**用带电源的 USB 集线器。如果用了集线器，U 盘也一起插在集线器上。

## 9. 启动 Debian

打开：

```
linux/start_usb.ll2.tns
```

linuxloader2 会加载：

```
zImage.tns
nspire.dtb.tns
```

并启动内核。大约 10 秒后，内核应该会在下面的位置找到 Debian 文件系统：

```
/dev/sda1
```

然后启动 Debian。第一次启动很慢（几分钟是正常的）。用 `root` 和你设置的密码登录。

## 10. 检查 Debian 是否正常运行

进入 shell 后运行：

```
uname -a
```

然后：

```
cat /etc/debian_version
```

应该会显示 Debian 9 Stretch 的版本号。

检查架构：

```
uname -m
```

应该显示 `armv5tejl`。

检查 swap 是否已启用：

```
free -m
```

### 配置网络

插上网卡（通过带电源的集线器），查看它的名字：

```
ip link
```

忽略 `lo`、`tunl0` 和 `sit0`，剩下那个就是你的网卡，比如 `enxc84d44296e4a` 或 `usb0`。下面以 `enxc84d44296e4a` 为例，记得换成你自己的。

测试一下：

```
ip link set enxc84d44296e4a up
dhclient enxc84d44296e4a
ping -c 3 debian.org
```

如果能通，就设置成每次开机自动连接：

```
cat >> /etc/network/interfaces <<'EOF'

allow-hotplug enxc84d44296e4a
iface enxc84d44296e4a inet dhcp
EOF
```

只要用的是同一个网卡，名字就不会变。换了网卡的话要用新名字重新做一遍。

用 `ip addr` 查看计算器的 IP，然后就可以从电脑 SSH 登录了（`ssh root@<计算器的 IP>`），比在键盘上打字方便太多了 lol

## 11. 使用计算器

-   **键盘**：可以当键盘用。`ctrl` 在左下角，`ctrl` + `c` 可以中止程序
-   **触摸板**：在图形界面里移动鼠标。按下触摸板是左键，按**右下角**是右键。速度和右键区域大小可以不重启直接修改：
    ```
    echo 10 > /sys/module/nspire_touchpad/parameters/speed
    echo 35 > /sys/module/nspire_touchpad/parameters/right_zone
    ```
-   **图形界面**：`startx`（在 fluxbox 的右键菜单里退出）
-   **时钟**：HTTPS 需要正确的时间。如果 `date` 不对，用 `date -s "2026-10-24 17:00"` 设置，或者在电脑上装好 `ntpdate`
-   **关机**：执行 `poweroff`，等屏幕不再变化后，按背面的复位按钮回到计算器原来的系统

## 12. 可选：超频

CX 默认运行在 132MHz。ndless 程序 **nover**（在 ticalc.org 上）可以在启动 Linux 之前把频率提到 220–240MHz 左右：

1.  把 `nover.tns` 放进 `ndless/startup/`
2.  重启计算器。nover 会自动寻找能稳定运行的最高频率（期间计算器可能会卡死，按复位让它继续就行）
3.  像平常一样启动 Linux，Linux 会保持超频后的速度

用 `cat /proc/cpuinfo` 检查。在这颗 CPU 上，bogomips 大约是 MHz 的**一半**（正常约 66，超频后约 115）。

要撤销超频，把 `nover.tns` 从 `ndless/startup/` 删掉然后复位即可。

## 13. 故障排除

-   **`Kernel panic - VFS: Unable to mount root fs on unknown-block(0,0)`**：没找到 U 盘。检查 OTG 转接头，用带电源的集线器，换一个 U 盘试试，并确认 cmdline 里有 `rootdelay=10`
-   **内核崩溃但找到了磁盘（输出里能看到 `sda`）**：第一个分区不是 ext4，或者上面没有 Debian，重做第 1–3 步
-   **`Read-only file system`**：缺少 `/etc/fstab`，或 cmdline 里没有 `rw`。临时解决：`mount -o remount,rw /`
-   **找不到网卡（`ip link` 只显示 `lo`、`tunl0`、`sit0`）**：用带电源的集线器。查看 `dmesg | tail -30`，如果网卡出现了但没有驱动接管，说明内核没有这个芯片的驱动
-   **有网卡但没有 IP**：先 `ip link set <名字> up`，再 `dhclient <名字>`。如果能 ping 通 IP 但域名不行：`echo "nameserver 1.1.1.1" > /etc/resolv.conf`
-   **换网卡后网络不通了**：新网卡名字不同，重做第 10 步的"配置网络"
-   **程序被 `Killed`**：内存不足。确认 swap 已开启（`free -m` 应显示约 255MB swap，没有的话执行 `swapon /swapfile`）。软件要在电脑上装（第 4 步），不要在计算器上用 apt
-   **HTTPS / 网站报证书错误**：计算器时间不对，见第 11 步
-   **流量走了 IPv6（比如绕过了 VPN）**：用 `sysctl -w net.ipv6.conf.all.disable_ipv6=1` 关闭 IPv6（在 `/etc/sysctl.conf` 里加上 `net.ipv6.conf.all.disable_ipv6=1` 可以永久生效）
-   **替换程序时提示 "Text file busy"**：程序还在运行，先停掉它，或者写到一个新文件再用 `mv` 覆盖旧文件

## 14. 可选：Windscribe CLI（VPN）

这是真正的 Windscribe CLI 程序（v2.25.5），为 Nspire 交叉编译的。基本上算是概念验证，在计算器上非常慢，但确实能登录并连上。

你需要本仓库里的 `windscribe-nspire.tar.gz` 和一个 Windscribe 账号（免费的就行）。

**在电脑上做这一步**，趁 U 盘还挂载着并且你还在 chroot 里（第 4 步，在 `exit` 之前）。在计算器上解压要很久。

先把文件复制到 U 盘上（在电脑的普通终端里，不是在 chroot 里）：

```
sudo cp windscribe-nspire.tar.gz /mnt/nspire/root/
```

然后在 chroot 里：

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

- 需要 `nspire` 用户，因为 Windscribe 不能以 root 运行（只有它的 helper 以 root 运行，也就是那个 systemd 服务）
- `chgrp` 和 `chmod 2755` 这两步很重要，没有它们程序没法和 helper 通信
- 关闭 IPv6 是因为否则 IPv6 流量会绕过 VPN（见下面的"不支持的功能"）

### 使用方法

在计算器上（或通过 SSH），以 `nspire` 而不是 root 登录：

```
su - nspire
windscribe login
```

`windscribe`（不是 `windscribe-cli`）是一个启动器，如果 Windscribe 引擎还没运行，它会先在后台启动引擎，再运行 CLI。启动引擎需要 1–3 分钟。你可能会看到一个用 `#` 符号组成的验证码，输入里面的数字就行。

然后连接：

```
windscribe connect best udp
```

我第一次连接花了大约 **10 分钟**，耐心等着。

**显示 `Connected` 之后不要按 `ctrl` + `c`**，那样会断开 VPN。连接期间这个命令会一直挂着，让它保持运行，用另一个 tty（或 SSH）做其他事。在另一个 tty / SSH 会话里检查 IP：

```
windscribe status
wget -4 -qO- icanhazip.com
```

应该显示的是 Windscribe 的 IP 而不是你自己的。在另一个 tty 里用 `windscribe disconnect` 断开连接。

### 不支持的功能

- **只能用 OpenVPN（`udp` 或 `tcp`）**。WireGuard 需要内核支持 `IP_MULTIPLE_TABLES`（策略路由），而这个内核没有
- **没有 kill switch / 防火墙**。内核没有 netfilter，所以防火墙只是假装开启，打开了也没用
- **DNS 不走 VPN**，因为 stretch 没有 Windscribe 的 DNS 脚本所需的 `resolvconf`
- 如果不关闭 IPv6，**IPv6 会泄漏**（上面的安装步骤已经关闭了）
- **很慢**。所有操作（登录、获取服务器列表、连接）都要好几分钟。有条件的话开启 swap 并超频

### Windscribe 故障排除

- **`Could not start application` / `Failed to connect to bus`**：在引擎启动前就运行了 `windscribe-cli`。改用 `windscribe`，它会自动启动引擎
- **`Error: this application should not run as root`**：先执行 `su - nspire`
- **`Could not log in`**：计算器太慢导致请求超时。检查时间是否正确（`date`），然后重启引擎（`pkill -9 -x Windscribe`）再试一次 `windscribe login`。不要开启 `ignoresslerrors`，没用而且不安全
- **一直卡在 "Logging in"**：用 `pkill -9 -x Windscribe` 重启引擎后重新登录
- **连接后 IP 没变**：你可能在 connect 命令上按了 `ctrl` + `c`，这会断开连接。重新连接并保持它运行，用另一个 tty 或 SSH 操作。同时确认 IPv6 已关闭，并且用的是 `wget -4`
- **日志**：`~/.local/share/Windscribe/Windscribe2/client.log`（程序）和 `/var/log/windscribe/helper.log`（helper）
