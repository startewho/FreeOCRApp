---
name: FreeOCR MCP
description: 首次使用必须先让用户下载 FreeOCRApp 的发布包并解压，或提供 FreeOcr.Cli.exe 的绝对路径；随后用 MCP 工具或 CLI 识别 PDF、图片和截图。包含路径校验、工具选择、页码范围、引擎选择和错误处理。
---

# FreeOCR

FreeOCR 的程序不在本技能目录里。发布地址：

<https://github.com/startewho/FreeOCRApp>

发布包里有：

- `FreeOcr.Cli.exe`：OCR 命令行工具，也可以作为 MCP 服务启动；
- `Data`：Indra 与 PaddleOCR 模型；
- `SKILL.md`：本说明。

## 0. 先由用户准备程序

**这一步必须由用户完成，代理不要猜测路径，也不要擅自下载或解压。**

按以下顺序处理：

1. 用户已经给出 `FreeOcr.Cli.exe` 的绝对路径：直接跳到第 1 节校验。
2. 用户给出解压目录：把 `FreeOcr.Cli.exe` 拼到目录后面，再校验。
3. 用户还没有程序：请用户完成：
   - 打开上面的 Releases 页面；
   - 下载 `FreeOCR-mcp-win-x64-*.zip`（ARM64 机器下载 `arm64` 版本）；
   - 解压到一个固定目录，**保留 `Data` 和 `FreeOcr.Cli.exe` 的相对位置**，不要直接在 ZIP 或临时目录里运行；
   - 把解压后的 `FreeOcr.Cli.exe` 绝对路径发回来。
4. 用户无法安装或拒绝提供路径：停止调用，说明 FreeOCR 必须先准备程序。

不要使用 `.\FreeOcr.Cli.exe` 这种相对写法，除非用户确认当前目录就是解压目录。

## 1. 校验程序

假设用户给出的路径保存在 `$cli`（下面是 PowerShell 示例）：

```powershell
$cli = "C:\Tools\FreeOCR\FreeOcr.Cli.exe"   # 换成用户提供的真实路径
Test-Path $cli
Test-Path (Join-Path (Split-Path $cli) "Data\det.onnx")
& $cli --version
```

期望：

- `Test-Path $cli` 为 `True`；
- `Data\det.onnx` 存在（模型随包预置，不会下载）；
- `--version` 输出 `freeocr 1.1.2` 或更高版本。

任何一项失败都不要继续 OCR，先让用户修正路径或重新解压。

## 2. 选择调用方式

| 当前环境 | 使用方式 |
|---|---|
| 会话里有 `ocr_file`、`ocr_files`、`ocr_image`、`get_server_info` 工具 | MCP 工具 |
| 只能执行命令，或没有配置 MCP | 直接运行 `$cli` |

MCP 和 CLI 都支持 Indra 与 PaddleOCR：

```powershell
# CLI 直接用 PaddleOCR
& $cli "D:\scans\page.png" --engine paddle --format text

# MCP 服务默认使用 PaddleOCR
& $cli mcp --engine paddle
```

手写中文、多语言扫描件建议用 PaddleOCR；Indra 对手写内容效果较差。

## 3. MCP 方式

MCP 需要用户先在客户端里添加服务。代理可以生成配置让用户粘贴，但不要假设服务已经存在。

客户端可能给工具加前缀，例如 `freeocr_ocr_file`、`mcp__freeocr__ocr_file`。按**后缀**匹配：

| 工具 | 用途 |
|---|---|
| `ocr_file` | 一个 PDF、图片或多帧 TIFF |
| `ocr_files` | 多个文件 |
| `ocr_image` | base64 图片或 `data:` URL |
| `get_server_info` | 查看版本和两个引擎的模型状态 |

客户端配置（把 `<CLI 绝对路径>` 换成第 1 节校验过的真实路径）：

```json
{
  "mcpServers": {
    "freeocr": {
      "command": "<CLI 绝对路径>",
      "args": ["mcp"]
    }
  }
}
```

启动服务本身是：

```text
"<CLI 绝对路径>" mcp
```

常用参数：

| 参数 | 默认 | 说明 |
|---|---|---|
| `path` / `paths` | 必填 | 本地文件绝对路径 |
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

## 4. CLI 方式

```powershell
& $cli "D:\docs\contract.pdf" --format md
```

常用命令：

```powershell
# 纯文本
& $cli "D:\scans\a.png" --format text

# PaddleOCR 识别中文/手写
& $cli "D:\scans\中文.png" --engine paddle --format text

# 只读前 5 页，输出到文件
& $cli "D:\docs\contract.pdf" --max-pages 5 --out "D:\ocr"

# 数学公式转 LaTeX
& $cli "D:\docs\paper.pdf" --math

# 多个图片依次识别到标准输出
& $cli "D:\scans\a.png" "D:\scans\b.jpg" --format text
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

CLI 会自动读取 exe 旁 `Data` 里的模型；如果 exe 和 `Data` 分开了，用 `--model-dir` 指向模型目录。

## 5. 路径规则

- 第 1 节必须先得到 `$cli` 的**绝对路径**，后续所有命令都使用它。
- MCP 参数和 CLI 文件参数都使用**绝对路径**；相对路径会按服务进程的工作目录解析。
- 客户端配置里的 `command` 也必须是这个绝对路径。
- 不要使用示例里的占位符 `<CLI 绝对路径>` 直接启动。

## 6. 读取结果

**MCP**：

- 主要文本在 `content[0].text`；后面的 `content` 块可能包含失败提示。
- `structuredContent` 包含每页的 `lines`（`box`、`text`、`conf`）、`blocks`、`note`、`timings`；需要坐标或置信度时用它。
- `isError: true` 表示工具执行失败，文本是原因，不是识别结果。

**CLI**：

- 文本输出到 stdout；`--out` 时每个输入一个文件。
- 错误和耗时输出到 stderr。
- `--format json` 每页一行 JSON。

不要编造识别结果里没有的文字；空白页就说空白。结果很长时先总结，用户要求全文时再完整输出。

## 7. 错误处理

| 现象 | 处理 |
|---|---|
| 程序不存在 / `Test-Path` 为 False | 让用户重新提供路径或重新解压 |
| `Data\det.onnx` 缺失 | 发布包不完整；让用户重新解压并保留 `Data` |
| `no such file` | 换成绝对路径 |
| `models are not available` | 确认 `Data` 在 exe 旁；或用 `--model-dir` |
| `PaddleOCR ... lacks ...` | 确认 `Data\paddleocr` 完整，或改用 Indra |
| `password-protected PDF` | 向用户询问密码 |
| `first_page ... beyond` | 检查页码和文档页数 |
| CLI 退出码 2 | 参数写法不对，先看 stderr 的 usage |
| `Request cancelled` | 不要自动重试 |
| JSON-RPC `-32602` | 工具名或参数不对 |

工具错误不会断开 MCP 会话；CLI 失败也不会影响下一次运行。修正后重试即可。

## 8. 使用原则

1. 先完成第 0、1 节；没有程序就不要开始识别。
2. 长 PDF 先限制页数，并告诉用户读了多少页。
3. 只做文字提取时用 `text`；需要版面或坐标时用 `json`；一般阅读用 `markdown`。
4. 只有小字扫描件才把 `dpi` 调到 300。
5. `math: true` / `--math` 更慢且仅 Indra 支持，只在有公式时开启。
6. 批量文件尽量在同一个 MCP 会话完成，已加载的模型会复用。
