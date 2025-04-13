# VSCode Remote SSH 连接 Alpine Linux

2025 年 4 月 11 日

远程连接时的步骤大致如下：

- 安装 remote-cli 到服务端
- remote-cli 下载并启动 vscode-server
- 本地客户端与 vscode-server 连接

## 基础

```bash
$ apk add bash curl git libstdc++ procps-ng
```

如果还是在启动时报错，可以查看 vscode 仓库对应脚本 `resources/server/bin/helpers/check-requirements-linux.sh`

## 插件

// TODO

## References

- <https://github.com/microsoft/vscode>
- <https://github.com/microsoft/vscode-remote-release/issues/6347>
