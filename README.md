# image-vision-qwen

一个 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) Skill：**用 Qwen3.7-Plus（阿里云 DashScope）识别图片，主会话模型不动**。

适用于你在用一个不支持视觉（或视觉弱）的聊天模型，但偶尔需要识图，不想为了识图就切走整个会话模型。

## 它解决什么问题

很多人用 Claude Code 时，聊天模型走中转/代理（GLM、DeepSeek、本地模型等），这些模型可能：
- 不支持图片
- 或者视觉能力弱

而阿里云 DashScope 上的 **qwen3.7-plus** 视觉能力强、便宜。但如果你把整个会话切到 Qwen，又会遇到：

- `max_tokens` 超过 32768 被 DashScope 拒绝（Claude Code 默认会申请更大的值）
- 切走后日常聊天体验下降
- 切来切去状态混乱

这个 skill 的做法：**主会话模型永远不切，识图时直接旁路 curl 调一次 DashScope 的 qwen3.7-plus，`max_tokens` 卡死 32768，拿到结果继续用原模型对话。** 不动任何 provider 配置、不重启会话、无状态切换。

## 工作流程

```
用户发图 + 问题
   │
   ▼
Claude (主会话, 任意模型) 检测到是图片任务
   │
   ▼  旁路直接 curl 调 DashScope
   │  model=qwen3.7-plus, max_tokens=32768
   ▼
Qwen3.7-Plus 识别图片 → 返回结果
   │
   ▼
Claude 把识别结果交给用户, 继续用原模型对话
```

## 安装

### 方式 1：直接放到 Claude Code skills 目录

```bash
# 克隆
git clone https://github.com/<你的用户名>/image-vision-qwen.git

# 放到 Claude Code 能识别的 skills 目录
# 用户级（所有项目可用）：
mkdir -p ~/.claude/skills
ln -s /path/to/image-vision-qwen ~/.claude/skills/image-vision-qwen

# 或项目级（仅当前项目）：
mkdir -p .claude/skills
ln -s /path/to/image-vision-qwen .claude/skills/image-vision-qwen
```

### 方式 2：cc-switch 用户（自动同步）

如果你用 [cc-switch](https://github.com/farion1231/cc-switch) 管理 Claude Code provider：

```bash
# 放到 cc-switch 的 skills 目录，cc-switch 会自动 symlink 到 ~/.claude/skills
ln -s /path/to/image-vision-qwen ~/.cc-switch/skills/image-vision-qwen
```

## 配置 API Key

按优先级取 key（skill 自动探测，不用改代码）：

### 优先：环境变量（推荐，跨平台通用）

去阿里云百炼/DashScope 控制台创建 API-KEY，然后：

```bash
# 加到 ~/.bashrc / ~/.zshrc / Windows 环境变量
export DASHSCOPE_API_KEY=sk-xxxxxxxxxxxxxxxx
```

### 备选：cc-switch 自动探测（仅 Windows + cc-switch 用户）

如果你在 cc-switch 里配了「阿里云qwen」provider，skill 会自动从 `cc-switch.db` 读 token，**无需任何额外配置**。

## 用法

安装 + 配好 key 后，新开一个 Claude Code 会话，直接发图：

```
你：[发一张截图] 这个报错什么意思？
Claude：（自动触发 skill → 调 qwen3.7-plus 识别 → 用你当前的模型组织语言回答）
```

触发关键词：看图 / 识图 / 识别 / 描述图片 / 读图 / OCR / 这上面写了什么 / 分析这张图 等。

## 前置依赖

- **Claude Code**（CLI 版）
- **curl**（系统自带）
- **python 3**（解析返回 JSON；几乎所有系统都有）
- **DashScope API key**（阿里云百炼控制台获取）

## 特性

- ✅ **不切换主会话模型**——旁路调用，主会话零改动
- ✅ **彻底绕开 max_tokens 32768 限制**——显式卡死，不会 400
- ✅ **跨平台**——兼容 Windows (Git Bash) / macOS / Linux
- ✅ **双 key 来源**——环境变量优先，cc-switch.db 自动探测兜底
- ✅ **支持多图**——一次给多张图也能识别
- ✅ **无硬编码密钥**——key 运行时读取，安全可公开

## 限制

- 目前只测过 `qwen3.7-plus`。换其他视觉模型（如 `qwen-vl-max`）改 SKILL.md 里的 `model` 字段即可。
- DashScope 的 Anthropic 兼容端点对 `max_tokens` 上限是 32768，这是阿里云的限制，不是 skill 的限制。
- 大图可能较慢（base64 让 payload 变大），skill 已设 `--max-time 90`。

## 许可证

[MIT](LICENSE)
