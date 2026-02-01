# Alpine Linux 软件打包

2024 年 1 月 14 日

软件仓库的总部叫 aports，目前是官方 GitLab 实例上的一个超大 repository，
所有与官方有关的软件包提交修改都基于仓库的流水线，有什么问题可以查阅 Wiki，
也可以前往 oftc.net 上的 `#alpine-devel` 频道（IRC）寻求帮助。

```bash
$ git clone --depth=1 ...
```

据观察基于邮箱的协作好像不太能用，只能注册 GitLab 使用。
找到官方的仓库 fork 到自己名下（就像 GitHub 那样），接下来的大致流程：

- 绑定密钥、设置用户名与用户邮箱
- 克隆自己的 fork 到本地，**耐心等待**
- 为自己需要修改的地方**创建分支**
- 修改（测试）并提交，推送新分支到仓库
- 在网页里自己的仓库页面提交合并请求

Alpine Linux 软件包的配置文件 _APKBUILD_ 与 Arch Linux 的 _PKGBUILD_ 非常相似，
可以偷偷去他们的网站学习打包经验，
但是不要抄袭——发行版不同、工具链不通用、分包策略也不同。
同时也非常推荐 Fedora 家的包，他们相比 Arch 更加严格规范，易于参考。

## 网址

- 包管理器使用 <https://wiki.alpinelinux.org/wiki/Alpine_Package_Keeper>
- 新建软件包 <https://wiki.alpinelinux.org/wiki/Creating_an_Alpine_package>
- 配置文件说明 <https://wiki.alpinelinux.org/wiki/APKBUILD_Reference>
- 打包相关工具 <https://wiki.alpinelinux.org/wiki/Abuild_and_Helpers>
- 与 aports 仓库相关的 [aports/README.md](https://gitlab.alpinelinux.org/alpine/aports/-/blob/master/README.md)
- 代码与提交格式要求 [CODINGSTYLE.md](https://gitlab.alpinelinux.org/alpine/aports/-/blob/master/CODINGSTYLE.md) [COMMITSTYLE.md](https://gitlab.alpinelinux.org/alpine/aports/-/blob/master/COMMITSTYLE.md)
- Arch Linux 的包 <https://archlinux.org/packages/>
- Fedora 的包 <http://src.fedoraproject.org/>

## FAQ

**由于 musl 引起的编译错误如何解决？**

应该都有人遇到过，aports 搜一搜自己也做个补丁

**如何知道软件需要的依赖？**

- 构建依赖：`abuild rootbld` 一个一个添加 dev 包测试
- 运行依赖：打包时会自动识别大部分动态库，手动加上额外的运行时依赖

**编译需要的 gcc 是依赖吗？**

`build-base` 已有，不需要写，`apk info -r build-base` 查看类似基础组件

**网络问题拉不下来构建需要的源码**

使用 _http_ 代理，例如：`https_proxy=http://localhost:7890/ abuild checksum`

**软件包依赖另外一个自己打的本地包**

`abuild rootbld` 时添加 `~/packages/testing/` 到 `~/aports/testing/.rootbld-repositories`

**怎么知道安装包里包含什么文件**

新版 tar 会自动识别压缩格式，执行`tar -vtf file.apk`获得文件列表

**每次下载软件包耗时很久**

1. 开启 `setup-apkcache`
2. `$ mirror=http://mirrors.tuna.tsinghua.edu.cn/alpine abuild rootbld`

最后如果有其他问题可以联系我

