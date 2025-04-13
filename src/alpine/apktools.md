# Apktools

2025 年 4 月 13 日

Alpine Linux 相关的基础使用，主要是系统包管理器

---

## mirror

所有软件包的镜像信息保存在 `/etc/apk/repositories`，系统默认自带有 vi 编辑器，可以尝试修改。

有三种选择推荐：指定版本号、最新稳定版、滚动版，不推荐混合使用可能造成依赖冲突。

```bash
# 国内推荐使用：南阳理工学院开源软件镜像站
# https://mirror.nyist.edu.cn/

# 指定版本号
http://mirror.nyist.edu.cn/alpine/v3.21/main
http://mirror.nyist.edu.cn/alpine/v3.21/community

# 最新稳定版
http://mirror.nyist.edu.cn/alpine/latest-stable/main
http://mirror.nyist.edu.cn/alpine/latest-stable/community

# 滚动版
http://mirror.nyist.edu.cn/alpine/edge/main
http://mirror.nyist.edu.cn/alpine/edge/community
http://mirror.nyist.edu.cn/alpine/edge/testing
```

修改后刷新生效（不刷新也没什么就是）

```bash
$ apk update
$ apk upgrade
```

## cli

已安装软件包信息保存在 `/etc/apk/world` 中，查看此文件了解系统中**显式**安装的软件包。

搜索比较弱鸡，推荐用谷歌、[官方搜包站](https://pkgs.alpinelinux.org/packages)、和本人[自建服务](https://pkgs.qaq.land/)辅助进行

```bash
# 默认
$ apk search helix

# 搜索终端命令
$ apk search cmd:hx

# 搜动态链接库
$ apk search so:libgit2.so
```

安装和卸载软件包

```bash
$ apk add xxx
$ apk del xxx
```

查看此系统中哪些包依赖 xxx

```bash
$ apk info -r xxx
```

查看**已安装**的给定包 xxx 内部包含哪些文件

```bash
$ apk info -L xxx
```

## example

docker

## world

`/etc/apk/world`

```bash
$ apk fix
```
