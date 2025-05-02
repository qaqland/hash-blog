# Setup Aports

2025年4月30日 设置 Aports 本地环境

## Abuild

配置本地 Abuild，遵照官方 [Wiki](https://wiki.alpinelinux.org/wiki/Creating_an_Alpine_package)

```bash
apk add abuild abuild-rootbld alpine-sdk
addgroup xxx abuild
abuild-keygen -a -i
```

在 `~/.abuild/abuild.conf` 可以配置部分设置选项，如颜色、镜像等

## Ssh

官方的 GitLab 注册并配置 Ssh 免密，与常见操作相同

```bash
cd .ssh
ssh-keygen  # 生成 key
cat xxx.pub # 复制到网页里

vim config 
ssh -T git@gitlab.alpinelinux.org
```

## Git

推荐 pull 时从镜像拉，因为官方的仓库又大又慢，以下是本人初始化过程

### Clone

从镜像进行首次 Clone，默认只包括 master 分支

```bash
git config --global user.name qaqland
git config --global user.email "qaq@qaq.land"
git clone https://mirror.nyist.edu.cn/git/aports
cd aports/
git remote -v
```

### Remote

将镜像上游改名到 mirror，设置官方上游到 origin

```bash
git remote add mirror https://mirror.nyist.edu.cn/git/aports
git remote set-url origin git@gitlab.alpinelinux.org:qaqland/aports.git
```

最终效果如下

```bash
$ git remote -v
mirror  https://mirror.nyist.edu.cn/git/aports (fetch)
mirror  https://mirror.nyist.edu.cn/git/aports (push)
origin  git@gitlab.alpinelinux.org:qaqland/aports.git (fetch)
origin  git@gitlab.alpinelinux.org:qaqland/aports.git (push)
```

### Branch

更新 remote 并把 master 绑定到 mirror，因为 master 只用来同步 Pull

```bash
git fetch origin
git fetch mirror

git branch --set-upstream-to=mirror/master
git branch -vv
```

每当自己有新修改时，推送自己的新分支到 origin

```bash
git checkout -b xxx
git commit
git push -u origin xxx
```
