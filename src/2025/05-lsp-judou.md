# JuDou 句读

2025 年 5 月 14 日 记录一下项目想法

## 概述

JuDou 是一个基于 _LSP_ 的代码审阅辅助工具，通过在本地建立**预缓存**，实现在网页端的语义高亮和引用跳转。

- 服务端：资源消耗极低
- 客户端：无需环境配置
- 流水线：自动集成构建

## 名称

“常买旧书的人，有时会遇到一部书，开首加过句读，夹些破句，中途却停了笔：他点不下去了。”

## 技术栈

- Golang
- Vue
- Json-RPC 2.0
- Json

## 盈利

前期贴皮广告，中期 SAAS，后期私有化部署。

直接竞争对手为 GitHub 和 Sourcegraph，潜在包括各类 LLM 分析工具。

## 参考

- <https://github.com/netmute/ctags-lsp>
- <https://github.com/sourcegraph/jsonrpc2>
- <https://microsoft.github.io/language-server-protocol/>
