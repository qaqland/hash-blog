# Alpine Linux 软件包仓库一致性检查

2025 年 6 月 6 日

## 问题描述

镜像中共有两种文件，`APKINDEX` 索引文件和具体的 `pkgname-pkgver-rpkgrel.apk` 安装包。当用户执行安装软件操作时，会进行以下过程：

- 下载最新的 APKINDEX 到本地
- 解析索引，拿到对应的具体包名
- 下载对应的软件包文件
- 执行安装、脚本等

如果索引文件的版本与具体软件包不匹配，就会造成安装时找不到软件包，报错信息等同于软件名输入错误，示例如下：

```bash
$ doas apk add cage
ERROR: unable to select packages:
  cage (no such package):
      required by: world[cage]
```

## 当前进展

- 联系上游制作基于本地的一致性检查工具
- 本地测试
- 联系镜像站测试 TODO
- 发现问题后如何解决？

## 使用方法

Go 项目很好编译，调用参数为 `APKINDEX.tar.gz` 所在的目录路径：

```bash
$ ./repo-tools/repo-tools local check test-repo/

# 如果缺失
Package zfs-scripts-2.3.2-r0.apk mentioned in the index not found

# 如果成功无显示 TODO 可能要进程返回值设置为 1
```

## 个人猜想

可以做成两步同步：先单独拉索引文件，生成 rsync 同步 apk 文件列表，再用这个列表做同步，最后检查没问题时，使用新索引文件覆盖掉旧的。

## 引用网址

- <https://gitlab.alpinelinux.org/alpine/infra/repo-tools/-/merge_requests/1>
- <https://salsa.debian.org/mirror-team/archvsync/>
