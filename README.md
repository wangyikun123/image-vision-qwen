# image-vision-qwen

一个 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) Skill：**用阿里云 DashScope 视觉模型识别图片，主会话模型不动**。

适用于你在用一个不支持视觉（或视觉弱）的聊天模型，但偶尔需要识图，不想为了识图就切走整个会话模型。

## 它解决什么问题

很多人用 Claude Code 时，聊天模型走中转/代理（GLM、DeepSeek、本地模型等），这些模型可能：
- 不支持图片
- 或者视觉能力弱

而阿里云 DashScope 上的视觉模型（如 **qwen3.7-plus**、qwen-vl-max）视觉能力强、便宜。但如果你把整个会话切到 Qwen，又会遇到：

- `max_tokens` 超过 32768 被 DashScope 拒绝（Claude Code 默认会申请更大的值）
- 切走后日常聊天体验下降
- 切来切去状态混乱

这个 skill 的做法：**主会话模型永远不切，识图时直接旁路 curl 调一次 DashScope，`max_tokens` 卡死 32768，拿到结果继续用原模型对话。** 不动任何 provider 配置、不重启会话、无状态切换。

## 工作流程

```
用户发图 + 问题
   │
   ▼
Claude (主会话, 任意模型) 检测到是图片任务
   │
   ▼  旁路直接 curl 调 DashScope
   │  model=$VISION_MODEL (默认 qwen3.7-plus), max_tokens=32768
   ▼
DashScope 视觉模型识别图片 → 返回结果
   │
   ▼
Claude 把识别结果交给用户, 继续用原模型对话
```

## 安装

```bash
# 克隆
git clone https://github.com/wangyikun123/image-vision-qwen.git

# 放到 Claude Code 能识别的 skills 目录
# 用户级（所有项目可用）：
mkdir -p ~/.claude/skills
ln -s /path/to/image-vision-qwen ~/.claude/skills/image-vision-qwen

# 或项目级（仅当前项目）：
mkdir -p .claude/skills
ln -s /path/to/image-vision-qwen .claude/skills/image-vision-qwen
```

## 用法

安装 + 配好 `DASHSCOPE_API_KEY` 环境变量后，新开一个 Claude Code 会话，直接发图：

```
你：[发一张截图] 这个报错什么意思？
Claude：（自动触发 skill → 调 DashScope 视觉模型识别 → 用你当前的模型组织语言回答）
```

触发关键词：看图 / 识图 / 识别 / 描述图片 / 读图 / OCR / 这上面写了什么 / 分析这张图 等。

## 前置依赖

- **Claude Code**（CLI 版）
- **curl**（系统自带）
- **python 3**（解析返回 JSON；几乎所有系统都有）
- **`DASHSCOPE_API_KEY` 环境变量**（必填）：阿里云百炼/DashScope 控制台获取 API-KEY，`export DASHSCOPE_API_KEY=sk-xxx`
- **`VISION_MODEL` 环境变量**（可选）：默认 `qwen3.7-plus`，可随意改成 `qwen-vl-max`、`qwen-vl-plus` 等任意 DashScope 视觉模型，无需改 skill 代码

## 特性

- ✅ **不切换主会话模型**——旁路调用，主会话零改动
- ✅ **彻底绕开 max_tokens 32768 限制**——显式卡死，不会 400
- ✅ **跨平台**——兼容 Windows (Git Bash) / macOS / Linux
- ✅ **模型可随意更改**——`VISION_MODEL` 环境变量换任意 DashScope 视觉模型，不改代码
- ✅ **支持多图**——一次给多张图也能识别
- ✅ **无硬编码密钥**——key 运行时从环境变量读取，安全可公开

## 限制

- 默认模型 `qwen3.7-plus`（实测通过）；其他视觉模型通过 `VISION_MODEL` 环境变量切换即可。
- DashScope 的 Anthropic 兼容端点对 `max_tokens` 上限是 32768，这是阿里云的限制，不是 skill 的限制。
- 大图可能较慢（base64 让 payload 变大），skill 已设 `--max-time 90`。

## 许可证

[MIT](LICENSE)
