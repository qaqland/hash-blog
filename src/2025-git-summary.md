# Git Summary

2025-11-12

## Git介绍及初始化

- <https://git-scm.com/>
- <https://git-scm.com/book/zh/v2>
- <https://git-scm.com/docs>
- <https://github.com/git-guides>
- <https://lore.kernel.org/git/>
- <https://training.github.com/downloads/zh_CN/github-git-cheat-sheet/>
- <https://planet.deepin.org/deepin-2022-01-21-git-from-the-bottom-up-git/>

Git是一个分布式（Version Control Software，CVS）版本控制工具，事实上的业界标准。

> 重要提示：Git一定要和SSH搭配用！

### 生成SSH密钥

```shell
# 进入用户的 ssh 目录
$ cd .ssh/

# 本地创建密钥对
$ ssh-keygen
Generating public/private ed25519 key pair.
Enter file in which to save the key (~/.ssh/id_ed25519): key-name   # 输入名字
Enter passphrase for "key-name" (empty for no passphrase):          # 留空
Enter same passphrase again:                                        # 留空
Your identification has been saved in key-name
Your public key has been saved in key-name.pub
The key fingerprint is:
SHA256:cUcRFTTxmKf1hXklQBBrfE8Muk8+AdwSwi9sM7Gc4KA anguoli@anguoli-PC
The key's randomart image is:
+--[ED25519 256]--+
|       .. ++B*Bo.|
|    . . o+ * o O.|
|   . o +.=X + B *|
|  E   . X+.* o =o|
|       .S+. o o .|
|           + .   |
|            +    |
|             .   |
|                 |
+----[SHA256]-----+

# 生成了两个文件
$ ls -l key-*
-r-------- 1 anguoli anguoli 411 11月10日 14:06 key-name
-rw-r--r-- 1 anguoli anguoli 100 11月10日 14:06 key-name.pub

# 公钥可以随意上传、分发
$ cat key-name.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGKl7MdmTVHQPbjJ2jKDzcMlLZre/eaEgUZaC9HcODR1 anguoli@anguoli-PC

# 私钥保存在本地，注意0400权限
$ cat key-name
-----BEGIN OPENSSH PRIVATE KEY-----
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
LZre/eaEgUZaC9HcODR1AAAAEmFuZ3VvbGlAYW5ndW9saS1QQwECAw==
-----END OPENSSH PRIVATE KEY-----
```

### 创建SSH配置

```
# ~/.ssh/config
Host gitlabwh.uniontech.com
    Hostname gitlabwh.uniontech.com
    User git
    IdentityFile ~/.ssh/gerrit

Host gerrit.uniontech.com
    Hostname gerrit.uniontech.com
    Port 29418
    User git
    IdentityFile ~/.ssh/gerrit

Host github.com
    Hostname ssh.github.com
    User git
    Port 443
    ProxyCommand nc -v -x 127.0.0.1:7897 %h %p
    IdentityFile ~/.ssh/github
```

Git使用到的SSH与普通用户字段基本相同

- **Host** 别名/配置名，对SSH有用
- **Hostname** 真实主机地址，IP或者域名
- **IdentityFile** 私钥绝对路径

配置代理推荐使用netcat-openbsd包提供的nc命令，可以加速走SSH的全部Git操作

- **ProxyCommand** 代理
- `-4`/`-6` 指定仅IPv4或IPv6
- `-X [4|5|connect]` SOCKS v.4, SOCKS v.5 and HTTPS proxy，默认5
- `-x address:port`
- `%h` Hostname，`%p` Port

### 测试Git&SSH

```
ssh -T git@Host
```

Disable pseudo-terminal allocation. 禁用伪终端分配，不会交互式Shell会话

```shell
$ ssh -T git@github.com
Connection to ssh.github.com 443 port [tcp/https] succeeded!
Hi qaqland! You've successfully authenticated, but GitHub does not provide shell access.

$ ssh -T gitlabwh.uniontech.com
Welcome to GitLab, @ut006245!

$ ssh -T gerrit.uniontech.com
git@gerrit.uniontech.com: Permission denied (publickey).
```

### Git配置

```
# 默认分支
git config --global init.defaultBranch main

# 用户和邮箱
git config --global user.name "qaqland"
git config --global user.email "qaq@qaq.land"

# 编辑器
git config --global core.editor "vim"

# 快捷别名
git config --global alias.ss status
git config --global alias.can "commit --amend --no-edit --date=now"
```

这些配置都会写入文件`~/.gitconfig`

```
[user]
    name = qaqland
    email = qaq@qaq.land
[init]
    defaultBranch = main

# 配置http/https需要的代理
[http "https://github.com/"]
    proxy = socks5h://127.0.0.1:7897

[alias]
    can = commit --amend --no-edit --date=now
    ss = status
[core]
    editor = vim
```

## Git的基本数据结构

- 标准Git仓库的版本数据保存在仓库的`.git`路径中
- Git命令支持的环境变量`GIT_DIR`应当指向仓库的`.git`路径
- 裸仓库（常见于服务端）只有`.git`路径和内容

### Objects

Git的大部分数据都以Object的形式保存在`.git/objects`路径下，
每个Object都有自己的UUID也叫做Oid，以SHA1或SHA256格式存在，同一仓库不能混合使用，
大部分操作都可以尝试Oid的前缀缩写。

可以通过命令`git cat-file -t OID`检查类型，常见Object种类有：

- CommitObject 提交
- TreeObject 目录
- BlobObject 内容

```shell
$ git init test-backend
已初始化空的 Git 仓库于 /tmp/test-backend/.git/

$ cd test-backend/

$ echo 1 > 1
$ echo 2 > 2

$ git add .

$ git commit -m test
[main（根提交） 084ca84] test
 2 files changed, 2 insertions(+)
 create mode 100644 1
 create mode 100644 2

$ tree .git/objects/
.git/objects/
├── 08
│   └── 4ca8409b7102d53b1b279627cb41ccba5bac98  # commit
├── 0c
│   └── fbf08886fca9a91cb753ec8734c84fcbe52c9f  # blob
├── d0
│   └── 0491fd7e5bb6fa28c517a0bb32b8b506539d4d  # blob
├── de
│   └── 0ea882503cdd9c984c0a43238014569a123cac  # tree
├── info
└── pack

7 directories, 4 files
```

### Loose & Packed

Git有两种Objects的保存后端，分别是Loose松散后端和Packed紧实后端。

如上文所示以Oid的前2位为前缀创建目录树的行为就是Loose松散后端。
松散后端由本地提交产生，其中对象仅以zlib算法压缩，占用空间较大，但写入速度快。

本地执行`git gc`会repack这些对象到紧实后端

```shell
$ git gc
枚举对象中: 4, 完成.
对象计数中: 100% (4/4), 完成.
使用 8 个线程进行压缩
压缩对象中: 100% (2/2), 完成.
写入对象中: 100% (4/4), 完成.
总共 4（差异 0），复用 0（差异 0），包复用 0（来自  0 个包）

$ tree .git/objects/
.git/objects/
├── info
│   ├── commit-graph
│   └── packs
└── pack
    ├── pack-5468214027a9484a198c7f3c5a6df15f12f48d9d.idx
    ├── pack-5468214027a9484a198c7f3c5a6df15f12f48d9d.pack
    └── pack-5468214027a9484a198c7f3c5a6df15f12f48d9d.rev

3 directories, 5 files
```

为了节省带宽、减小请求，通常在网络交互时也会repack（传输的部分）

```shell
# 文件协议，也算传输，有repack
git clone file:///tmp/test-backend/ test-clone-file

# 路径克隆，直接复制，没有repack
git clone /tmp/test-backend/ test-clone-path
```

先看Loose，我们写个小脚本查看实际保存的Object

```python
#!/usr/bin/env python3

import zlib, sys

data = open(sys.argv[1], "rb").read() if len(sys.argv) > 1 else sys.stdin.buffer.read()
result = zlib.decompress(data)
sys.stdout.buffer.write(result)
```

### CommitObject

提交Commit指向自己的历史来源，有0个、1个、2个或者更多个Parent Commit。
不同的数量表示了这次提交的不同种类，是root节点还是fast-forward这种单线链表，
或者是合并分支的Merge节点。超过2的情况不多见，是特殊的合并节点，比如内核里有：

> Merge branches 'arch-alpha', 'arch-arm', 'arch-arm64', 'arch-avr32', … · torvalds/linux@9b25d60
>
> <https://github.com/torvalds/linux/commit/9b25d604182169a08b206306b312d2df26b5f502>

> A commit object may have any number of parents. With exactly one parent, it is
> an ordinary commit. Having more than one parent makes the commit a merge
> between several lines of history. Initial (root) commits have no parents.
>
> [Git - git-commit-tree Documentation](https://git-scm.com/docs/git-commit-tree)

当然像提交信息、提交人、提交时间、以及Committer与Author之间的区别这里不再赘述。
Commit还保存了提交时的文件（树）快照，指向当前Commit相随的Tree。

- TreeOid
- Parents' CommitOid
- Author
- Committer
- Commit Message

这就是一个Commit对象包含的全部，当一个普通线性提交发生时，
Git会扫描当前WorkTree生成TreeOid，底层保存的数据中并不关心此次提交的修改。

```shell
$ git cat-file -p 91520a2890c9cd9e99bf6cf0148811c33ffe5a3b
tree f93e3a1a1525fb5b91020da86e44810c87a2d7bc
author qaqland <qaq@qaq.land> 1762827659 +0800
committer qaqland <qaq@qaq.land> 1762827659 +0800

add readme

$ git cat-file -t 91520a2890c9cd9e99bf6cf0148811c33ffe5a3b
commit

$ git cat-file -s 91520a2890c9cd9e99bf6cf0148811c33ffe5a3b
155

$ git cat-file commit 91520a2890c9cd9e99bf6cf0148811c33ffe5a3b | wc -c
155
```

以这次提交为例，把原始文件zlib解压后16进制打印输出如下

```shell
$ ./zlib-cat.py .git/objects/91/520a2890c9cd9e99bf6cf0148811c33ffe5a3b | xxd
00000000: 636f 6d6d 6974 2031 3535 0074 7265 6520  commit 155.tree 
00000010: 6639 3365 3361 3161 3135 3235 6662 3562  f93e3a1a1525fb5b
00000020: 3931 3032 3064 6138 3665 3434 3831 3063  91020da86e44810c
00000030: 3837 6132 6437 6263 0a61 7574 686f 7220  87a2d7bc.author 
00000040: 7161 716c 616e 6420 3c71 6171 4071 6171  qaqland <qaq@qaq
00000050: 2e6c 616e 643e 2031 3736 3238 3237 3635  .land> 176282765
00000060: 3920 2b30 3830 300a 636f 6d6d 6974 7465  9 +0800.committe
00000070: 7220 7161 716c 616e 6420 3c71 6171 4071  r qaqland <qaq@q
00000080: 6171 2e6c 616e 643e 2031 3736 3238 3237  aq.land> 1762827
00000090: 3635 3920 2b30 3830 300a 0a61 6464 2072  659 +0800..add r
000000a0: 6561 646d 650a                           eadme.
```

起始`commit`和`155`表示类型和大小（不包括头部），和前文的`cat-file -t|-s`对应

|  HEX |   Description  |
|------|----------------|
| `00` | Null character |
| `20` | Space          |
| `0a` | Line Feed      |
| `30` | Zero           |

### TreeObject

```shell
$ git cat-file -p f93e3a1a1525fb5b91020da86e44810c87a2d7bc
100644 blob e69de29bb2d1d6434b8b29ae775ad8c2e48c5391    README.md

$ git cat-file -t f93e3a1a1525fb5b91020da86e44810c87a2d7bc
tree

$ git cat-file -s f93e3a1a1525fb5b91020da86e44810c87a2d7bc
37

$ git cat-file tree f93e3a1a1525fb5b91020da86e44810c87a2d7bc | wc -c
37

$ git cat-file tree f93e3a1a1525fb5b91020da86e44810c87a2d7bc | xxd
00000000: 3130 3036 3434 2052 4541 444d 452e 6d64  100644 README.md
00000010: 00e6 9de2 9bb2 d1d6 434b 8b29 ae77 5ad8  ........CK.).wZ.
00000020: c2e4 8c53 91
```

一个TreeObject保存任意数量的TreeObject和BlobObject，保存信息有子目录或文件的
权限（条目类型）、条目名称、条目Oid。

```shell
$ ./zlib-cat.py .git/objects/f9/3e3a1a1525fb5b91020da86e44810c87a2d7bc | xxd
00000000: 7472 6565 2033 3700 3130 3036 3434 2052  tree 37.100644 R
00000010: 4541 444d 452e 6d64 00e6 9de2 9bb2 d1d6  EADME.md........
00000020: 434b 8b29 ae77 5ad8 c2e4 8c53 91         CK.).wZ....S.
```

仓库的Oid长度是统一且固定的，如果采用SHA1就是40个十六进制字符，SHA256拓展到64个。

在TreeObject对象中，条目是按照路径排序的，TreeObject条目会在排序时默认带上尾随斜杠

> The entries in a tree are ordered in the _path_ order, which means that a
> directory entry is ordered by adding a slash to the end of it.
>
> So a directory called "a" is ordered _after_ a file called "a.c",
> because "a/" sorts after "a.c".
>
> [git/fsck.c at 6074a7d4ae6b658c18465f10bbbf144882d2d4b0 · git/git](https://github.com/git/git/blob/6074a7d4ae6b658c18465f10bbbf144882d2d4b0/fsck.c#L497)

```shell
$ git ls-tree 00a741484baadea211c493ebbf5fb00208f86493
100644 blob 85d4df20ce3cce7d9bf31f98ee2239683fdc776e	.editorconfig
100644 blob 0493cf8daa1629bdba77e9bdde6106ff9783fc50	.gitattributes
040000 tree 1297f3467c63c4ff48a98fd2a24d747c68aa3f80	.githooks
040000 tree b79b5cc40348d01f293bd9cbf8483cc077459c38	.github
100644 blob 43fb11d16d0008c3314eeab288f49b6189d680dd	.gitignore
100644 blob 70d4ae27a4b250d03eaded0509f86347e6192c42	.gitlab-ci.yml
040000 tree aefeddacabdf8150f47faf3aabd098c5c7c32440	.gitlab
100644 blob b7bf3d42eb30206489f669629bd95c8fcb2d2ee6	.mailmap
100644 blob 8d11373a95ba9d87c8a51193c37d5aa02e2dd301	CODINGSTYLE.md
100644 blob ab8c36d86bcf6eaaff348f1c24bf6594bdcabdfb	COMMITSTYLE.md
100644 blob 27d11c0186c4e846c5d5f64af7121cadbb8d785f	README.md
040000 tree 42aae92dacdae1a5d53531d23ddbac8aeabd13e8	community
040000 tree 0cc9f0ba0475a8302206e431ca51319ce21d6b54	main
040000 tree dcaedd687a45f21b59296a063787a5ee15385716	scripts
040000 tree 8f5eebf9181a091b171b123f6377108560f5ecdf	testing
```

### BlobObject

没有任何技巧和优化的：类型、大小、内容，不保存自己的名称

假如文件内容为`Hello World\n`：

```shell
# 文件末尾有换行
$ hexdump -C hello
00000000  48 65 6c 6c 6f 20 57 6f  72 6c 64 0a              |Hello World.|
0000000c

# 文件长度为12
$ wc -c hello
12 hello
```

可以计算得到此时的sha1sum

```shell
# 类型 + 空格 + 长度 + NUL + 内容
$ printf "blob 12\0Hello World\n" | sha1sum
557db03de997c86a4a028e1ebd3a1ceb225be238  -
```

添加文件并提交后可以看到git给出的BlobOid与我们手动计算的相同

```shell
$ tree .git/objects/
.git/objects/
├── 11
│   └── 7c62a8c5e01758bd284126a6af69deab9dbbe2
├── 55
│   └── 7db03de997c86a4a028e1ebd3a1ceb225be238  <<< 这里
├── f7
│   └── a2bdf7b9df15cdbc88907855a2f55170839af8
├── info
└── pack

6 directories, 3 files
```

### Reference

引用（Reference）类似C语言的指针，
分为直接引用（指向具体的Oid）和符号引用（指向其他引用）两种，
保存在`.git/*HEAD`、`.git/packed-refs`和`.git/refs/`路径下。

直接引用（Direct Reference）存储的是完整的Oid，
如果指向的提交被删除或修改，直接引用可能失效（成为悬空引用），
如果直接引用被删除，可能导致指向的一系列对象变为垃圾对象（分离状态）。
常见的直接引用有：

- `refs/heads/main`分支
- `refs/tags/v1.0`标签
- `refs/remotes/origin/main`远程分支

符号引用（Symbolic Reference）本身不指向具体对象，而是指向另一个引用，相当于间接指针。
最常见的`HEAD`就是符号引用，指向当前分支（如`refs/heads/main`）

```shell
$ cat .git/HEAD
ref: refs/heads/main

$ cat .git/refs/heads/main
f7a2bdf7b9df15cdbc88907855a2f55170839af8

$ git log -1
commit f7a2bdf7b9df15cdbc88907855a2f55170839af8 (HEAD -> main)
Author: qaqland <qaq@qaq.land>
Date:   Tue Nov 11 13:55:14 2025 +0800

    hello
```

// HEAD @ ~ ^

## Git的日常使用方法

老生常谈的“工作区”、“暂存区”、“版本库”三个概念，他们是为了确保提交的原子性（完整性）。
类比到Wayland：

- Git：`git commit`操作是原子性的。
要么整个暂存区的内容全部成功提交，创建一个新的版本记录；要么失败，版本库保持原样。
永远不会看到一个“只提交了一半文件”的版本库状态。
- Wayland：缓冲区交换操作也是原子性的。
在垂直同步信号到来时，系统会瞬间将指向屏幕的指针从“前缓冲区”切换到“完成后缓冲区”。
用户永远不会看到一帧“画了一半”的图像。

这个“原子性”确保了从一个状态（上次提交的版本/上一帧画面）平滑过渡到下一个状态（新的提交/新的一帧画面）。

### git-log

```shell
# 显示提交图
$ git log --oneline --graph --all

# 显示提交代码
$ git log -p

# 添加路径筛选
$ git log -- PATH

# 限制数量
$ git log -n NUM
$ git log -5

# 指定分支/标签/提交的历史
$ git log HEAD
```

如果希望查看当前其他分支的最新提交

```shell
$ git branch -v
  bump-py3-pytest-asyncio-1.2.0 072086e312b community/py3-pytest-asyncio: upgrade to 1.2.0
  lazydocker                    729c43ddb8b community/lazydocker: add runtime depends ncurses
* master                        00c9c05d721 [ahead 186] main/{kea,pgpool}: rebuild against postgresql 18
  new-linyaps-box               9fc79988ded testing/linyaps-box: new aport
  new-microsocks                29e2abdbc99 testing/microsocks: new aport
```

### git-diff

对比差异，观察修改。下面举例说明顺位规律，先创建两次提交，针对同一文件做修改：

```diff
# 反序输出，从上到下
$ git log --oneline --reverse -p
221152d 111                     <<< 第一次提交
diff --git a/hello b/hello
new file mode 100644
index 0000000..2e3e313
--- /dev/null
+++ b/hello
@@ -0,0 +1 @@
+第一次增加

c95a70b (HEAD -> main) 222      <<< 第二次提交
diff --git a/hello b/hello
index 2e3e313..6c27065 100644
--- a/hello
+++ b/hello
@@ -1 +1,2 @@
 第一次增加
+第二次增加
```

`git diff`期待两个位置参数，旧提交在前，新提交在后。

```diff
$ git diff 221152d c95a70b
diff --git a/hello b/hello
index 2e3e313..6c27065 100644
--- a/hello
+++ b/hello
@@ -1 +1,2 @@
 第一次增加
+第二次增加
```

接下来对文件进行修改，添加修改进暂存区，再次修改，结果如下：

```shell
$ git blame -s hello
^221152d 1) 第一次增加
c95a70bf 2) 第二次增加
00000000 3) 暂存区增加（未提交）
00000000 4) 工作区增加
```

工作区 vs 暂存区

```diff
$ git diff
diff --git a/hello b/hello
index e364217..9332b3e 100644
--- a/hello
+++ b/hello
@@ -1,3 +1,4 @@
 第一次增加
 第二次增加
 暂存区增加（未提交）
+工作区增加
```

暂存区 vs HEAD

```diff
$ git diff --staged
$ git diff --cached
diff --git a/hello b/hello
index 6c27065..e364217 100644
--- a/hello
+++ b/hello
@@ -1,2 +1,3 @@
 第一次增加
 第二次增加
+暂存区增加（未提交）
```

工作区 vs HEAD

```diff
$ git diff HEAD
$ git diff HEAD
diff --git a/hello b/hello
index 6c27065..9332b3e 100644
--- a/hello
+++ b/hello
@@ -1,2 +1,4 @@
 第一次增加
 第二次增加
+暂存区增加（未提交）
+工作区增加
```

生成补丁的格式为Unified Diff Format，一种标准化的补丁格式，被广泛用于软件开发和版本控制系统。

```diff
$ git diff
diff --git a/hello b/hello
index e364217..9332b3e 100644
--- a/hello
+++ b/hello
@@ -1,3 +1,4 @@
 第一次增加
 第二次增加
 暂存区增加（未提交）
+工作区增加
```

对于补丁来说，前两行算注释可以删掉，也可以手动增加一点描述信息。
重要的部分从`@@`行开始，这是变更块头（chunk header）描述了修改发生的上下文：

```
@@ -旧版本起始行,旧版本行数 +新版本起始行,新版本行数 @@
```

如果需要手动修改生成的补丁，注意补丁显示的代码行数要和变更块头的描述对应：

```diff
这是注释部分，可以随意写。以下两条命令生成的补丁上下文范围不同，但效果相同

$ git diff -U2                  <<< 指定上下文范围2
diff --git a/hello b/hello
index e364217..9332b3e 100644
--- a/hello
+++ b/hello
@@ -2,2 +2,3 @@                 <<<
 第二次增加
 暂存区增加（未提交）
+工作区增加

$ git diff -U3                  <<< 指定上下文范围3
diff --git a/hello b/hello
index e364217..9332b3e 100644
--- a/hello
+++ b/hello
@@ -1,3 +1,4 @@                 <<<
 第一次增加
 第二次增加
 暂存区增加（未提交）
+工作区增加
```

### git-email

用的不多，推荐教程

> Learn to use email with git!
>
> <https://git-send-email.io/>

### git-reset

软重置，将之前的提交"取消"，所有修改保留在暂存区

```shell
$ git reset --soft

Before: A - B - C (HEAD)
After:  A - B (HEAD)

          C 的修改在暂存区
```

默认混合重置，重置暂存区，修改保留在工作区

```shell
$ git reset [--mixed]

Before: A - B - C (HEAD)
After:  A - B (HEAD)

          C 的修改在工作区（未暂存）
```

硬重置，丢弃所有未提交的修改，完全回退到指定提交的状态

```shell
$ git reset --hard

Before: A - B - C (HEAD) + 工作区修改
After:  A - B (HEAD)  # 完全回到B的状态，所有修改丢失
```

硬重置很危险，如果误删可以尝试恢复：

```shell
# 查看所有操作历史
$ git reflog

# 找到重置前的提交哈希
$ git reset --hard HEAD@{1}  # 恢复到前一个状态
```

恢复历史将会在`git gc`后清空。

### git-rebase

更新当前分支到主分支最新状态

```shell
# 初始状态
A - B - C (main)
     \
      D - E (feature)

$ git checkout feature
$ git rebase main

# 变基后状态
A - B - C (main)
           \
            D' - E' (feature)
```

交互式操作提交

```shell
# 不包含起点
$ git rebase -i 基准OID

# 修改最后5个提交
$ git rebase -i HEAD~5

# 把其中的pick改为edit
$ git add
$ git commit --amend
$ git rebase --continue
```

注意rebase会修改基准提交之后所有提交的Oid

### git-bundle

如何把本地的仓库备份或发送给别人

```shell
$ git clone --mirror https://github.com/alpinelinux/apk-tools.git
$ cd apk-tools
```

```shell
$ git bundle create apk-tools.bundle --all
```

生成的bundle文件就是一个完整仓库的打包，企业微信或者U盘发送后，再clone出来就行

```shell
$ du -sh apk-tools.bundle
4.1M    apk-tools.bundle
```

```shell
$ git clone --bare apk-tools.bundle <new directory>
```

如果都是自己的电脑有SSH，直接clone就行

```shell
$ git clone n5105:~/dotfiles
正克隆到 'dotfiles'...
remote: Enumerating objects: 23, done.
remote: Counting objects: 100% (23/23), done.
remote: Compressing objects: 100% (19/19), done.
remote: Total 23 (delta 4), reused 0 (delta 0), pack-reused 0 (from 0)
接收对象中: 100% (23/23), 完成.
处理 delta 中: 100% (4/4), 完成.
```

## Git的内部优化算法

理论上讲Git中有三个大类别的对象：Commit、Tree、Blob，具体到解析时还有Commit的同类Tag及Note。
这些对象以各自Object的hash作为Oid索引，经zlib压缩后保存在`.git/objects`下的文件中，
纯正文件系统驱动，并在需要时被解析。

Git 的数据结构为写多读少设计，因此其他程序应避免将 Git 作为数据库使用。

### CommitGraph

Git中的每次提交都会创建对应的CommitObject，但是当需要遍历仓库历史的时候，
大量读就成了一个问题。
Git并没有在Commit中描述这次提交修改了哪些文件，所以若要知晓给定文件的最后修改日期，过程较为艰难：

1. 获取Commit的TreeOid与Parent Commit的TreeOid
2. 在两个Tree Object中遍历，查找修改的文件
3. 检查给定文件是否在本次Commit中修改
4. 重复上述操作，直到两次提交的树之间存在期望差异

Git内部在第二步和第三步之间有Diff优化，仅对比给定的路径，
但整体在没有对应数据结构的情况下进行类似的结构化查询还是相当消耗性能，
面对稍微大一点的仓库，时间来到秒级：

```shell
$ /usr/bin/time git -c core.commitGraph=false log --oneline -n 10 community/xmake/
81380060446 community/xmake: upgrade to 2.9.9
7c21bea5624 */*: replace non-POSIX find -not option
a92fe0ba060 community/xmake: upgrade to 2.9.7
12374870a8f community/xmake: upgrade to 2.9.6
12c188ec967 community/xmake: upgrade to 2.9.5
0096aeef07c community/xmake: upgrade to 2.9.4
4b130eb7f5a community/xmake: upgrade to 2.9.3
d516ffbb476 community/xmake: upgrade to 2.9.2
d74b311e776 community/xmake: move from testing
real    0m 51.11s
user    0m 41.44s
sys     0m 9.56s
```

Git在2.18版本后引入了提交图（CommitGraph）的概念，保存提交相关的额外索引信息到
`.git/objects/info/commit-graph`。
针对每个对象，使用独立的布隆过滤器对提交修改做缓存，理想情况下，最快的查询条件为：

```shell
# 激进垃圾回收并repack
$ git gc --aggressive

# 创建带有路径修改信息的提交图索引
$ git commit-graph write --changed-paths
```

经过上述修改性能提升明显：

```shell
$ /usr/bin/time git -c core.commitGraph=true log --oneline -n 10 community/xmake/
81380060446 community/xmake: upgrade to 2.9.9
7c21bea5624 */*: replace non-POSIX find -not option
a92fe0ba060 community/xmake: upgrade to 2.9.7
12374870a8f community/xmake: upgrade to 2.9.6
12c188ec967 community/xmake: upgrade to 2.9.5
0096aeef07c community/xmake: upgrade to 2.9.4
4b130eb7f5a community/xmake: upgrade to 2.9.3
d516ffbb476 community/xmake: upgrade to 2.9.2
d74b311e776 community/xmake: move from testing
real    0m 1.99s
user    0m 1.17s
sys     0m 0.81s
```

扩展阅读：

- [Git - git-commit-graph Documentation](https://git-scm.com/docs/git-commit-graph)
- [Supercharging the Git Commit Graph - Azure DevOps Blog](https://devblogs.microsoft.com/devops/supercharging-the-git-commit-graph/)
- [Updates to the Git Commit Graph Feature - Azure DevOps Blog](https://devblogs.microsoft.com/devops/updates-to-the-git-commit-graph-feature/)

### Packfile

算法部分暂时略，核心思想就是差异压缩：

- 所有对象按照Oid顺序保存方便mmap与二分
- 在文件开头创建范围索引
- 对象之间按照窗口期查找“基对象”进行差异压缩
- 多个Packfile之间创建Oid索引

经过差异压缩，体积可实现有20倍的缩小。接下来进行演示

读一点随机数模拟文件内容

```shell
$ head -c 500 /dev/urandom | base64 | head -n 50 > random_output.txt
```

在这个文件的基础上创建文件2

```shell
$ cp random_output.txt random_output_2.txt
$ echo "add new line" >> random_output_2.txt
```

直接添加提交，Git不会主动repack压缩，此时是原始Loose Objects存储

```shell
$ ls -lh
总计 8.0K
-rw-rw-r-- 1 anguoli anguoli 690 11月11日 22:12 random_output_2.txt
-rw-rw-r-- 1 anguoli anguoli 677 11月11日 22:09 random_output.txt

$ tree -h .git/objects/
[ 160]  .git/objects/
├── [  60]  02
│   └── [ 556]  3dc71e0501e56c41bc7a2b6236f167c7094360  <<<
├── [  60]  06
│   └── [ 112]  ffbb31a44b95173b16baa580941ce54984cb04
├── [  60]  c3
│   └── [ 567]  c41712bdc2d4c2de08aee19d8f3e319fb82e83  <<<
├── [  60]  fd
│   └── [  89]  040d11c4b1b4af03ac409dbefbf6fb94c50b1c
├── [  40]  info
└── [  40]  pack

7 directories, 4 files

$ git cat-file -s 023dc71e
677
```

通过手动gc达成repack压缩

```shell
$ git gc
枚举对象中: 4, 完成.
对象计数中: 100% (4/4), 完成.
使用 8 个线程进行压缩
压缩对象中: 100% (3/3), 完成.
写入对象中: 100% (4/4), 完成.
总共 4（差异 1），复用 4（差异 1），包复用 0（来自  0 个包）

$ tree -h .git/objects/
[  80]  .git/objects/
├── [  80]  info
│   ├── [1.1K]  commit-graph
│   └── [  54]  packs
└── [ 100]  pack
    ├── [1.2K]  pack-6184e7108e795f0abc08d03baa4d2b49cd6d5d80.idx
    ├── [ 805]  pack-6184e7108e795f0abc08d03baa4d2b49cd6d5d80.pack  <<<
    └── [  68]  pack-6184e7108e795f0abc08d03baa4d2b49cd6d5d80.rev

3 directories, 5 files

$ git verify-pack -v .git/objects/pack/pack-6184e7108e795f0abc08d03baa4d2b49cd6d5d80.pack
06ffbb31a44b95173b16baa580941ce54984cb04 commit 144 107 12
023dc71e0501e56c41bc7a2b6236f167c7094360 blob   677 549 119
c3c41712bdc2d4c2de08aee19d8f3e319fb82e83 blob   21 33 668 1 023dc71e0501e56c41bc7a2b6236f167c7094360
fd040d11c4b1b4af03ac409dbefbf6fb94c50b1c tree   92 84 701
非 delta：3 个对象
链长 = 1: 1 对象
.git/objects/pack/pack-6184e7108e795f0abc08d03baa4d2b49cd6d5d80.pack: ok
```

最后分别显示的是

```
<unpack-size> <size-in-packfile> <offset-in-packfile>
```

## Git周边生态及开发

### 代码托管

- <https://github.com>
- <https://about.gitlab.com/>
- <https://gitee.com/> 用了很多前者的基建
- <https://gogs.io/> CVE修得少，不推荐
- <https://about.gitea.com/> MIT，前者的fork
- <https://forgejo.org/> GPL，前者的fork
- <https://sourcehut.org/> Old School
- <https://git.zx2c4.com/cgit/about/> 只读网页
- <https://github.com/sitaramc/gitolite> 无界面
- <https://www.gerritcodereview.com/> 补丁审阅

其他链接：

- <https://en.wikipedia.org/wiki/Comparison_of_source-code-hosting_facilities>
- <https://en.wikipedia.org/wiki/Forge_(software)#Examples>
- <https://onedev.io/> Java，v2ex有创始人
- <https://pagure.io/> Fedora家的
- <https://gerrit.googlesource.com/gitiles> Gerrit搭档
- <https://radicle.xyz/> Web3分布式
- <https://forge.lindenii.org/>
- <https://github.com/yuki-kimoto/gitprep>
- <https://github.com/gitblit-org/gitblit>
- <https://gitlist.org/> PHP只读
- <https://github.com/PGYER/codefever> 蒲公英家的
- <https://github.com/oxalorg/stagit> 静态页面

当从GitHub拉仓库的时候，如果选择SSH则地址如下：

```
>>> git@github.com:user/repo.git
```

约等于SSH中`git`用户在操作`github.com`这个**Host**的`user/repo.git`路径下的仓库。

对个人来说最简单的情况不需要考虑权限，就是直接用SSH地址。

### 鉴权 & 认证

// 服务端与本地如何交互

// SSH认证

// Hooks鉴权

### ChangeId

### 开源项目

- <https://github.com/libgit2/libgit2> Pure C
  - <https://github.com/libgit2/libgit2sharp> C#
  - <https://github.com/rust-lang/git2-rs> Rust
  - <https://github.com/libgit2/rugged> Ruby
  - <https://github.com/libgit2/pygit2> Python
- <https://github.com/go-git/go-git> Pure Go
- <https://github.com/GitoxideLabs/gitoxide> Pure Rust
- <https://github.com/FredrikNoren/ungit> Pure JavaScript
- <https://gitlab.com/gitlab-org/gitaly/> Git RPC

