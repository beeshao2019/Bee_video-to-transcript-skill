# Bee video-to-transcript

一个用于 TRAE / Claude Code / Cursor 等 AI 编程助手的 Skill，将视频链接自动转录为带时间戳的 Markdown 文件。

## 它是什么

这不是一个独立的 Python 程序，而是一个 **AI Agent Skill**（`SKILL.md`）。安装后，当你对 AI 助手说"把这个视频链接转成文字"时，AI 会自动按照 SKILL.md 中定义的 7 步闭环流程执行全部操作。

## 支持的平台

| 平台 | 下载方式 | 备注 |
|------|---------|------|
| B站 (bilibili.com, b23.tv) | yt-dlp | 直接支持 |
| 抖音 (douyin.com) | yt-dlp | 直接支持 |
| YouTube / TikTok / Twitter/X | yt-dlp | 直接支持 |
| 快手 (kuaishou.com, v.kuaishou.com) | KS-Downloader + curl_cffi | 全自动，无需 Cookie |
| 头条 / 西瓜视频 / 微信视频号 / 百度视频 | — | 自动 B 站搜索回退 |

## 核心特性

### 7 步闭环流程

```
下载视频/音频 → 音频质量检测 → 自动选模型 → faster-whisper 转录
→ 生成 transcript.md → 逻辑自检 → 修正问题 → 输出最终文件
```

### 智能模型选择

根据视频标题关键词预判 + 比特率分析 + 时长，自动在 `tiny` / `base` / `medium` 之间选择最合适的 Whisper 模型：

| 音频质量 | 时长 | 选择模型 | 原因 |
|---------|------|---------|------|
| clean | <10min | medium | 最佳精度 |
| clean | 10-30min | base | 速度与精度平衡 |
| clean | >30min | tiny | 长音频速度优先 |
| noisy/poor | <30min | medium | 高精度补偿噪音 |
| noisy/poor | >30min | base | 避免过慢 |

### 转录结果逻辑自检（5 项）

转录完成后自动对全文进行逻辑自检，这是本 Skill 区别于普通转录工具的核心差异：

1. **可疑高频词检测**（最高优先级）— whisper 的同音误识别往往是全局性的（如"爷叔"被识别为"耶稣"出现 39 次），通过词频 + 语境冲突检测自动发现
2. **历史人物姓名校验** — 检查"季文子""子闻之"等低频词是否被误识别
3. **文言文/古语校验** — 检查"三思而后行"等古语是否被错误转录
4. **同音词校验** — 检查"潜台词"→"钱太子"等常见同音误识别
5. **叙述逻辑连贯性** — 检查前后矛盾、话题跳转、人物名称一致性

自检报告附加在 `transcript.md` 末尾，高置信度问题直接修正，低置信度问题保留原文并标注建议。

### 快手反爬绕过

- KS-Downloader 使用 `curl_cffi` 模拟浏览器 TLS 指纹，**无需 Cookie** 即可下载
- 当 KS-Downloader 被验证码拦截时，可降级为直接解析页面 `__APOLLO_STATE__` 提取视频 URL
- 仍失败则自动触发 B 站搜索回退

## 输出格式

每个视频生成一个 `transcript.md`，包含：

```markdown
# 20260719_视频标题

> Source: [原始链接](...)
> Platform: Kuaishou
> Duration: 5m9s
> Model: medium | Quality: noisy

---

## Full Text
（完整的连续文本，已合并分段，已应用自检修正）

---

## Timestamped Segments
- [00:00 - 00:02] 第一句话
- [00:02 - 00:04] 第二句话
...

---

## 转录自检报告（仅在发现问题时附加）
| # | 类型 | 原文 | 建议修正 | 置信度 |
|---|------|------|---------|--------|
```

## 依赖环境

```bash
pip install yt-dlp faster-whisper requests curl-cffi
```

- **faster-whisper**: OpenAI Whisper 的 CTranslate2 加速版本，用于语音转录
- **yt-dlp**: B站/抖音/YouTube 等平台的视频下载
- **curl-cffi**: 快手反爬绕过（TLS 指纹模拟）
- **ffmpeg**: 音频提取（faster-whisper 自带）

## 使用方法

### 在 TRAE 中使用

将 `SKILL.md` 放入项目的 `.trae/skills/video-to-transcript/` 目录下，然后直接对话：

```
Use Skill: video-to-transcript https://v.kuaishou.com/xxxxx 提取视频，输出到 D:\output_dir
```

### 在 Claude Code / Cursor 中使用

将 `SKILL.md` 内容放入 `CLAUDE.md` 或 `.cursor/rules/` 中作为系统指令，AI 会自动识别视频链接并执行转录流程。

## 实战验证

| 日期 | 平台 | 类型 | 模型 | 自检结果 | 耗时 |
|------|------|------|------|---------|------|
| 2026-07-19 | 快手 | 职场解说 | medium | 8项修正 + 2项存疑 | 53min |
| 2026-07-17 | 快手 | 影视解说(繁花) | medium | 7项52处(含可疑高频词"耶稣"→"爷叔") | 141min |
| 2026-07-17 | 快手 | 影视剪辑 | medium | 8处修正 | — |
| 2026-07-16 | 快手 | TED演讲 | base | 无(中英混合自动检测) | — |
| 2026-07-15 | B站 | — | base | 无 | — |

> CPU 模式下 medium 模型速度约 0.05x 实时（1 分钟音频 ≈ 20 分钟处理），建议噪音/重要内容使用 medium，清晰长视频使用 base 或 tiny。

## 误识别知识库

Skill 内置了一个持续积累的误识别模式字典，涵盖：
- 同音词（"潜台词"→"钱太子"、"前途不可限量"→"钱都不得先量"）
- 文言文（"三思而后行"→"三思二呼吸"）
- 历史人物（"子闻之"→"子文之"、"孔子"→"同时"）
- 地名（"提篮桥"→"提兰桥"）
- 角色名（"陶陶"→"淘淘"、"爷叔"→"耶稣"）

每次转录发现新模式都会自动追加，越用越准。

## 文件结构

```
SKILL.md              # Skill 定义文件（AI Agent 的完整指令）
README.md             # 本文件
```

## License

MIT