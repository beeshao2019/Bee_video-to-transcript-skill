---
name: "video-to-transcript"
description: "从视频链接提取音频并转录为 Markdown 文件。支持B站/抖音/快手/YouTube/TikTok等平台。快手通过KS-Downloader全自动下载（无需Cookie）；头条/视频号自动B站搜索回退。包含音频质量检测自动选模型、转录结果逻辑自检（可疑高频词/历史人物/文言文/连贯性）。当用户提供视频链接并要求转文字、转录、提取字幕时调用。"
---

# Video to Transcript — 多平台视频链接转文字

将视频链接自动下载音频并转录为带时间戳的 Markdown 文件。支持多平台，不支持的平台自动触发跨平台搜索回退。

## 支持的平台

| 平台 | 下载方式 | 状态 |
|------|---------|------|
| B站 (bilibili.com, b23.tv) | yt-dlp | 直接支持 |
| 抖音 (douyin.com) | yt-dlp | 直接支持 |
| YouTube (youtube.com, youtu.be) | yt-dlp | 直接支持 |
| TikTok (tiktok.com) | yt-dlp | 直接支持 |
| Twitter/X (x.com, twitter.com) | yt-dlp | 直接支持 |
| 快手 (kuaishou.com, v.kuaishou.com) | KS-Downloader 桥接 | **全自动，无需 Cookie** |
| 头条/西瓜视频 | — | 自动B站搜索回退 |
| 微信视频号 (weixin.qq.com/sph/) | — | 自动B站搜索回退 |
| 百度视频 | — | 自动B站搜索回退 |

## 依赖环境

**基础依赖（仅需一次）：**
```powershell
pip install yt-dlp faster-whisper requests curl-cffi aiofiles aiosqlite fastapi lxml pyyaml rich emoji
```

**KS-Downloader（已安装到 tools/KS-Downloader-master）：**
- 已下载并修复 Python 3.10 兼容性（f-string 和 path.walk）
- 桥接脚本：`d:\20260704_链接-视频-文本-后处理\ks_downloader_bridge.py`
- **2026-07-16 验证：无需 Cookie 即可直接下载快手视频**

## 完整流程（7步闭环）

```
Step 1: 下载视频/音频
Step 2: 音频质量检测 → 自动选择模型
Step 3: faster-whisper 转录
Step 4: 生成 transcript.md（Full Text + Timestamped Segments）
Step 5: 逻辑自检（历史人物/文言文/故事连贯性/可疑高频词）
Step 6: 如自检发现问题 → 标注修正建议
Step 7: 输出最终 transcript.md
```

## Step 2: 音频质量检测与模型选择

### 检测维度

下载音频后，在转录前进行以下检测，综合判断音频质量：

```python
NOISY_KEYWORDS = ['剪辑', '混剪', '影视', '片段', '解说', '配音', '混剪']
CLEAN_KEYWORDS = ['演讲', '采访', '播客', '对谈', '课程', '教学', '教程', 'TED']

def assess_audio_quality(title, duration_seconds, file_size_bytes):
    """
    返回质量等级: 'clean' / 'noisy' / 'poor'
    综合标题关键词 + 比特率 + 时长判断。
    """
    quality_issues = []

    # 1. 标题关键词预判
    title_lower = str(title).lower()
    has_noisy = any(kw in title_lower for kw in NOISY_KEYWORDS)
    has_clean = any(kw in title_lower for kw in CLEAN_KEYWORDS)

    # 2. 比特率检测
    if duration_seconds > 0 and file_size_bytes > 0:
        bitrate_kbps = (file_size_bytes * 8) / (duration_seconds * 1000)
        if bitrate_kbps < 64:
            quality_issues.append("低比特率（<64kbps），音频压缩严重")

    # 3. 极短视频
    if duration_seconds < 60:
        quality_issues.append("极短视频，语句可能不完整")

    # 综合: 标题关键词权重最高
    if has_noisy:
        return 'noisy'
    if has_clean and not quality_issues:
        return 'clean'
    if quality_issues:
        return 'poor' if len(quality_issues) > 1 else 'noisy'
    return 'noisy'  # 保守默认：快手短视频默认 noisy
```

### 视频标题关键词 → 音频质量预判

| 关键词模式 | 预判质量 | 原因 |
|-----------|---------|------|
| 剪辑、混剪、影视、片段、解说、配音 | noisy | 背景音乐/原片音轨叠加 |
| TED、演讲、采访、播客、对谈 | clean | 原声录制，清晰 |
| 短视频、随手拍、vlog | noisy | 环境噪音 |
| 动画、动漫、漫画 | clean | 人工配音，清晰 |
| 教程、课程、教学 | clean | 录屏或录音棚 |

### 模型选择决策树

```
评估音频质量
  │
  ├─ clean + 时长 < 10min → medium（最佳精度）
  ├─ clean + 时长 10-30min → base
  ├─ clean + 时长 > 30min → tiny
  │
  ├─ noisy + 时长 < 10min → medium（需要更高精度补偿噪音）
  ├─ noisy + 时长 10-30min → medium（同上）
  ├─ noisy + 时长 > 30min → base
  │
  ├─ poor + 时长 < 10min → medium
  ├─ poor + 时长 > 10min → base（太长用 medium 太慢）
  │
  └─ 中英混合/多语言 → base + 不指定 language（自动检测）
```

### 代码实现

```python
def choose_model(duration_seconds, quality='clean', multilingual=False):
    if multilingual:
        return 'base'  # 自动检测语言

    if quality in ('noisy', 'poor'):
        # 噪音环境需要更高精度
        if duration_seconds <= 600:   # <10min
            return 'medium'
        elif duration_seconds <= 1800: # <30min
            return 'medium'
        else:
            return 'base'
    else:  # clean
        if duration_seconds < 600:    # <10min
            return 'medium'
        elif duration_seconds < 1800: # 10-30min
            return 'base'
        else:
            return 'tiny'
```

### 强制简体中文输出

对于中文内容，始终使用 `initial_prompt` 强制简体输出，避免繁体乱码：

```python
segments, info = model.transcribe(
    audio_path,
    language="zh",
    task="transcribe",
    beam_size=5,
    initial_prompt="以下是普通话简体中文，请用简体中文输出。",
)
```

## Step 5: 转录结果逻辑自检

转录完成后，**必须**对 Full Text 进行逻辑自检。这是转录流程的内置步骤，不是可选操作。

### 自检流程

```
转录完成 → 读取 Full Text → 逐项自检 → 生成自检报告
  │
  ├─ 自检通过 → 输出 transcript.md
  │
  └─ 自检发现异常 → 在 transcript.md 末尾附加自检报告
       → 列出可疑内容 + 修正建议
       → 由执行者（AI agent）决定是否直接修正
```

### 自检清单

#### 0. 可疑高频词检测（最高优先级）

**原理**：whisper 的同音误识别往往是全局性的——同一个错误词会在全文反复出现（如"爷叔"被识别为"耶稣"出现 39 次）。通过检测高频词的语义与上下文是否匹配，可以在不需要预建角色名字典的情况下发现大量错误。

**不预建角色名字典的原因**：影视类视频标题通常不含剧名（如"看个毛线电影"），无法自动触发特定剧集的角色表；国产剧每年数百部无法穷举；同音替换有误伤风险（如角色"阿里"会替换正常文本）。

```python
# 已知容易在错误语境中高频出现的词
SUSPICIOUS_HIGH_FREQ_WORDS = {
    "耶稣": {"hint": "如果出现在中国现代/商业/都市语境，极可能是角色名误识别", "example": "爷叔"},
    "佛教": {"hint": "如果出现在非宗教语境，可能是'佛教'→'反馈'等", "example": "反馈"},
    "伊斯兰": {"hint": "如果出现在非宗教语境，可能是同音词误识别", "example": "视情况定"},
}

def check_suspicious_high_freq_words(text):
    """
    检测文本中的可疑高频词。
    策略：如果一个词在文本中出现 3 次以上，且属于已知可疑词表，标记为需人工确认。
    同时：即使不在已知表中，如果某个名词/人名在全文中出现 5 次以上但上下文语义
    完全不通顺（如"耶稣"出现在"上海滩""股票""和平饭店"的语境中），也应标记。
    """
    issues = []

    # 方法1: 已知可疑词表命中
    for word, info in SUSPICIOUS_HIGH_FREQ_WORDS.items():
        count = text.count(word)
        if count >= 3:
            issues.append(
                f"可疑高频词 \"{word}\" 出现 {count} 次 | "
                f"提示: {info['hint']} | 可能是: {info['example']}"
            )

    # 方法2: 统计全文所有词频，找出出现 >=5 次的名词/人名
    # 如果这些高频词的上下文存在明显语义冲突，标记
    # （此方法由 AI agent 在自检时通过语义判断执行，不依赖固定字典）

    return issues
```

**自检规则**：
- 任何词在全文出现 **3 次以上** 且在已知可疑词表中 → **必须标记**
- 出现 5 次以上的专有名词，如果上下文语境明显不匹配（如宗教人物出现在商战语境）→ 标记为"可疑高频词+语境冲突"
- 执行者（AI agent）看到可疑高频词标记后，应根据上下文语义判断正确词是什么，然后**全文替换修正**
- 每次确认修正后，将新发现的模式追加到 `SUSPICIOUS_HIGH_FREQ_WORDS` 字典和下方知识库

#### 1. 历史人物姓名校验

检查文本中出现的人名是否为已知历史人物或知名人物的正确写法：

```python
# 常见被误识别的历史人物名（从实际案例积累）
KNOWN_MISRECOGNIZED_NAMES = {
    "计文子": "季文子",       # 春秋鲁国大夫
    "子文之": "子闻之",       # 论语原文
    "顾宇": "古语",           # 非人名，是"古语说得好"
    "同时": "孔子",           # 上下文为"孔子听说了"
}

def check_historical_names(text):
    """检查文本中是否有被误识别的历史人物名。"""
    issues = []
    for wrong, correct in KNOWN_MISRECOGNIZED_NAMES.items():
        if wrong in text:
            issues.append(f"人名/术语误识别: \"{wrong}\" → 应为 \"{correct}\"")
    return issues
```

**自检规则**：
- 遇到"XX子""XX公""XX父"等古代称谓 → 核实是否为正确的历史人物名
- 遇到疑似人名但上下文不通顺 → 检查是否为同音词误识别
- 遇到"孔子""老子""孟子""庄子"等 → 确认上下文是否匹配该人物

#### 2. 文言文/古语校验

检查文本中引用的文言文、成语、古语是否正确：

```python
# 常见古语误识别模式
KNOWN_MISRECOGNIZED_CLASSICAL = {
    "三思二呼吸": "三思而后行",
    "三思而无行": "三思而后行",
    "再思可疑": "再思可矣",
    "此文之约再思可疑": "子闻之，再思可矣",
}

def check_classical_chinese(text):
    """检查文言文引用是否正确。"""
    issues = []
    for wrong, correct in KNOWN_MISRECOGNIZED_CLASSICAL.items():
        if wrong in text:
            issues.append(f"文言文误识别: \"{wrong}\" → 应为 \"{correct}\"")
    return issues
```

**自检规则**：
- 遇到半文半白的句子 → 重点检查文言部分
- 遇到"XX曰""XX云""XX谓" → 后面的引文可能是古语，需核实
- 遇到四字短语读不通 → 检查是否为成语/古语的同音误识别

#### 3. 故事/叙述逻辑连贯性校验

检查文本的叙事逻辑是否通顺连贯：

```python
def check_narrative_logic(text):
    """检查故事叙述的逻辑连贯性。"""
    issues = []

    # 1. 检查是否有前后矛盾
    # 例：前面说"三思而后行"，后面变成"三思而无行"

    # 2. 检查是否有突兀的话题跳转
    # 例：商业话题突然跳到历史典故，中间缺少过渡

    # 3. 检查因果链是否完整
    # 例："因为A，所以B" → A和B之间逻辑是否成立

    # 4. 检查对话 attribution 是否合理
    # 例：A说的话被标成B说的

    # 5. 检查数字/时间/人名的一致性
    # 例：同一个人前面叫"张三"后面叫"章三"

    return issues
```

**自检规则**：
- 对话体内容 → 检查每个说话人的观点是否前后一致
- 叙事体内容 → 检查时间线、因果关系、人物行为是否合理
- 列举体内容（如"第一...第二..."）→ 检查编号和内容是否对应

#### 4. 常识/同音词校验

```python
# 常见同音词误识别（从实际案例积累）
KNOWN_MISRECOGNIZED_HOMOPHONES = {
    "钱都不得先量": "前途不可限量",
    "钱太子": "潜台词",
    "陈连": "职位",
    "版面例子": "反面例子",
}

def check_homophones(text):
    """检查同音词误识别。"""
    issues = []
    for wrong, correct in KNOWN_MISRECOGNIZED_HOMOPHONES.items():
        if wrong in text:
            issues.append(f"同音词误识别: \"{wrong}\" → 应为 \"{correct}\"")
    return issues
```

### 自检报告格式

自检报告附加在 transcript.md 末尾（仅在发现问题时附加）：

```markdown
---

## 转录自检报告

> 模型: medium | 检测时间: 2026-07-17 00:30

### 发现问题 (3项)

| # | 类型 | 原文 | 建议修正 | 置信度 |
|---|------|------|---------|--------|
| 1 | 同音词 | 钱太子 | 潜台词 | 高 |
| 2 | 文言文 | 三思而无行 | 三思而后行 | 高 |
| 3 | 历史人物 | 子文之曰再思可疑 | 子闻之，再思可矣 | 高 |

### 逻辑连贯性

- 对话逻辑：通顺，两人对话围绕"三思而后行"展开，A引用古语，B误解其含义，A纠正
- 叙事完整：完整呈现了一个用古语暗示商业决策的场景

### 已修正

以上3项已在 Timestamped Segments 和 Full Text 中直接修正。
```

### 知识库积累

每次发现新的误识别模式，应更新以下内容到本 SKILL.md 的 `KNOWN_MISRECOGNIZED_*` 字典中：

**已积累的误识别模式（2026-07-16/17 实战）：**

| 原文（误识别） | 正确 | 类型 | 根因 |
|--------------|------|------|------|
| 耶稣（39处） | 爷叔 | 可疑高频词 | 《繁花》角色名，与"耶稣"同音，全文反复出现 |
| 钱都不得先量了 | 前途不可限量 | 同音词 | 快手影视剪辑，背景音干扰 |
| 钱太子就是 | 潜台词就是 | 同音词 | 同上 |
| 专门设计一个陈连啊 | 专门设计一个职位了 | 同音词 | 同上 |
| 顾宇说得好啊 | 古语说得好啊 | 同音词 | 同上 |
| 三思二呼吸 / 三思而无行 | 三思而后行 | 文言文 | 古语，模型训练数据不足 |
| 季文子的版面例子 | 季文子的反面例子 | 同音词 | 同上 |
| 子文之曰再思可疑 | 子闻之，再思可矣 | 文言文+历史人物 | 论语原文，低频词 |
| 同时听说了 | 孔子听说了 | 历史人物 | 同音词 + 上下文 |
| 提兰桥 | 提篮桥 | 地名 | 上海著名监狱，模型未学习过 |
| 暴露了阿宝想法的机灵 | 暴露了阿宝想法的肤浅 | 语义不通 | "机灵"与上下文矛盾 |
| 一百二十块?粗 | 一百二十块?不 | 语气词 | "粗"应为"不" |
| 旁亲好的时候 | 行情好的时候 | 同音词 | 股市语境 |
| 淘淘 | 陶陶 | 角色名 | 《繁花》角色 |
| 报庭 | 报亭 | 名词 | 街边设施 |

## 快手视频处理流程（全自动闭环）

### 闭环流程图

```
检测到快手链接 (kuaishou.com / v.kuaishou.com)
  │
  ├─ Step 1: KS-Downloader 初始化（无需 Cookie）
  │    KS(server_mode=True) + curl_cffi 自动模拟浏览器 TLS 指纹
  │
  ├─ Step 2: 短链重定向 → 获取视频详情 → 下载视频
  │
  ├─ Step 3: 音频质量检测
  │    根据视频标题/描述预判 + 比特率分析 → 选择模型
  │
  ├─ Step 4: faster-whisper 转录（带 initial_prompt）
  │
  ├─ Step 5: 逻辑自检（可疑高频词/历史人物/文言文/同音词/连贯性）
  │
  ├─ Step 6: 如自检发现问题 → 修正
  │
  └─ Step 7: 输出 transcript.md（含自检修正） → 闭环完成
```

### 完整下载脚本（无需 Cookie）

```python
import sys, os, asyncio, subprocess
ks_root = r'D:\20260704_链接-视频-文本-后处理\tools\KS-Downloader-master'
if ks_root not in sys.path:
    sys.path.insert(0, ks_root)
os.chdir(ks_root)
from source.app.app import KS

URL = '快手链接'
OUTPUT_DIR = '输出目录'

async def download():
    os.makedirs(OUTPUT_DIR, exist_ok=True)
    ks = KS(server_mode=True)
    async with ks:
        await ks.database.update_config_data('Disclaimer', 1)
        urls = await ks.examiner.run(URL)
        if not urls:
            print('ERROR: Failed to resolve short URL'); return None
        real_url = urls[0]
        data = await ks.detail_one(real_url, download=False, cookies='')
        if isinstance(data, str):
            print(f'ERROR: {data}'); return None
        if not data:
            print('ERROR: No data'); return None
        video_url = str(data.get('download', '')).split()[0]
        if not video_url:
            print('ERROR: No download URL'); return None
        import requests as req
        video_path = os.path.join(OUTPUT_DIR, 'video_raw.mp4')
        resp = req.get(video_url, timeout=180, stream=True)
        with open(video_path, 'wb') as f:
            for chunk in resp.iter_content(chunk_size=8192):
                f.write(chunk)
        audio_path = os.path.join(OUTPUT_DIR, 'audio_raw.m4a')
        try:
            subprocess.run(
                ['ffmpeg', '-i', video_path, '-vn', '-acodec', 'copy', audio_path, '-y'],
                capture_output=True, timeout=120)
            if os.path.exists(audio_path):
                os.remove(video_path)
                return audio_path
        except:
            pass
        return video_path

audio = asyncio.run(download())
```

### 转录 + 自检脚本

```python
from faster_whisper import WhisperModel
import time, os

# Step 1: Choose model based on audio quality
model_name = choose_model(duration_seconds, quality=assess_quality(title))

# Step 2: Transcribe with simplified Chinese
model = WhisperModel(model_name, device="cpu", compute_type="int8")
segments, info = model.transcribe(
    audio_path, language="zh", task="transcribe", beam_size=5,
    initial_prompt="以下是普通话简体中文，请用简体中文输出。",
)

# Step 3: Write markdown（格式参考下方"输出文件"章节的 transcript.md 结构）

# Step 4: Self-check
issues = []
issues.extend(check_suspicious_high_freq_words(full_text))
issues.extend(check_historical_names(full_text))
issues.extend(check_classical_chinese(full_text))
issues.extend(check_homophones(full_text))
issues.extend(check_narrative_logic(full_text))

# Step 5: If issues found, append self-check report
if issues:
    md += build_self_check_report(issues, model_name)
```

### 快手容错流程

```
快手链接
  │
  ├── KS-Downloader 直接下载（无需 Cookie）→ 成功 → 继续
  │
  ├── 下载失败（反爬/网络）→ 提示用户提供 Cookie → 用 Cookie 重试
  │    └── 用户不提供 → 自动触发 B站 搜索回退
  │
  └── 搜索回退 → B站搜索API → 用户确认 → yt-dlp 下载 → 继续
```

## 跨平台搜索回退

当快手下载失败、头条/视频号等平台不支持时：
1. 用文件夹名或用户提供的关键词搜索B站
2. 调用 B站搜索 API
3. 展示搜索结果，用户确认
4. 用 yt-dlp 下载并转录

```python
import requests
resp = requests.get(
    "https://api.bilibili.com/x/web-interface/search/type",
    params={"search_type": "video", "keyword": keyword, "page": 1},
    headers={"User-Agent": "Mozilla/5.0...", "Referer": "https://www.bilibili.com/"}
)
first_result = resp.json()["data"]["result"][0]
bvid = first_result["bvid"]
```

## 各平台代码参考

### 平台检测

```python
def detect_platform(url):
    url_lower = url.lower()
    if any(x in url_lower for x in ['bilibili.com', 'b23.tv']): return 'bilibili'
    elif any(x in url_lower for x in ['douyin.com']): return 'douyin'
    elif any(x in url_lower for x in ['kuaishou.com', 'v.kuaishou.com']): return 'kuaishou'
    elif any(x in url_lower for x in ['toutiao.com', 'ixigua.com']): return 'toutiao'
    elif 'weixin.qq.com/sph/' in url_lower: return 'wechat_video'
    elif any(x in url_lower for x in ['youtube.com', 'youtu.be']): return 'youtube'
    else: return 'unknown'
```

### yt-dlp 下载（B站/抖音/YouTube）

```python
import yt_dlp
ydl_opts = {
    'format': 'bestaudio',
    'outtmpl': os.path.join(output_dir, 'audio_raw.%(ext)s'),
    'postprocessors': [],
    'quiet': True,
}
with yt_dlp.YoutubeDL(ydl_opts) as ydl:
    info = ydl.extract_info(url, download=True)
```

### KS-Downloader 桥接（快手，无需 Cookie）

```python
# 桥接脚本已处理所有细节：
# - 短链重定向
# - curl_cffi TLS 指纹模拟（无需 Cookie）
# - 视频详情解析
# - 视频下载
# - ffmpeg 音频提取
# 用法:
python ks_downloader_bridge.py "快手链接" "输出目录"
# 如遇反爬，可手动传 Cookie:
python ks_downloader_bridge.py "快手链接" "输出目录" --cookies "Cookie值"
```

## 模型选择

| 模型 | 大小 | CPU速度 | 准确率 | 适用场景 |
|------|------|---------|--------|----------|
| `tiny` | ~75MB | ~0.5x实时 | 中等 | 长音频（>30min）+ 清晰音频 |
| `base` | ~145MB | ~0.3x实时 | 较好 | 中等时长 + 噪音音频 |
| `medium` | ~1.5GB | ~0.05x实时 | 很好 | 短音频（<30min）或噪音环境 |
| `large` | ~3GB | 极慢 | 最好 | 不推荐CPU使用 |

**自动决策**: 由 `choose_model(duration, quality)` 函数自动选择，参见上方决策树。

## 输出文件

```
<output_dir>/
├── audio_raw.m4a / video_raw.mp4  # 下载的原始音频（或视频文件）
└── transcript.md                   # 转录结果（元信息 + 完整文本 + 时间戳分段 + 自检报告）
```

## 已验证的闭环案例

| 日期 | 平台 | 链接 | 质量 | 模型 | 自检 | 输出 |
|------|------|------|------|------|------|------|
| 2026-07-17 | 快手 | nJZSY4qH | noisy(影视解说) | medium | 7项52处(含可疑高频词) | 10.0KB |
| 2026-07-17 | 快手 | nPX5Jhun | noisy(影视剪辑) | medium→base→medium | 8处修正 | 3.4KB |
| 2026-07-16 | 快手 | KQI7Lka9 | clean(TED) | tiny→base(中英混合) | 无 | 2.0KB |
| 2026-07-16 | 快手 | J9brG6Yw | clean | tiny | 无 | 15.7KB |
| 2026-07-15 | 快手→B站回退 | KjKTEkln | — | tiny | 无 | 16.3KB |
| 2026-07-15 | B站 | idGpCeR | clean | base | 无 | 28.8KB |

## 常见问题

| 问题 | 解决方案 |
|------|----------|
| `Unsupported URL` (yt-dlp) | 不支持的平台会自动触发跨平台搜索回退 |
| 快手"获取作品数据失败" | 通常是反爬触发，可等待后重试或提供 Cookie 重试 |
| ffmpeg 音频提取失败 | faster-whisper 可直接处理 mp4 文件，跳过音频提取步骤 |
| SSL 错误下载模型失败 | 从 hf-mirror.com 手动下载到 `~/.cache/faster_whisper/<model>/` |
| `Fraction` 格式错误 | 加 `float()` 转换 |
| 转录语言错误 | 手动指定 `language` 参数 |
| ffmpeg not found | `pip install faster-whisper` 自带 FFmpeg |
| 跨平台搜索无结果 | 尝试缩短关键词或换用不同的关键词 |
| KS-Downloader Python 3.12+ 错误 | 已修复 f-string 和 path.walk 兼容性 |
| av 库 duration 读取异常 | 加判断：`if dur > 3600 and file_size < 100MB: duration_str = "unknown"` |
| 繁体输出 | 加 `initial_prompt="以下是普通话简体中文，请用简体中文输出。"` |
| 影视剪辑类识别差 | 自动选 medium 模型 + 转录后逻辑自检 |
| 文言文/古语识别错误 | 自检步骤会检测已知误识别模式并建议修正 |