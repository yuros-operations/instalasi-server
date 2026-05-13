## 1. partitioning
### physical layout nvme0n1
| disk | partition | type              | luks  | lvm   | label    | size      | format | mount                      |
| ---- | --------- | ----------------- | ----- | ----- | -------- | --------- | ------ | -------------------------- |
| 0    | 1         | efi               | false | false | boot     | 2.5G      | fat 32 | /boot                      |
| 0    | 2         | linux server data | true  | false | keys     | 512M      | luks   | none                       |
| 0    | 3         | linux file system | true  | true  | proc     | 50G       | luks   | see logical volume proc    |
| 0    | 4         | linux server data | true  | true  | pond     | 100% Free | luks   | see logical volume pond    |

### physical layout sda 

| disk | partition | type              | luks  | lvm   | label    | size      | format | mount                      |
| ---- | --------- | ----------------- | ----- | ----- | -------- | --------- | ------ | -------------------------- |
| 1    | 1         | linux file system | true  | true  | -        | 5 G       | -      | -                          |
| 1    | 2         | linux server data | true  | true  | cave     | 100% Free | luks   | see logical volume cave    |

### logical volume layout nvme0n1

#### logical volume proc
| partition | list | group  | name | size   | mount                | format |
| --------- | ----- | ------ | ---- | ------ | --------------------- | --------|
| 3         | 1     | proc   | root | 10G    | /mnt                  | xfs     |
| 3         | 2     | proc   | vars | 10G    | /mnt/var              | xfs     |
| 3         | 3     | proc   | vlog | 1.5G   | /mnt/var/log/         | xfs     |
| 3         | 4     | proc   | vaud | 1G     | /mnt/var/log/audit    | xfs     |
| 3         | 5     | proc   | vtmp | 2.5G   | /mnt/var/tmp/         | xfs     |
| 3         | 6     | proc   | vpac | 3G     | /mnt/var/cache/pacman | xfs     |
| 3         | 7     | proc   | swap | 4G     | swapon                | swap    |

#### logical volume pond
| partition | list  | group  | name | size    | mount                  | format  |
| --------- | ----- | ------ | ---- | -------- | ----------------------- | --------|
| 4         | 1     | pond   | home | 1G       | /mnt/home               | xfs     |
| 4         | 2     | pond   | srvc | 512M     | /mnt/srv                | xfs     |
| 4         | 2     | pond   | pods | 50%free  | /mnt/var/lib/containers | xfs     |

###### reng
```
sudo cryptsetup luksFormat --type luks2 \
    --align-payload 4096 \
    --sector-size 4096 \
    --label "reng" \
    /dev/nvme0n1p2
```
###### proc
```
sudo cryptsetup luksFormat --type luks2 \
    --align-payload 4096 \
    --sector-size 4096 \
    --label "proc" \
    /dev/nvme0n1p3
```
```
sudo cryptsetup open /dev/nvme0n1p3 proc \
    --perf-no_read_workqueue \
    --perf-no_write_workqueue \
    --persistent
```
```
pvcreate --dataalignment 4096 /dev/mapper/proc
```
```
vgcreate proc /dev/mapper/proc
```
###### pond
```
sudo cryptsetup luksFormat --type luks2 \
    --align-payload 4096 \
    --sector-size 4096 \
    --label "pond" \
    /dev/nvme0n1p4
```
```
sudo cryptsetup open /dev/nvme0n1p4 pond \
    --perf-no_read_workqueue \
    --perf-no_write_workqueue \
    --persistent
```
```
pvcreate --dataalignment 4096 /dev/mapper/pond
```
```
vgcreate pond /dev/mapper/pond
```

##### root partition
```
lvcreate -L 10G proc -n root
```
```
mkfs.xfs -s size=4096 /dev/proc/root
```
```
mount /dev/proc/root /mnt
```
##### boot
```
mkdir /mnt/boot
```
```
mount -o uid=0,gid=0,fmask=0077,dmask=0077 /dev/nvme0n1p1 /mnt/boot
```

##### vars partition
```
lvcreate -L 10G proc -n vars
```
```
mkfs.xfs -s size=4096 /dev/proc/vars
```
```
mkdir /mnt/var
```
```
mount -o rw,nodev,noexec,nosuid,relatime /dev/proc/vars /mnt/var
```

##### vlog partition
```
lvcreate -L 1.5G proc -n vlog
```
```
sudo mkfs.xfs -s size=4096 /dev/proc/vlog
```
```
mkdir /mnt/var/log
```
```
mount -o rw,nodev,noexec,nosuid,relatime /dev/proc/vlog /mnt/var/log
```

##### vaud partition
```
lvcreate -L 512M proc -n vaud
```
```
mkfs.xfs -s size=4096 /dev/proc/vaud
```
```
mkdir /mnt/var/log/audit
```
```
mount -o rw,nodev,noexec,nosuid,relatime /dev/proc/vaud /mnt/var/log/audit
```
##### vtmp partition
```
lvcreate -L 2.5G proc -n vtmp
```
```
mkfs.xfs -s size=4096 /dev/proc/vtmp
```
```
mkdir /mnt/var/tmp
```
```
mount -o rw,nodev,noexec,nosuid,relatime /dev/proc/vtmp /mnt/var/tmp
```

##### vpac partition
```
lvcreate -L 3G proc -n vpac
```
```
mkfs.xfs -s size=4096 /dev/proc/vpac
```
```
mkdir -p /mnt/var/cache/pacman
```
```
mount -o rw,nodev,noexec,nosuid,relatime /dev/proc/vpac /mnt/var/cache/pacman
```

##### swap partition
```
lvcreate -L 10G proc -n swap
```
```
mkswap /dev/proc/swap
```
```
swapon /dev/proc/swap
```

##### home partition
```
lvcreate -L 1G pond -n home
```
```
mkfs.xfs -s size=4096 /dev/pond/home
```
```
mkdir /mnt/home
```
```
mount -o rw,nodev,noexec,nosuid,relatime /dev/pond/home /mnt/home
```

##### srvc partition
```
lvcreate -L 512M pond -n srvc
```
```
mkfs.xfs -s size=4096 /dev/pond/srvc
```

```
mkdir -p /mnt/srv/http
```
```
mount -o rw,nodev,noexec,nosuid,relatime /dev/pond/srvc /mnt/srv/http
```

##### host partition
```
lvcreate -l50%free pond -n pods
```
```
mkfs.xfs -s size=4096 /dev/pond/host
```
```
mkdir -p /mnt/var/lib/containers
```
```
mount -o rw,nodev,noexec,nosuid,relatime /dev/pond/host /mnt/var/lib/containers
```

## 2. Instalation
### intel server
```
pacstrap /mnt linux-hardened linux-hardened-headers clevis luksmeta tpm2-tools scx-scheds linux-firmware-realtek linux-firmware-intel linux-firmware-other mkinitcpio intel-ucode libpwquality git base neovim lvm2 openssh ethtool iptables-nft firewalld apparmor rsync sudo debugedit fakeroot pkgconf bison gcc pcre flex wget make gcc curl tuned irqbalance which xfsprogs podman --noconfirm
```

### network configuration
```
cp /etc/systemd/network/20-ethernet.network /mnt/etc/systemd/network/
```

### generate partition layout
```
genfstab -U /mnt > /mnt/etc/fstab
```

## 3. chrooting

```
arch-chroot /mnt
```

### hostname

```
echo 'nama_hostname' > /etc/hostname
```


### local time

```
ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime
```

```
hwclock --systohc
```
```
mkdir /etc/systemd/timesyncd.conf.d
```
```
nvim /etc/systemd/timesyncd.conf.d/local.conf
```
```
[Time]
NTP=0.id.pool.ntp.org 1.id.pool.ntp.org 2.id.pool.ntp.org 3.id.pool.ntp.org
FallbackNTP=time.cloudflare.com time.google.com time.aws.com
```
```
timedatectl set-ntp true
```
```
timedatectl set-timezone Asia/Jakarta
```
```
timedatectl status
```
```
timedatectl show-timesync --all
```
```
systemctl enable systemd-timesyncd.service
```

### locale

```
nvim /etc/locale.gen
```
uncommenting
```
en_US.UTF-8 UTF-8
en_US ISO-8859-1
```
```
locale-gen && locale > /etc/locale.conf
```
```
sed -i 's/^LANG=C.UTF-8/LANG=en_US.UTF-8/' /etc/locale.conf
```
```
sed -i 's/^LC_ALL=/LC_ALL=en_US.UTF-8/' /etc/locale.conf
```
```
nvim /etc/vconsole.conf
```
```
FONT=lat2-16
FONT_MAP=8859-2
```

### user

#### change /etc/sudoe
```
nvim /etc/sudoers
```
> find ## User privilege specification
```
/## User privilege specification
```
```
root ALL=(ALL:ALL) ALL
http ALL=(ALL:ALL) ALL
```
```
chown -R http:http /srv/http
```

#### change /etc/passwd

```
nvim /etc/passwd
```
> change line http:x:33:33::/srv/http:/usr/bin/nologin

```
http:x:33:33::/srv/http:/usr/bin/bash
```
```
passwd http
```
```
chage -E -1 http 
```
```
su http
```
```
sudo su
```
```
exit
```
```
exit
```

### os release

```
echo '' > /usr/lib/os-release
```
```
nvim /usr/lib/os-release
```
```
NAME="Blackbird"
PRETTY_NAME="Blackbird"
ID=blackbird
BUILD_ID=rolling
ANSI_COLOR="38;2;23;147;209"
HOME_URL="https://blackbird.lektor.co.id/"
DOCUMENTATION_URL="https://blackbird.lektor.co.id/"
SUPPORT_URL="https://blackbird.lektor.co.id/support/"
BUG_REPORT_URL="https://gitlab.blackbird.org/groups/issues"
PRIVACY_POLICY_URL="https://blackbird.lektor.co.id/privacy-policy/"
LOGO=blackbird-logo
```

### package manager

```
nvim /etc/pacman.conf
```
> add TrustedOnly on line SigLevel = Required DatabaseOptional
```
SigLevel = Required DatabaseOptional TrustedOnly
```
uncomment
```
UseSysLog
Color
VerbosePkgLists
```

### apparmor 

```
systemctl enable apparmor.service
```

### pamd
```
nvim /etc/pam.d/passwd
```
```
password required pam_cracklib.so retry=3 minlen=14 difok=3 dcredit=-1 ucredit=-1 ocredit=-1 lcredit=-1
password required pam_unix.so     use_authtok sha512 shadow
```
### secure shell hardening

```
mv /etc/ssh/sshd_config /etc/backup
```
```
nvim /etc/ssh/sshd_config
```
```
# Include drop-in configurations
Include /etc/ssh/sshd_config.d/*.conf

Protocol 2

Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
KexAlgorithms mlkem768x25519-sha256,sntrup761x25519-sha512,curve25519-sha256@libssh.org
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512,hmac-sha2-256

PermitRootLogin no
PermitEmptyPasswords no
LoginGraceTime 20
MaxAuthTries 3
MaxSessions 10
ClientAliveCountMax 3
Banner no

AllowAgentForwarding no
AllowTcpForwarding no
X11Forwarding no

# override default of no subsystems
#Subsystem	sftp	/usr/lib/ssh/sftp-server
```
```
systemctl enable ssh
```

### kernels hardening

```
nvim /etc/sysctl.d/30-secs.conf
```

```
## disable ipv6
net.ipv6.conf.all.disable_ipv6 = 1

# prevent the automatic loading of line disciplines
# https://lore.kernel.org/patchwork/patch/1034150
dev.tty.ldisc_autoload=0


# additional protections for fifos, hardlinks, regular files, and symlinks
# https://patchwork.kernel.org/patch/10244781
# slightly tightened up from the systemd default values of "1" for each
fs.protected_fifos=2
fs.protected_regular=2


## yama ptrac
## https://theprivacyguide1.github.io/linux_hardening_guide
kernel.yama.ptrace_scope=2


# prevents processes from creating new io_uring instances
# https://security.googleblog.com/2023/06/learnings-from-kctf-vrps-42-linux.html
kernel.io_uring_disabled=2


# disable unprivileged user namespaces
# https://lwn.net/Articles/673597
# (these two values are redundant, but not all kernels support the first one)
user.max_user_namespaces=0


# reverse path filtering to prevent some ip spoofing attacks
# (default in some distributions)
net.ipv4.conf.all.rp_filter=1
net.ipv4.conf.default.rp_filter=1


# reverse path filtering to prevent some ip spoofing attacks
# (default in some distributions)
net.ipv4.conf.all.rp_filter=1
net.ipv4.conf.default.rp_filter=1


# disable icmp redirects and RFC1620 shared media redirects
net.ipv4.conf.all.accept_redirects=0
net.ipv4.conf.all.secure_redirects=0
net.ipv4.conf.all.send_redirects=0
net.ipv4.conf.all.shared_media=0
net.ipv4.conf.default.accept_redirects=0
net.ipv4.conf.default.secure_redirects=0
net.ipv4.conf.default.send_redirects=0
net.ipv4.conf.default.shared_media=0
net.ipv6.conf.all.accept_redirects=0
net.ipv6.conf.default.accept_redirects=0


# disable tcp timestamps to avoid leaking some system information
# https://www.whonix.org/wiki/Disable_TCP_and_ICMP_Timestamps
net.ipv4.tcp_timestamps=0

# disable usb
kernel.deny_new_usb=1

#disable coredum
kernel.core_pattern=|/bin/false
```
### module hardening

#### network
```
nvim /etc/modprobe.d/disable-network-protocols.conf
```
```
install dccp /bin/true
install sctp /bin/true
install rds /bin/true
install tipc /bin/true
install n-hdlc /bin/true
install ax25 /bin/true
install netrom /bin/true
install x25 /bin/true
install rose /bin/true
install decnet /bin/true
install econet /bin/true
install af_802154 /bin/true
install ipx /bin/true
install appletalk /bin/true
install psnap /bin/true
install p8023 /bin/true
install p8022 /bin/true
```

#### filesystem
```
nvim /etc/modprobe.d/disable-filesystem-protocols.conf
```
```
install cramfs /bin/true
install freevxfs /bin/true
install jffs2 /bin/true
install hfs /bin/true
install hfsplus /bin/true
install squashfs /bin/true
install udf /bin/true
```

### loging config
```
mkdir -p /etc/systemd/journald.conf.d/
```
```
nvim /etc/systemd/journald.conf.d/01-default.conf
```
```
[Journal]
SystemMaxUse=1G
SystemKeepFree=500M
RuntimeMaxUse=200M
RuntimeKeepFree=50M
MaxFileSec=1month
Storage=persistent
```

### sleep config
```
mkdir -p /etc/systemd/sleep.conf.d/
```
```
nvim /etc/systemd/sleep.conf.d/01-blackbird.conf
```
```
[Sleep]
AllowSuspend=no
AllowHibernation=no
AllowHybridSleep=no
AllowSuspendThenHibernate=no
```

### coredump config
```
nvim /etc/systemd/coredump.conf
```
> comment "[coredump]" adding on last line
```
[Coredump]
Storage=none
ProcessSizeMax=0
```

### login sudoers
```
nvim /etc/sudo.conf
```
> adding on last line
```
## Config Log
Defaults logfile="/var/log/sudo.log"
```
### autoupdate
```
nvim /etc/systemd/system/update.service
```
```
[Unit]
Description=Run system update

[Service]
Type=oneshot
ExecStart=/usr/bin/pacman --sync --refresh --sysupgrade --noconfirm
```
```
nvim /etc/systemd/system/update.timer
```
```
[Unit]
Description=Run the system update daily

[Timer]
OnCalendar=hourly
Persistent=true
Unit=update.service

[Install]
WantedBy=timers.target
```
```
systemctl enable update.timer 
```

### irqbalance
```
systemctl enable irqbalance
```
```
mkdir -p /usr/lib/systemd/system/irqbalance.service.d/
```
```
cat > /usr/lib/systemd/system/irqbalance.service.d/10-no-private-users.conf <<EOF
[Service]
PrivateUsers=false
EOF
```
### tuned
```
systemctl enable tuned
```

### network
```
nvim /etc/systemd/network/20-ethernet.network
```
```
[Network]
Address=[IP]/24
Gateway=10.10.1.1
DNS=1.1.1.1 8.8.8.8
MulticastDNS=yes
```
```
systemctl enable systemd-networkd.socket
```
```
systemctl enable systemd-resolved
```

### boot directory
```
rm /boot/initramfs-linux-*
```
```
mkdir -p /boot/kernel /boot/efi /boot/efi/linux /boot/efi/systemd /boot/efi/rescue /boot/efi/boot
```
```
mv /boot/intel-ucode.img /boot/vmlinuz-linux-* /boot/kernel
```

### kernel parameter

```
mkdir /etc/cmdline.d
```
```
touch /etc/cmdline.d/{01-boot.conf,02-mods.conf,03-secs.conf,04-perf.conf,05-nets.conf,06-misc.conf}
```
#### 01-boot
```
echo "rd.luks.name=$(blkid -s UUID -o value /dev/nvme0n1p3)=proc rd.luks.options=timeout=0s root=/dev/proc/root" > /etc/cmdline.d/01-boot.conf
```

#### 03-secs
```
echo "lsm=landlock,lockdown,yama,integrity,apparmor,bpf lockdown=integrity" > /etc/cmdline.d/03-secs.conf
```

#### 04-perf
```
echo "ipv6.disable=1" > /etc/cmdline.d/04-perf.conf
```

#### 05-nets.conf
```
sudo nvim /etc/cmdline.d/05-nets.conf
```

```
ip=[ip address static]::[ip gateway]:255.255.255.0::eth0:none nameserver=10.10.1.1
```

#### 06-misc

```
echo "rw quiet" > /etc/cmdline.d/06-misc.conf
```

### cryptab
```
echo "data UUID=$(blkid -s UUID -o value /dev/nvme0n1p4) none" >> /etc/crypttab
```


### default.conf for mkinitcpio

#### initram directory

```
rm -fr /etc/mkinitcpio.conf.d
```
```
cp /etc/mkinitcpio.conf /etc/mkinitcpio.conf.bak
```
```
mv /etc/mkinitcpio.conf /etc/mkinitcpio.d/default.conf
```
```
echo '' > /etc/mkinitcpio.d/default.conf
```
```
nvim /etc/mkinitcpio.d/default.conf
```
```
# nvim:set ft=sh:
# MODULES
# The following modules are loaded before any boot hooks are
# run.  Advanced users may wish to specify all system modules
# in this array.  For instance:
#     MODULES=(usbhid xhci_hcd)
MODULES=()

# BINARIES
# This setting includes any additional binaries a given user may
# wish into the CPIO image.  This is run last, so it may be used to
# override the actual binaries included by a given hook
# BINARIES are dependency parsed, so you may safely ignore libraries
BINARIES=()

# FILES
# This setting is similar to BINARIES above, however, files are added
# as-is and are not parsed in any way.  This is useful for config files.
FILES=()

# HOOKS
# This is the most important setting in this file.  The HOOKS control the
# modules and scripts added to the image, and what happens at boot time.
# Order is important, and it is recommended that you do not change the
# order in which HOOKS are added.  Run 'mkinitcpio -H <hook name>' for
# help on a given hook.
# 'base' is _required_ unless you know precisely what you are doing.
# 'udev' is _required_ in order to automatically load modules
# 'filesystems' is _required_ unless you specify your fs modules in MODULES
# Examples:
##   This setup specifies all modules in the MODULES setting above.
##   No RAID, lvm2, or encrypted root is needed.
#    HOOKS=(base)
#
##   This setup will autodetect all modules for your system and should
##   work as a sane default
#    HOOKS=(base udev autodetect microcode modconf block filesystems fsck)
#
##   This setup will generate a 'full' image which supports most systems.
##   No autodetection is done.
#    HOOKS=(base udev microcode modconf block filesystems fsck)
#
##   This setup assembles a mdadm array with an encrypted root file system.
##   Note: See 'mkinitcpio -H mdadm_udev' for more information on RAID devices.
#    HOOKS=(base udev microcode modconf keyboard keymap consolefont block mdadm_udev encrypt filesystems fsck)
#
##   This setup loads an lvm2 volume group.
#    HOOKS=(base udev microcode modconf block lvm2 filesystems fsck)
#
##   This will create a systemd based initramfs which loads an encrypted root filesystem.
#    HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt filesystems fsck)
#
##   NOTE: If you have /usr on a separate partition, you MUST include the
#    usr and fsck hooks.
#HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block filesystems fsck)
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole sd-encrypt lvm2 block filesystems fsck)


# COMPRESSION
# Use this to compress the initramfs image. By default, zstd compression
# is used for Linux ≥ 5.9 and gzip compression is used for Linux < 5.9.
# Use 'cat' to create an uncompressed image.
#COMPRESSION="zstd"
#COMPRESSION="gzip"
#COMPRESSION="bzip2"
#COMPRESSION="lzma"
#COMPRESSION="xz"
#COMPRESSION="lzop"
#COMPRESSION="lz4"

# COMPRESSION_OPTIONS
# Additional options for the compressor
#COMPRESSION_OPTIONS=()

# MODULES_DECOMPRESS
# Decompress loadable kernel modules and their firmware during initramfs
# creation. Switch (yes/no).
# Enable to allow further decreasing image size when using high compression
# (e.g. xz -9e or zstd --long --ultra -22) at the expense of increased RAM usage
# at early boot.
# Note that any compressed files will be placed in the uncompressed early CPIO
# to avoid double compression.
#MODULES_DECOMPRESS="no"
```

#### configure linux preset

```
cp /etc/mkinitcpio.d/linux-hardened.preset /etc/mkinitcpio.d/linux-hardened.preset.bak 
```
```
echo '' > /etc/mkinitcpio.d/linux-hardened.preset
```
```
nvim /etc/mkinitcpio.d/linux-hardened.preset
```
```
# mkinitcpio preset file for the 'linux-hardened' package

ALL_config="/etc/mkinitcpio.d/default.conf"
ALL_kver="/boot/kernel/vmlinuz-linux-hardened"
ALL_kerneldest="/boot/kernel/vmlinuz-linux-hardened"

PRESETS=('default')
#PRESETS=('default' 'fallback')

#default_config="/etc/mkinitcpio.conf"
#default_image="/boot/initramfs-linux-hardened.img"
default_uki="/boot/efi/linux/blackbird-hardened.efi"
#default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"

#fallback_config="/etc/mkinitcpio.conf"
#fallback_image="/boot/initramfs-linux-hardened-fallback.img"
#fallback_uki="/efi/EFI/Linux/arch-linux-hardened-fallback.efi"
#fallback_options="-S autodetect"
```
```
bootctl --path=/boot install
```
```
mkinitcpio -P
```
### instrusion detection

```
cd /tmp
```
```
pkg-config --libs --cflags glib-2.0
```
```
wget https://github.com/aide/aide/releases/download/v0.19.2/aide-0.19.2.tar.gz
```
```
tar xf aide-0.19.2.tar.gz
```
```
cd aide-0.19.2
```
```
./configure --with-zlib --with-posix-acl --with-xattr --with-curl --with-locale --with-syslog-ident --with-config-file=/etc/aide.conf
```
```
make && make install
```
```
nvim /etc/systemd/system/aide.service
```
```
[Unit]
Description=Aide Check
ConditionACPower=true

[Service]
Type=simple
ExecStart=/usr/local/bin/aide --check

[Install]
WantedBy=multi-user.target
```
```
nvim /etc/systemd/system/aide.timer
```
```
[Unit]
Description=Aide check every 8 Hours

[Timer]
OnCalendar=*:0/8:00
Unit=aidecheck.service

[Install]
WantedBy=multi-user.target
```
```
systemctl enable aide.timer
```
```
mkdir -p /var/log/aide
```
```
mkdir -p /var/lib/aide
```
```
touch /var/log/aide/aide.log 
```
```
nvim /etc/aide.conf 
```
```
# Example configuration file for AIDE.
# More information about configuration options available in the aide.conf manpage.
# Inspired from https://src.fedoraproject.org/rpms/aide/raw/rawhide/f/aide.conf

# ┌───────────────────────────────────────────────────────────────┐
# │ CONTENTS OF aide.conf                                         │
# ├───────────────────────────────────────────────────────────────┘
# │
# ├──┐VARIABLES
# │  ├── DATABASE
# │  └── REPORT
# ├──┐RULES
# │  ├── LIST OF ATTRIBUTES
# │  ├── LIST OF CHECKSUMS
# │  └── AVAILABLE RULES
# ├──┐PATHS
# │  ├──┐EXCLUDED
# │  │  ├── ETC
# │  │  ├── USR
# │  │  └── VAR
# │  └──┐INCLUDED
# │     ├── ETC
# │     ├── USR
# │     ├── VAR
# │     └── OTHERS
# │
# └───────────────────────────────────────────────────────────────

# ################################################################ VARIABLES

# ################################ DATABASE

@@define DBDIR /var/lib/aide
@@define LOGDIR /var/log/aide

# The location of the database to be read.
database_in=file:@@{DBDIR}/aide.db.gz

# The location of the database to be written.
#database_out=sql:host:port:database:login_name:passwd:table
#database_out=file:aide.db.new
database_out=file:@@{DBDIR}/aide.db.new.gz

# Whether to gzip the output to database
gzip_dbout=yes

# ################################ REPORT

# Default.
log_level=warning
report_level=changed_attributes

report_url=file:@@{LOGDIR}/aide.log
report_url=stdout
#report_url=stderr
#NOT IMPLEMENTED report_url=mailto:root@foo.com
#NOT IMPLEMENTED report_url=syslog:LOG_AUTH

# ################################################################ RULES

# ################################ LIST OF ATTRIBUTES

# These are the default parameters we can check against.
#p:             permissions
#i:             inode:
#n:             number of links
#u:             user
#g:             group
#s:             size
#b:             block count
#m:             mtime
#a:             atime
#c:             ctime
#S:             check for growing size
#acl:           Access Control Lists
#selinux        SELinux security context (must be enabled at compilation time)
#xattrs:        Extended file attributes

# ################################ LIST OF CHECKSUMS

#md5:           md5 checksum
#sha1:          sha1 checksum
#sha256:        sha256 checksum
#sha512:        sha512 checksum
#rmd160:        rmd160 checksum
#tiger:         tiger checksum
#haval:         haval checksum (MHASH only)
#gost:          gost checksum (MHASH only)
#crc32:         crc32 checksum (MHASH only)
#whirlpool:     whirlpool checksum (MHASH only)

# ################################ AVAILABLE RULES

# These are the default rules
#R:             p+i+l+n+u+g+s+m+c+md5
#L:             p+i+l+n+u+g
#E:             Empty group
#>:             Growing logfile p+l+u+g+i+n+S

# You can create custom rules - my home made rule definition goes like this 
ALLXTRAHASHES = sha1+rmd160+sha256+sha512+whirlpool+tiger+haval+gost+crc32
ALLXTRAHASHES = sha1+rmd160+sha256+sha512+tiger
# Everything but access time (Ie. all changes)
EVERYTHING = R+ALLXTRAHASHES

# Sane, with multiple hashes
# NORMAL = R+rmd160+sha256+whirlpool
# NORMAL = R+sha256+sha512
NORMAL = p+i+l+n+u+g+s+m+c+sha256

# For directories, don't bother doing hashes
DIR = p+i+n+u+g+acl+xattrs

# Access control only
PERMS = p+i+u+g+acl

# Logfile are special, in that they often change
LOG = >

# Just do sha256 and sha512 hashes
FIPSR = p+i+n+u+g+s+m+c+acl+xattrs+sha256
LSPP = FIPSR+sha512

# Some files get updated automatically, so the inode/ctime/mtime change
# but we want to know when the data inside them changes
DATAONLY = p+n+u+g+s+acl+xattrs+sha256

# ################################################################ PATHS

# Next decide what directories/files you want in the database.

# ################################ EXCLUDED

# ################ ETC

# Ignore backup files
!/etc/.*~

# Ignore mtab
!/etc/mtab

# ################ USR

# These are too volatile
!/usr/src
!/usr/tmp

# ################ VAR

# Ignore logs
!/var/lib/pacman/.*
!/var/cache/.*
!/var/log/.*  
!/var/log/aide.log
!/var/run/.*  
!/var/spool/.*

# ################################ INCLUDED

# ################ ETC

# Check only permissions, inode, user and group for /etc, but cover some important files closely.
/etc                               PERMS
/etc/aliases                       FIPSR
/etc/at.allow                      FIPSR
/etc/at.deny                       FIPSR
/etc/audit/                        FIPSR
/etc/bash_completion.d/            NORMAL
/etc/bashrc                        NORMAL
/etc/cron.allow                    FIPSR
/etc/cron.daily/                   FIPSR
/etc/cron.deny                     FIPSR
/etc/cron.d/                       FIPSR
/etc/cron.hourly/                  FIPSR
/etc/cron.monthly/                 FIPSR
/etc/crontab                       FIPSR
/etc/cron.weekly/                  FIPSR
/etc/cups                          FIPSR
/etc/exports                       NORMAL
/etc/fstab                         NORMAL
/etc/group                         NORMAL
/etc/grub/                         FIPSR
/etc/gshadow                       NORMAL
/etc/hosts.allow                   NORMAL
/etc/hosts.deny                    NORMAL
/etc/hosts                         FIPSR
/etc/inittab                       FIPSR
/etc/issue                         FIPSR
/etc/issue.net                     FIPSR
/etc/ld.so.conf                    FIPSR
/etc/libaudit.conf                 FIPSR
/etc/localtime                     FIPSR
/etc/login.defs                    FIPSR
/etc/login.defs                    NORMAL
/etc/logrotate.d                   NORMAL
/etc/modprobe.conf                 FIPSR
/etc/nscd.conf                     NORMAL
/etc/pam.d                         FIPSR
/etc/passwd                        NORMAL
/etc/postfix                       FIPSR
/etc/profile.d/                    NORMAL
/etc/profile                       NORMAL
/etc/rc.d                          FIPSR
/etc/resolv.conf                   DATAONLY
/etc/securetty                     FIPSR
/etc/securetty                     NORMAL
/etc/security                      FIPSR
/etc/security/opasswd              NORMAL
/etc/shadow                        NORMAL
/etc/skel                          NORMAL
/etc/ssh/ssh_config                FIPSR
/etc/ssh/sshd_config               FIPSR
/etc/stunnel                       FIPSR
/etc/sudoers                       NORMAL
/etc/sysconfig                     FIPSR
/etc/sysctl.conf                   FIPSR
/etc/vsftpd.ftpusers               FIPSR
/etc/vsftpd                        FIPSR
/etc/X11/                          NORMAL
/etc/zlogin                        NORMAL
/etc/zlogout                       NORMAL
/etc/zprofile                      NORMAL
/etc/zshrc                         NORMAL

# ################ USR

/usr                               NORMAL
/usr/sbin/stunnel                  FIPSR

# ################ VAR

/var/log/faillog                   FIPSR
/var/log/lastlog                   FIPSR
/var/spool/at                      FIPSR
/var/spool/cron/root               FIPSR

# ################ OTHERS

/boot                              NORMAL
/bin                               NORMAL
/lib                               NORMAL
/lib64                             NORMAL
/opt                               NORMAL
/root                              NORMAL
```
```
aide --init
```
```
mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
```
```
exit
```
```
umount -R /mnt
```
```
reboot
```
