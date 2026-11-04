---
title: 常用 MCP
tags:
  - ai
  - mcp
  - 工具
---

# 常用 MCP

## 总览

| 名称 | 用途 | 入口 |
| --- | --- | --- |
| GitNexus | 分析 / 索引代码库，生成可查询的代码知识图谱 | `npx gitnexus analyze` |
| chrome-devtools | 让 agent 控制并检查真实 Chrome：抓网络、截图、性能 trace、控制台报错 | `chrome-devtools-mcp` |

## 一、GitNexus

- 仓库：https://github.com/abhigyanpatwari/GitNexus
- 常用命令：

  ```bash
  npx gitnexus analyze
  ```

  对当前仓库做分析 / 建索引，之后可以用 GitNexus 的 skills 查询代码结构、调用链和改动影响面。

## 二、chrome-devtools

### 安装

```bash
npm install -g chrome-devtools-mcp
```

也可以直接让 Claude 自己安装并配置。

### 默认行为

`chrome-devtools-mcp` 默认会**新开一个浏览器实例**，与日常在用的 Chrome 相互独立。若想直接操作用户正在使用的 Chrome（保留登录态、扩展、标签页），见下面两种方式。

### 方式一：连接已开启调试端口的 Chrome

1. 以调试模式启动 Chrome：

   ```powershell
   "C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-port=9222 --user-data-dir=D:\ChromeAI
   ```

   或者复制一个 Chrome 快捷方式，在「目标」末尾追加：

   ```
   --remote-debugging-port=9222 --user-data-dir=D:\ChromeAI
   ```

   之后双击该快捷方式启动即可。

2. 在 MCP 配置里指向该端口：

   ```json
   {
     "mcpServers": {
       "chrome-devtools": {
         "command": "chrome-devtools-mcp",
         "args": [
           "--browserUrl", "http://127.0.0.1:9222",
           "--workspace=D:/Documents"
         ],
         "type": "stdio"
       }
     }
   }
   ```

> [!note] `--autoConnect` 与 `--browserUrl`
> 原记录提到“加上 `--autoConnect`”，但示例配置里实际用的是 `--browserUrl http://127.0.0.1:9222`（连到已开的调试端口）。以示例为准，`--autoConnect` 若在所用版本中不可用会直接报参数错误。

### 路径沙箱限制（`--workspace`）

新版 `chrome-devtools-mcp` 为修复路径穿越漏洞（CVE-2026-53766）**强制开启了文件路径沙箱**；同时 Claude Code 升级后会向 MCP 传递严格的“当前工作目录（Workspace Root）”限制。

结果是：当 MCP 需要访问运行目录之外的文件（比如让它读 `D:/Documents` 下的东西）时会失败。解决办法是在 `args` 里显式放行目录：

```
--workspace=D:/Documents
```

> [!tip] 排查顺序
> 1. 浏览器连不上 → 确认 Chrome 是用 `--remote-debugging-port=9222` 启动的，且端口没被占用
> 2. 读文件报路径错误 → 检查是否漏了 `--workspace`
> 3. 仍不通 → 换成不指定 `--browserUrl` 的默认配置，让它自己新开浏览器，先确认 MCP 本身可用
