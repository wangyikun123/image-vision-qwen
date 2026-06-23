---
name: codex-image-vision-kimi
description: 用 Nvidia NIM 的 Kimi K2.6 (moonshotai/kimi-k2.6) 识别图片。当用户提供图片文件、截图、base64 图片，或要求看图/识图/分析图片内容/读图/OCR 时使用。主会话模型永远不变——图片识别是一次直接旁路调用 Nvidia NIM API（OpenAI 兼容端点），结果返回后继续用 Codex 当前模型。支持通过 NVIDIA_API_KEY 环境变量或 cc-switch.db 自动取 key。
---

# 图片识别 via Nvidia NIM (Kimi K2.6)

## 核心原则（最重要）

**主会话的模型永远不切换。** 图片识别是一次**直接 PowerShell 调用 Nvidia NIM** 的旁路操作：
- 不动 cc-switch、不改 env、不重启会话
- 主会话继续用 Codex 当前选中的模型（GLM / DeepSeek / 任意）
- Nvidia NIM 是 OpenAI 兼容端点，格式标准，无需担心 max_tokens 限制

适用于：你在用某个不支持视觉（或视觉弱）的聊天模型，但偶尔需要识别图片，不想为了识图就切走整个会话的模型。

## 何时触发

满足任一即触发：
- 用户给出**图片文件路径**（`.png` `.jpg` `.jpeg` `.gif` `.webp` `.bmp`）
- 用户**贴了 base64 图片数据**
- 用户要求：「看图」「识图」「识别这张图」「描述图片」「读图」「OCR」「这上面写了什么」「分析这张截图/界面/图表」
- 任务里出现截图、UI 图、图表、照片且需要理解其内容

**不要触发**：纯文本任务、写代码、git 操作、问答——这些用主会话当前模型即可。

## 前置准备：API Key

用 PowerShell 获取 Nvidia API key。**优先环境变量 `NVIDIA_API_KEY`**，没有则从 `cc-switch.db` 自动查找：

```powershell
# 方式 1: 环境变量（推荐）
$NVIDIA_KEY = $env:NVIDIA_API_KEY

# 方式 2: 从 cc-switch.db 自动取（查找 base URL 含 nvidia 的 provider）
if (-not $NVIDIA_KEY) {
    $NVIDIA_KEY = python -c @"
import sqlite3, json
c = sqlite3.connect(r'C:\\Users\\hao18\\.cc-switch\\cc-switch.db')
for row in c.execute("SELECT settings_config FROM providers WHERE app_type='claude'").fetchall():
    if not row[0]: continue
    cfg = json.loads(row[0]); env = cfg.get('env', {})
    if 'nvidia' in (env.get('ANTHROPIC_BASE_URL') or '').lower():
        print(env.get('ANTHROPIC_AUTH_TOKEN', '')); break
"@
}

if (-not $NVIDIA_KEY) {
    Write-Error "未找到 NVIDIA API key。请设置 \"`$env:NVIDIA_API_KEY 或在 cc-switch 里配置 Nvidia provider。"
    exit 1
}
```

> **想换视觉模型？** 设 `$env:VISION_MODEL` 环境变量即可。默认用 `moonshotai/kimi-k2.6`。Nvidia NIM 上可用的视觉模型很多（如 `qwen/qwen3-vl-plus`、`meta/llama-4-maverick` 等），可在 [NVIDIA NIM 目录](https://build.nvidia.com/explore/discover) 查找。

## 执行流程

### 1. 准备图片 → base64

将图片转为 base64 写入临时文件（避免大图超命令行长度限制）：

```powershell
$IMG_PATH = "C:\\path\\to\\image.png"  # 用户给的路径

# 推断 media_type
$MT = switch -Wildcard ($IMG_PATH) {
    '*.png'  { 'image/png' }
    '*.jpg'  { 'image/jpeg' }
    '*.jpeg' { 'image/jpeg' }
    '*.gif'  { 'image/gif' }
    '*.webp' { 'image/webp' }
    '*.bmp'  { 'image/bmp' }
    default  { 'image/png' }
}

# 转 base64（Windows 上 base64 不加换行）
$B64 = [Convert]::ToBase64String([IO.File]::ReadAllBytes($IMG_PATH))
$B64_PATH = "$env:TEMP\\img.b64"
[IO.File]::WriteAllText($B64_PATH, $B64)
```

**如果用户直接给了 base64**（贴在对话里），直接使用原值，但要去掉可能的 `data:image/...;base64,` 前缀和所有换行符。

### 2. 构造请求体，调用 Nvidia NIM

⚠️ Nvidia NIM 是 **OpenAI 兼容端点**（不是 Anthropic 格式）。图片用 `image_url` 块。

用 Python 构建 JSON（保证格式正确），PowerShell `Invoke-RestMethod` 发送：

```powershell
# 模型（可通过 VISION_MODEL 环境变量切换）
$VISION_MODEL = if ($env:VISION_MODEL) { $env:VISION_MODEL } else { 'moonshotai/kimi-k2.6' }

# Python 构建请求 JSON 到临时文件
python -c @"
import json
b64 = open(r'$env:TEMP\\img.b64').read().strip()
mt = '$MT'
body = {
    'model': '$VISION_MODEL',
    'messages': [{
        'role': 'user',
        'content': [
            {'type': 'image_url', 'image_url': {'url': f'data:' + mt + ';base64,' + b64}},
            {'type': 'text', 'text': r'<用户对图片的问题>'}
        ]
    }]
}
json.dump(body, open(r'$env:TEMP\\nvidia_req.json', 'w'), ensure_ascii=False)
"@

# 调用 Nvidia NIM API（OpenAI 兼容端点）
$RESP = Invoke-RestMethod \
    -Uri 'https://integrate.api.nvidia.com/v1/chat/completions' \
    -Method Post \
    -Headers @{ Authorization = "Bearer $NVIDIA_KEY" } \
    -ContentType 'application/json' \
    -InFile "$env:TEMP\\nvidia_req.json" \
    -TimeoutSec 90
```

### 3. 解析返回，提取识别结果

Nvidia NIM 返回标准 OpenAI Chat Completions 格式：

```powershell
# 主结果始终在 choices[0].message.content
$TEXT = $RESP.choices[0].message.content
Write-Output $TEXT
```

如果模型返回了 reasoning/thinking 内容，可能在 `choices[0].message.reasoning_content` 里（取决于模型），主结果始终在 `.content`。

### 4. 把结果交回主会话

把识别结果告诉用户，然后**继续用当前模型**走后续对话。不需要任何“切回”操作——因为从一开始就没切走。

## 多图场景

用户一次给多张图：在 content 数组里放多个 `image_url` 块：

```python
'content': [
    {'type': 'image_url', 'image_url': {'url': f'data:image/png;base64,{b64_1}'}},
    {'type': 'image_url', 'image_url': {'url': f'data:image/png;base64,{b64_2}'}},
    {'type': 'text', 'text': '对比这两张图...'}
]
```

## 注意事项

- **API 格式**：Nvidia NIM 是 OpenAI 兼容端点，用 `image_url` 块；不要用 Anthropic 的 `image`/`source` 块。
- **API key 永远不打印**：取完直接用，不要 echo / Write-Host / print。
- **大图用临时文件**：请求体写入 `$env:TEMP\nvidia_req.json`，通过 `-InFile` 参数发送，避免命令行长度限制。
- **key 前缀**：cc-switch 里的 Nvidia key 前缀是 `nvapi-...`，不是 `sk-...`。
- **超时设置**：`-TimeoutSec 90` 给大图足够的处理时间。
- **media_type 必须和实际图片格式一致**：不要对 jpg 写 image/png。

## 故障排查

| 现象 | 原因 | 处理 |
|------|------|------|
| `401 Unauthorized` | API key 无效或过期 | 检查 `$env:NVIDIA_API_KEY` 或 cc-switch 里 Nvidia provider 的 token |
| `404 Not Found` | 模型名不存在 | 到 [NVIDIA NIM](https://build.nvidia.com/) 确认模型名正确且已开通 |
| `400 Bad Request` | 请求格式不对 | 确认用的是 OpenAI 格式（`image_url` 块），不是 Anthropic 格式（`image` 块） |
| 超时 | 图太大 | 加大 `-TimeoutSec` 或先压缩图片 |
| 认证失败 | key 没传到 header | 确认 `Authorization: Bearer ` 后接 key，key 不能有前后空格 |

## 和 Claude 版本的关系

本 skill 是 `image-vision-qwen`（Claude Code 用 DashScope Qwen 识图）的 Codex 姐妹篇。核心设计理念一致——旁路调用，主会话零改动：

- **Claude 版**：调 DashScope Anthropic 端点，用 `image` 块（Anthropic 格式）
- **Codex 版**：调 Nvidia NIM OpenAI 端点，用 `image_url` 块（OpenAI 格式）

两个 skill 共存在同一个 GitHub 仓库 [image-vision-qwen](https://github.com/wangyikun123/image-vision-qwen)。
