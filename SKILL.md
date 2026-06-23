---
name: image-vision-qwen
description: 用阿里云 DashScope 视觉模型识别图片（默认 Qwen3.7-Plus，可通过 VISION_MODEL 环境变量随意更换为 qwen-vl-max 等）。当用户提供图片文件、截图、base64 图片，或要求看图/识图/分析图片内容/读图/OCR 时使用。主会话模型永远不变——图片识别是一次直接旁路调用 DashScope，结果返回后继续用用户当前模型。
---

# 图片识别 via DashScope 视觉模型

## 核心原则（最重要）

**主会话的模型永远不切换。** 图片识别是一次**直接 curl 调用 DashScope** 的旁路操作：
- 不动任何 provider 配置、不改 env、不重启会话
- 主会话继续用用户当前选中的模型（任意）
- 调用视觉模型时 **max_tokens 显式卡死 ≤ 32768**（DashScope 的硬上限，超了会 400）

适用于：你在用某个不支持视觉（或视觉弱）的聊天模型，但偶尔需要识别图片，不想为了识图就切走整个会话的模型。

## 何时触发

满足任一即触发：
- 用户给出**图片文件路径**（`.png` `.jpg` `.jpeg` `.gif` `.webp` `.bmp`）
- 用户**贴了 base64 图片数据**
- 用户要求：「看图」「识图」「识别这张图」「描述图片」「读图」「OCR」「这上面写了什么」「分析这张截图/界面/图表」
- 任务里出现截图、UI 图、图表、照片且需要理解其内容

**不要触发**：纯文本任务、写代码、git 操作、问答——这些用主会话当前模型即可。

## 前置准备：API Key 与模型

```bash
# API key（必填）：阿里云百炼/DashScope 控制台获取
DASHSCOPE_KEY="${DASHSCOPE_API_KEY:-}"

# 识别模型（可选，默认 qwen3.7-plus，可随意改成 qwen-vl-max / qwen-vl-plus / qwen3-vl-plus 等任意 DashScope 视觉模型）
VISION_MODEL="${VISION_MODEL:-qwen3.7-plus}"

if [ -z "$DASHSCOPE_KEY" ]; then
  echo "错误: 未设置 DASHSCOPE_API_KEY 环境变量。"
  echo "获取方式：阿里云百炼/DashScope 控制台 → 创建 API-KEY，然后 export DASHSCOPE_API_KEY=sk-xxx"
  exit 1
fi
```

> **想换视觉模型？** 设 `VISION_MODEL` 环境变量即可，例如 `export VISION_MODEL=qwen-vl-max`，无需改 skill 代码。实测可用：`qwen3.7-plus`（默认）、`qwen-vl-max`、`qwen-vl-plus`、`qwen3-vl-plus`。

## 执行流程

### 1. 准备图片 → base64

**文件转 base64**（跨平台处理换行差异）：

```bash
IMG_PATH="/path/to/image.png"   # 用户给的路径

# 自动推断 media_type
case "$(echo "$IMG_PATH" | tr '[:upper:]' '[:lower:]')" in
  *.png)  MT="image/png" ;;
  *.jpg|*.jpeg) MT="image/jpeg" ;;
  *.gif)  MT="image/gif" ;;
  *.webp) MT="image/webp" ;;
  *.bmp)  MT="image/bmp" ;;
  *) MT="image/png" ;;  # 默认
esac

# base64 编码并去换行（兼容 macOS/BSD 和 Linux/GNU）
if base64 -w 0 "$IMG_PATH" >/dev/null 2>&1; then
  base64 -w 0 "$IMG_PATH" > /tmp/img.b64        # GNU coreutils / Git Bash
else
  base64 "$IMG_PATH" | tr -d '\n' > /tmp/img.b64  # macOS / BSD
fi
IMG_B64=$(cat /tmp/img.b64)
```

**如果用户直接给了 base64**（贴在对话里），直接用，但要：
- 去掉可能的 `data:image/png;base64,` 前缀
- 去掉所有换行后写入 `/tmp/img.b64`

### 2. 构造请求体到文件（关键：避免大图超命令行长度限制）

⚠️ **不要用 `curl -d "..."` 内联 base64**——大图 base64 会超命令行长度限制（`Argument list too long`）。必须先把请求体写到文件，再用 `--data @file`：

```bash
# 把图片 base64 和用户问题组装成 JSON 请求体，写到文件
python -c "
import json
body = {
    'model': '$VISION_MODEL',
    'max_tokens': 32768,
    'messages': [{
        'role': 'user',
        'content': [
            {'type': 'image', 'source': {'type': 'base64', 'media_type': '$MT', 'data': open('/tmp/img.b64').read().strip()}},
            {'type': 'text', 'text': '''<这里填用户的识图问题>'''}
        ]
    }]
}
json.dump(body, open('/tmp/req.json','w'), ensure_ascii=False)
"
```

### 3. 调用 DashScope（关键：max_tokens ≤ 32768）

```bash
RESP=$(curl -s --max-time 90 https://dashscope.aliyuncs.com/apps/anthropic/v1/messages \
  -H "Authorization: Bearer $DASHSCOPE_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  --data @/tmp/req.json)
```

### 4. 解析返回，提取识别结果

返回 JSON 结构（实测）：
```json
{"content":[{"type":"thinking","thinking":"..."},{"type":"text","text":"实际识别结果"}], "stop_reason":"end_turn", ...}
```

**提取 `text` 块**（跳过 thinking 块）：

```bash
echo "$RESP" | python -c "
import sys, json
d = json.load(sys.stdin)
if 'error' in d:
    print('错误:', d['error']); sys.exit(1)
for b in d.get('content', []):
    if b.get('type') == 'text':
        print(b['text'])
        break
"
```

### 5. 把结果交回主会话

把识别结果告诉用户，然后**继续用当前模型**走后续对话。不需要任何"切回"操作——因为从一开始就没切走。

## 多图场景

用户一次给多张图：把多个 image block 放进 content 数组（同样写到文件再 `--data @file`）：
```json
"content": [
  {"type":"image","source":{"type":"base64","media_type":"image/png","data":"<图1>"}},
  {"type":"image","source":{"type":"base64","media_type":"image/png","data":"<图2>"}},
  {"type":"text","text":"对比这两张图..."}
]
```

## 注意事项

- **max_tokens 永远 ≤ 32768**。默认就用 32768（DashScope 上限）。别用更大的值，DashScope 会 400 拒绝并报 `Range of max_tokens should be [1, 32768]`。
- **media_type 必须和实际图片格式一致**，否则 DashScope 报错。
- **base64 不能有换行**：写入文件前确保 `tr -d '\n'`。
- **大图必须用 `--data @file`**：`curl -d "..."` 内联 base64 会因命令行长度限制失败（`Argument list too long`）。
- **大图**：base64 会让 JSON payload 变大，`--max-time 90` 给足超时。
- **不要打印 DASHSCOPE_KEY**：取完直接用，不要 echo。

## 故障排查

| 现象 | 原因 | 处理 |
|------|------|------|
| `Range of max_tokens should be [1, 32768]` | max_tokens 超 32768 | 已固定 32768，不该出现；若出现检查 curl 命令是否被改 |
| `InvalidParameter` model 相关 | 模型名错 | 确认 `VISION_MODEL` 的值是 DashScope 支持的视觉模型 |
| `403 Forbidden` | 该模型未在账号开通 | 到阿里云百炼控制台开通该模型，或换 `qwen3.7-plus` |
| `Argument list too long` | 用了 `curl -d` 内联大图 base64 | 改用 `--data @file`（见执行流程第 2-3 步） |
| 401 / 认证失败 | key 失效 | 重新检查 `DASHSCOPE_API_KEY` |
| 图片格式错 | media_type 不匹配 | 按实际扩展名设 media_type |
| 超时 | 图太大 | 加大 `--max-time`；或先压缩图片 |

## 设计原理（给未来的你）

如果用"切 provider → 识图 → 切回"的方案：
1. 要改全局激活 provider
2. 要重启会话 / proxy 才生效
3. `CLAUDE_CODE_EFFORT_LEVEL=max` 会让客户端发 >32768 的 max_tokens，DashScope 直接 400
4. 切回时还得记住原来用哪个——状态管理复杂

旁路直接调 DashScope 绕开了**全部**这些问题：主会话零改动，max_tokens 完全可控，无状态切换。
