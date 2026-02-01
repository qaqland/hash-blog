# mkdot

mkdot 是一款 dotfiles 安装小工具，用于以较低的精神成本初始化新 Linux 系统。
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

脑海里浮现了一些功能相似的系统组件：`cp`、`ln` 和 `install`，
以下内容来自 BusyBox：

```
$ ln --help
Usage: ln [-sfnbtv] [-S SUF] TARGET... LINK|DIR

Create a link LINK or DIR/TARGET to the specified TARGET(s)

    -s      Make symlinks instead of hardlinks
    -f      Remove existing destinations
    -n      Don't dereference symlinks - treat like normal file
    -b      Make a backup of the target (if exists) before link operation
    -S SUF  Use suffix instead of ~ when making backup files
    -T	Treat LINK as a file, not DIR
    -v	Verbose

$ install --help
Usage: install [-cdDsp] [-o USER] [-g GRP] [-m MODE] [-t DIR] [SOURCE]... DEST

Copy files and set attributes

    -c      Just copy (default)
    -d      Create directories
    -D      Create leading target directories
    -s      Strip symbol table
    -p      Preserve date
    -o USER Set ownership
    -g GRP  Set group ownership
    -m MODE Set permissions
    -t DIR  Install to DIR
```

其中的部分内容对本工具来说多余，比如 install 的权限和用户（直接复制就好），
剩下的部分直接照搬过来：

```
$ mkdot --help

usage: mkdot [-bfsv] [-S SUF] TOPIC... DIR
   or: mkdot [-bfsv] [-S SUF] -t DIR TOPIC...

Install dotfiles to DEST by TOPIC

    -b      Make a backup of the target (if exists) before install operation
    -f      Remove existing destinations
    -s      Make symlinks instead of copy
    -S SUF  Use suffix instead of ~ when making backup files
    -t DIR  Install all SOURCEs into DIR
    -v      Verbose
```

参考知名备份工具 restic，它在工作时有三个参数：数据、动作、对象：

```
$ restic --repo /tmp/backup backup ~/work
```

本文的 DIR 首先是个抽象的位置，一般默认是用户家目录，
但是应当认可接 sudo 后安装到 etc 或者其他位置也 Ok 所以不可省略。

## 实现

首先想到的是 Rust，但是又即刻冷静下来——错误处理、非 UTF-8 路径处理、体积控制等等，
我都不会哈哈！PS：真的有人忍心为功能这么小的工具上 clap 和 thiserror 吗？

Golang 路边一条，使用这门语言写「有理想的」系统小工具属于给未来的自己挖坑。

最适合系统编程的 C 语言涉及到创建文件夹也非常难受，创建文件夹需要一层层的

