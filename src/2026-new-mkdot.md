# New MkDot

MkDot 是一款 dotfiles 安装小工具，用于以较低的精神成本初始化新 Linux 系统。
这个帖子是一个伪实况，记录了小工具的构思及实现过程。

2026 年 1 月 6 日

## 需求

围绕 dotfile 为主题有几个概念我不喜欢，第一个是「管理」，其中 70% 是「同步」
10% 是「安全」。在我这里认为不对所以完全不需要考虑：

* 绝大多数同步依赖 Git，但是我并不觉得 Git 适合保存配置文件
* 配置文件放在 GitHub 不仅会在国内有访问问题，公开配置也放不了密钥

另一个概念还是「管理」，但是更偏向「功能」或者说工具「定位」，经过观察：

* 我的配置最多 200 行，用不着为此阅读 2000 字的文档学习一个似是而非的教程
* 再复杂的 dotfile 管理工具也不可能全自动托管，除非全部底裤交给 Nix

目前我有一个 U 盘，每次装系统手动复制一些文件过去。
因此小工具只需要把配置文件复制到正确的位置就能满足我的日常需求。

## 功能

脑海里浮现了一些功能相似的系统组件：`cp`、`ln` 和 `install`：

```
$ busybox install --help
BusyBox v1.37.0 (2025-12-16 14:19:28 UTC) multi-call binary.

Usage: install [-cdDsp] [-o USER] [-g GRP] [-m MODE] [-t DIR] [SOURCE]... DEST

Copy files and set attributes

	-c	Just copy (default)
	-d	Create directories
	-D	Create leading target directories
	-s	Strip symbol table
	-p	Preserve date
	-o USER	Set ownership
	-g GRP	Set group ownership
	-m MODE	Set permissions
	-t DIR	Install to DIR
```

其中的部分内容对本工具来说多余，比如 install 的权限和用户（直接复制就好），
剩下的部分直接照搬过来：

```
$ mkdot -h
usage: mkdot [-fins] TOPIC... BASE
   or: mkdot [-fins] -t BASE TOPIC...

install dotfiles from TOPIC(s) to BASE

  -f      overwrite existing files (default)
  -i      prompt before overwriting (interactive)
  -n      no overwrite, skip existing files
  -s      create symbolic links instead of copying
  -t BASE specify BASE directory for all TOPICs
```

参考知名备份工具 restic，它在工作时有三个参数：数据、动作、对象：

```
$ restic --repo /tmp/backup backup ~/work
```

本文小工具只有一个安装动作；数据由 CWD 和调用时的 TOPIC 两部分组合而成；
对象 BASE 是我们安装的目标路径，一般默认是用户家目录，
但是应当认可接 sudo 后安装到 etc 或者其他位置也 Ok 所以不可省略。

## 实现

首先想到的是 Rust，但是又即刻冷静下来——错误处理、非 UTF-8 路径处理、体积控制等等，
我都不会哈哈！PS：真的有人忍心为功能这么小的工具上 clap 和 thiserror 吗？

用 C 结合 POSIX 标准库搓搓，当当当当！顺便用 bats 套件糊了测试。

## 安全

搜了一下找到一个看起来信任度、维护度很高的 gocryptfs，文档也很完善

```
# 虽然是 go 写的但源里有
$ doas apk add gocryptfs

$ gocryptfs -speed
gocryptfs v2.6.1; go-fuse [vendored]; 2026-01-15 go1.25.6 linux/amd64
cpu: 13th Gen Intel(R) Core(TM) i5-13420H; with AES-GCM acceleration
AES-GCM-256-OpenSSL             2730.45 MB/s
AES-GCM-256-Go                  5627.85 MB/s    (selected in auto mode)
AES-SIV-512-Go                   624.00 MB/s
XChaCha20-Poly1305-OpenSSL      1398.80 MB/s
XChaCha20-Poly1305-Go           2064.54 MB/s    (selected in auto mode)

# 需要两个文件夹，一个存数据一个当挂载点
$ mkdir cipher plain

# 初始化数据，设置密码
$ gocryptfs -init cipher

# 加载 fuse 模块、解密、挂载
$ doas modprobe fuse
$ gocryptfs cipher/ plain/

# 卸载，因为有 suid 所以不需要 root
$ fusermount -u plain
```

## 结语

似乎可以结束了，但是 fuse 看起来还挺好玩，不知道能不能做点东西。

