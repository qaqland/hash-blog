# setup-alpine

为了学习怎么做镜像，把官方的装机初始化脚本梳理一遍。
本文的所有内容都来自 <https://gitlab.alpinelinux.org/alpine/alpine-conf>。

一般进入 LiveCD 系统后会提示调用 `setup-alpine` 安装系统到硬盘。

> PS：这里存在一个小问题，就算安装完成这行提示还是会存在，非常不友好。

它的功能是一个个小模块组合起来的，大致包含关系如下：

```
setup-alpine
  setup-keymap
  setup-hostname
  setup-devd # just to bootstrap
  setup-interfaces
  setup-dns
  setup-timezone
  setup-proxy
  setup-ntp
  setup-apkrepos
  setup-devd
  setup-user
  setup-sshd
  setup-disk
  setup-lbu
  setup-apkcache
```

## setup-keymap

键盘的配置包括 layout 布局和 variant 变体，比如 us 布局下两种变体

```
/usr/share/bkeymaps/us/us-dvorak.bmap.gz    # dvorak
/usr/share/bkeymaps/us/us.bmap.gz           # qwerty*
```

此处配置文件来自 kbd-bkeymaps 包，拿到需要的部分后包就可以扬掉。
busybox-openrc 中的 loadkmap 服务会在 boot 时设置对应的键盘配置。

我们选 us 默认的 qwerty 或者一开始选择 none 就好。

## setup-hostname

主机名必须小于等于 255，且

- must only contain letters (a-z A-Z), digits (0-9), `.` or `-`
- must not start with `-` or `.`

写入 `/etc/hostname` 文件。（不知道为什么此处没有重启反而是后续在做）

```
# rc-service --quiet hostname restart
```

## setup-devd

Set up the device manager.

- mdev (from busybox) is the default.
- mdevd is standalone, compatible with mdev, more efficient.
- udev (from eudev) is the complex, full-featured one.

装机中会设置两次，
第一次是默认的 mdev 用来 bootstrap，
第二次（可能无）随环境变量 `DEVDOPTS`。

如果后续起桌面会被其它脚本覆盖。

## setup-interfaces

默认调用参数是 `-a`（Automatic interface setup using DHCP） 和
`-r`（Restart the networking service after the setup）。

```
set -- $(available_ifaces)  # 将输出设置为函数的位置参数
if [ $# -eq 0 ]; then       # 参数的数量
    return
fi
for iface in "$@"; do       # 所有参数列表
    $MOCK ip link set dev "$iface" up
done
```

不知道为什么接口要从 `ip link` 中读取，没有直接遍历 `/sys/class/net`。

```
# ip link show dev "$iface"
# ip link set dev "$iface" up
# ip link set dev "$iface" down
```

有线网络除了写应该没别的操作了 `/etc/network/interfaces`

```
auto lo
iface lo inet loopback

auto $iface
iface $iface inet dhcp
```

TODO 这里的 up 和 down 对 `rc-service networking restart` 有什么影响，
服务重启经历了什么。

## setup-dns

写入基础 nameserver 到 `/etc/resolv.conf`

TODO 谁负责更新这个文件，文件有什么用

## setup-timezone

时区信息在 `tzdata` 包，结果最后以软链接的形式保存在 `/etc/localtime`。

```
/etc/zoneinfo/
└── Asia
    └── Shanghai
```

## setup-proxy

busybox wget does not handle http proxies well

```
PROFILE="$ROOT/etc/profile.d/proxy.sh"

export http_proxy=$proxyurl
export https_proxy=$proxyurl
export ftp_proxy=$proxyurl
export no_proxy=localhost
```

## setup-ntp

时间同步，可能要看下 hwclock 这个服务在干什么

之前版本默认是 chrony 现在改为了 busybox 实现避免额外装包。
不过我没用出来差异。

## setup-apkrepos

主要被用来添加社区源

```
APKREPOS_PATH="${ROOT}"etc/apk/repositories
```

TODO：ROOT 是在哪里设置的？

## setup-user

adduser 与 addgroup 的调用，无新意。注意 doas 的设置可能覆盖用户设置。

```
if [ -n "$admin" ]; then
	apk add doas
	mkdir -p "$ROOT"/etc/doas.d
	echo "permit persist :wheel" >> "$ROOT"/etc/doas.d/20-wheel.conf
	$MOCK addgroup "$username" "wheel" || exit
fi
```

## setup-sshd

## setup-disk

本文只用考虑 GPT 的 UEFI，虚拟机中可以用环境变量 `USE_EFI=1` 强制指定，
物理机会查看 `/sys/firmware/efi` 路径是否存在。
UEFI 的 bootloader 是 GRUB（实际安装 `grub-efi`）。

```
$devnum=$(mountpoint -d MOUNT_TO_PATH)
cat /sys/dev/block/$devnum/loop/backing_file
```

```
$ df -h
文件系统        大小  已用  可用 已用% 挂载点
udev            7.7G     0  7.7G    0% /dev
tmpfs           1.6G  5.3M  1.6G    1% /run
/dev/nvme0n1p3  218G   11G  197G    5% /
/dev/sda1        92G   21G   66G   24% /persistent
usr-overlay      92G   21G   66G   24% /usr
opt-overlay      92G   21G   66G   24% /opt
etc-overlay      92G   21G   66G   24% /etc
/dev/sda2       825G  9.5G  773G    2% /home
tmpfs           7.8G     0  7.8G    0% /dev/shm
tmpfs           5.0M   12K  5.0M    1% /run/lock
efivarfs        192K   33K  155K   18% /sys/firmware/efi/efivars
tmpfs           7.8G  113M  7.7G    2% /tmp
tmpfs           1.6G  456K  1.6G    1% /run/user/1000
/dev/sdb1        58G   42G   17G   72% /media/anguoli/Ventoy

$ mountpoint -d /media/anguoli/Ventoy/
8:17

$ cat /sys/dev/block/8:17/uevent
MAJOR=8
MINOR=17
DEVNAME=sdb1
DEVTYPE=partition
DISKSEQ=11
PARTN=1
```

`/dev/dm*/` 是软件 RAID：<https://linux.die.net/man/4/md>

```
for i in $(_blkid "$1"); do
    case "$i" in
        UUID=*) eval $i;;
    esac
done
```

`/etc/fstab` 分别写了

- 设备/文件系统标识，`UUID=xxx`、`/dev/sdX`、`tmpfs` 等
- 挂载点，常用 `/` 搭配 `/boot/efi`
- 文件系统类型，有 `ext4`、`vfat` 等
- 挂载选项，`defaults`、`rw`、`nosuid`、`nodev` 等
- 备份标志，很少用写 `0`
- Fsck 检查顺序，根目录写 `1`、其他分区写 `2`、临时文件系统写 `0`

```
case "$ARCH" in
    x86_64)		target=x86_64-efi ; fwa=x64 ;;
    aarch64)	target=arm64-efi ; fwa=aa64 ;;
esac

$MOCK grub-install --target=$target --efi-directory="$efi_directory" \
    --bootloader-id=alpine --boot-directory="$mnt"/boot --no-nvram
$MOCK install -D "$efi_directory"/EFI/alpine/grub$fwa.efi \
    "$efi_directory"/EFI/boot/boot$fwa.efi
```

我自己电脑上这两个文件确实相同，有的系统可能会做其他处理：

```
$ find /boot/efi/EFI/ -type f -exec md5sum {} \;
e753007da5d815751a0e47e95fef91e1  /boot/efi/EFI/alpine/grubx64.efi
e753007da5d815751a0e47e95fef91e1  /boot/efi/EFI/boot/bootx64.efi
```

```
local initfs_features="ata base ide scsi usb virtio"
[ "$ARCH" = "aarch64" ] && initfs_features="$initfs_features phy"
initfs_features="${initfs_features% nvme} nvme"

# setup GRUB config. trigger will generate final grub.cfg
install -d "$mnt"/etc/default/
cat > "$mnt"/etc/default/grub <<- EOF
GRUB_TIMEOUT=2
GRUB_DISABLE_SUBMENU=y
GRUB_DISABLE_RECOVERY=true
GRUB_CMDLINE_LINUX_DEFAULT="modules=$modules $kernel_opts"
EOF

echo "features=\"$initfs_features\"" > "$mnt"/etc/mkinitfs/mkinitfs.conf

modules="sd-mod,usb-storage,${root_fs}${raidmod}"
```

脚本中硬盘的配置使用 `sfdisk` 分区、使用 `mkfs` 格式化

```
case "$BOOTFS" in
    vfat) mkfs_args="-F 32";; # UEFI firmware often only support FAT32
esac
$MOCK mkfs.$BOOTFS $MKFS_OPTS_BOOT $mkfs_args $bootdev
```

```
$MOCK mkfs.$ROOTFS $MKFS_OPTS_ROOT $mkfs_args "$root_dev"
mount -t "$ROOTFS" "$root_dev" "$sysroot" || return 1
if [ -n "$boot_dev" ]; then
    if [ -n "$USE_EFI" ] && [ -z "$USE_CRYPT" ]; then
        mount -t "$BOOTFS" "$boot_dev" "$sysroot"/boot/efi || return 1
    fi
fi
```

## setup-lbu



## setup-apkcache

```
$ ls -l /etc/apk/cache
lrwxrwxrwx    1 root     root            14 Oct 25 11:11 /etc/apk/cache -> /var/cache/apk
```

其余逻辑与 lbu 有关

