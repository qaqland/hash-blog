# Radxa && Alpine Linux

WIP 年前正经学一学怎么做镜像

系统的固件发布在论坛[^1]，教程在文档[^2]

[^1]: <https://forum.radxa.com/t/radxa-dragon-q6a-firmware-snapshot/28886/20>
[^2]: <https://docs.radxa.com/dragon/q6a>

先用官方的镜像测测可用性，刷系统方法与树莓派类似。
准备妥当后开始适配（迁移）系统到 Alpine Linux。

## 官方镜像

```
$ fdisk -l radxa-dragon-q6a_noble_gnome_t7.output_512.img
Found valid GPT with protective MBR; using GPT

Disk radxa-dragon-q6a_noble_gnome_t7.output_512.img: 11665474 sectors, 1600M
Logical sector size: 512
Disk identifier (GUID): 1b8d5935-e2df-49d6-a092-0933b5695f8d
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 11665440

Number  Start (sector)    End (sector)  Size Name
     1           32768           65535 16.0M primary
     2           65536         2162687 1024M primary
     3         2162688        11665440 4640M primary
```

镜像里一共三个分区，挂载出来

```
$ doas kpartx -av radxa-dragon-q6a_noble_gnome_t7.output_512.img 
add map loop0p1 (253:0): 0 32768 linear 7:0 32768
add map loop0p2 (253:1): 0 2097152 linear 7:0 65536
add map loop0p3 (253:2): 0 9502753 linear 7:0 2162688
```

```
$ doas blkid /dev/mapper/loop0p*
/dev/mapper/loop0p1: LABEL="config" UUID="C6C6-480F" TYPE="vfat"
/dev/mapper/loop0p2: LABEL="efi" UUID="C6C6-9801" TYPE="vfat"
/dev/mapper/loop0p3: LABEL="rootfs" UUID="eb668200-3eea-44a1-8a57-5a2429e2d62d" TYPE="ext4"
```

第一个分区在实机看了一眼什么都没有，可以略过

```
dd if=/dev/zero of=custom.img bs=1M count=2048
```

```
$ doas parted custom.img
GNU Parted 3.6
Using /home/qaq/radxa-empty/custom.img
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mklabel gpt
(parted) mkpart primary fat32 1MiB 101MiB
(parted) set 1 esp on
(parted) mkpart primary ext4 101MiB 100%
(parted) print
Model:  (file)
Disk /home/qaq/radxa-empty/custom.img: 2147MB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name     Flags
 1      1049kB  106MB   105MB   fat32        primary  boot, esp
 2      106MB   2146MB  2041MB  ext4         primary

(parted) quit
```

```
$ doas losetup -fP custom.img

$ doas mkfs.vfat -F 32 /dev/loop0p1

$ doas mkfs.ext4 /dev/loop0p2

$ doas losetup -d /dev/loop0
```
