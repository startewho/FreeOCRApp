---
name: FreeOCR MCP
description: 使用 FreeOCR 在本地识别 PDF、图片和截图。优先调用 MCP 工具；没有 MCP 时直接运行 FreeOcr.Cli.exe。包含工具选择、绝对路径、页码范围、引擎选择、结果读取和错误处理。
---

# FreeOCR

FreeOCR 的发布包里同时有：

- `FreeOcr.Cli.exe`：OCR 命令行工具，也可以作为 MCP 服务启动；
- `Data`：Indra 与 PaddleOCR 模型；
- `SKILL.md`：本说明，包根目录和 `skills/freeocr-mcp/` 下各一份。

解压后即可使用，不需要安装。

## 1. 选择调用方式

| 当前环境 | 使用方式 |
|---|---|
| 会话里有 `ocr_file`、`ocr_files`、`ocr_image`、`get_server_info` 工具 | MCP 工具 |
| 只能执行命令，或没有配置 MCP | 直接运行 `FreeOcr.Cli.exe` |

MCP 和 CLI 都支持 Indra 与 PaddleOCR：

```powershell
# CLI 直接用 PaddleOCR
.\FreeOcr.Cli.exe "D:\scans\page.png" --engine paddle --format text

# MCP 服务默认使用 PaddleOCR
.\FreeOcr.Cli.exe mcp --engine paddle
```

手写中文、多语言扫描件建议用 PaddleOCR；Indra 对手写内容效果较差。

## 2. MCP 方式

客户端可能给工具加前缀，例如 `freeocr_ocr_file`、`mcp__freeocr__ocr_file`。按**后缀**匹配：

| 工具 | 用途 |
|---|---|
| `ocr_file` | 一个 PDF、图片或多帧 TIFF |
| `ocr_files` | 多个文件 |
| `ocr_image` | base64 图片或 `data:` URL |
| `get_server_info` | 查看版本和两个引擎的模型状态 |

启动服务：

```text
FreeOcr.Cli.exe mcp
```

客户端配置中的命令写解压后的实际路径：

```json
{
  "mcpServers": {
    "freeocr": {
      "command": "<解压目录>\\FreeOcr.Cli.exe",
      "args": ["mcp"]
    }
  }
}
```

常用参数：

| 参数 | 默认 | 说明 |
|---|---|---|
| `path` / `paths` | 必填 | 本地文件路径，建议绝对路径 |
| `engine` | 服务默认 | `indra` 或 `paddle` |
| `format` | `markdown` | `markdown`、`text`、`json` |
| `mode` | `auto` | `auto`、`ocr`、`text`；`text` 仅 Indra |
| `first_page` | `1` | 起始页，从 1 开始 |
| `max_pages` | `0` | 最多读几页；`0` 表示全部 |
| `dpi` | `200` | PDF 渲染分辨率 |
| `password` | — | 加密 PDF 密码 |
| `math` | `false` | 输出 LaTeX（仅 Indra） |
| `keep_furniture` | `false` | 保留页眉页脚（仅 Indra） |
| `image_ocr` | `true` | 是否 OCR 文字层页面里的图片文字 |
| `threads` | `0` | 线程数；`0` 表示全部核心 |

注意：`first_page` 从 1 开始，而返回结果里的 `page` 从 0 开始，汇报时要换算。

## 3. CLI 方式

在发布包解压目录运行：

```powershell
.\FreeOcr.Cli.exe "D:\docs\contract.pdf" --format md
```

常用命令：

```powershell
# 纯文本
.\FreeOcr.Cli.exe "D:\scans\a.png" --format text

# PaddleOCR 识别中文/手写
.\FreeOcr.Cli.exe "D:\scans\中文.png" --engine paddle --format text

# 只读前 5 页，输出到文件
.\FreeOcr.Cli.exe "D:\docs\contract.pdf" --max-pages 5 --out "D:\ocr"

# 数学公式转 LaTeX
.\FreeOcr.Cli.exe "D:\docs\paper.pdf" --math

# 多个图片依次识别到标准输出
.\FreeOcr.Cli.exe "D:\scans\a.png" "D:\scans\b.jpg" --format text
```

常用参数：

| 参数 | 默认 | 说明 |
|---|---|---|
| `inputs...` | 必填 | 一个或多个 PDF/图片路径 |
| `--format md\|text\|json` | `md` | 输出格式 |
| `--out DIR` | 无 | 每个输入写一个文件到目录 |
| `--mode auto\|ocr\|text` | `auto` | PDF 文字层策略；`text` 仅 Indra |
| `--engine indra\|paddle` | `indra` | OCR 引擎 |
| `--max-pages N` | `0` | 每个文件最多读几页 |
| `--dpi N` | `200` | PDF 渲染分辨率 |
| `--password PWD` | 无 | 加密 PDF 密码 |
| `--math` | 关 | 数学公式输出 LaTeX（仅 Indra） |
| `--keep-furniture` | 关 | 保留页眉页脚（仅 Indra） |
| `--no-image-ocr` | 关 | 不识别文字层页面里的图片文字 |
| `--cores N` | 全部 | 使用的核心数 |
| `--model-dir DIR` | exe 旁 `Data` | 指定 Indra 模型目录 |
| `--paddle-dir DIR` | `Data\paddleocr` | 指定 PaddleOCR 模型目录 |

退出码：`0` 成功，`1` 有输入或模型失败，`2` 参数错误。失败原因在 stderr。

CLI 会自动读取 exe 旁 `Data` 里的 Indra 模型；如果 exe 和 `Data` 分开了，用 `--model-dir` 指向模型目录。

## 4. 路径规则

- MCP 参数和 CLI 文件参数都使用**绝对路径**；相对路径会按服务进程的工作目录解析。
- 客户端配置里的 `command` 也必须是解压后的实际路径。
- 不要把示例里的 `<解压目录>` 原样传入，它只是一个占位符。

## 5. 读取结果

**MCP**：

- 主要文本在 `content[0].text`；后面的 `content` 块可能包含失败提示。
- `structuredContent` 包含每页的 `lines`（`box`、`text`、`conf`）、`blocks`、`note`、`timings`；需要坐标或置信度时用它。
- `isError: true` 表示工具执行失败，文本是原因，不是识别结果。

**CLI**：

- 文本输出到 stdout；`--out` 时每个输入一个文件。
- 错误和耗时输出到 stderr。
- `--format json` 每页一行 JSON。

不要编造识别结果里没有的文字；空白页就说空白。结果很长时先总结，用户要求全文时再完整输出。

## 6. 错误处理

| 现象 | 处理 |
|---|---|
| `no such file` | 换成绝对路径 |
| `path is required` | 补上 `path` 或 `paths` |
| `models are not available` | 确认 `Data` 在 exe 旁；或用 `--model-dir` |
| `PaddleOCR ... lacks ...` | 确认 `Data\paddleocr` 完整，或改用 Indra |
| `password-protected PDF` | 向用户询问密码 |
| `first_page ... beyond` | 检查页码和文档页数 |
| CLI 退出码 2 | 参数写法不对，先看 stderr 的 usage |
| `Request cancelled` | 不要自动重试 |
| JSON-RPC `-32602` | 工具名或参数不对 |

工具错误不会断开 MCP 会话；CLI 失败也不会影响下一次运行。修正后重试即可。

## 7. 使用原则

1. 长 PDF 先限制页数，并告诉用户读了多少页。
2. 只做文字提取时用 `text`；需要版面或坐标时用 `json`；一般阅读用 `markdown`。
3. 只有小字扫描件才把 `dpi` 调到 300。
4. `math: true` / `--math` 更慢且仅 Indra 支持，只在有公式时开启。
5. 批量文件尽量在同一个 MCP 会话完成，已加载的模型会复用。
