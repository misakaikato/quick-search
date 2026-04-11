---
name: quick-opencli-search
description: |
  轻量、快速的搜索 skill。以 opencli 为主完成"搜一下 / 查一下 / 快速了解"类任务，强调低延迟、少步骤、直接给结论。
  Use when: 快速搜索、搜一下、查一下、先帮我看看、快速了解、找资料、找链接、read this page、summarize this page
  DO NOT TRIGGER when: 需要多轮深度调研、知识图谱沉淀、系统性证据链分析、正式研究报告
metadata:
  version: "1.5.0"
  category: research
  license: MIT
  last_updated: "2026-04-12"
  derived_from: "insight-probe"
  changelog: |
    1.5.0 — 六语言搜索支持：中文、英语、法语、日语、俄语、德语同时搜索，自动合并去重
    1.4.0 — 修复超时处理误区：-o 文件已由 opencli 增量写入；B站字幕已由 opencli 自动拉取
    1.3.0 — 修复 opencli --sources JSON 结构解析问题；curl searxng 作为可靠备选
    1.2.0 — searxng 每次必选；社区类作为通用源；摘要统一中文，英文自动翻译
    1.1.0 — 增加搜索超时处理、curl fallback、opencli web read 输出说明
    1.0.0 — 初始版本
---

# Quick OpenCLI Search
目标：保留 insight-probe 里最好用的 opencli 搜索 / 读取能力，但去掉知识图谱、任务编排、质量门控和重型流程。

核心定位：
- 更像"高速搜索助手"而不是"研究系统"
- 默认一次性完成搜索、抽样阅读、总结
- 优先响应速度，必要时再扩展

## 自然语言接口

| 用户说法 | 路由 | 默认动作 |
|---|---|---|
| "搜一下 XXX" / "查一下 XXX" / "帮我看看 XXX" | QUICK_SEARCH | 直接搜索并总结前几条结果 |
| "读一下这个页面" / "总结这个链接" | READ_PAGE | 直接读取页面并总结 |
| "再展开一点" / "多找几个来源" | EXPAND_SEARCH | 在上一轮基础上补充搜索 |
| "给我链接列表" | LINKS_ONLY | 只返回筛选后的链接 |
| "对比一下 A 和 B" | COMPARE | 分别搜索 A/B 并做快速对比 |

## 核心原则

- ALWAYS 优先使用 `opencli web search` 做搜索
- ALWAYS 优先使用 `opencli web read --url ...` 读取正文
- ALWAYS 先搜用户原始表述，避免一上来过度改写查询词
- ALWAYS 控制抓取量：默认只读前 3-5 个高价值页面
- ALWAYS 直接给出结论 + 链接，不做重量级中间产物
- ALWAYS 先确认 JSON 结构后再解析（见"JSON 解析规范"）
- NEVER 为简单搜索启动 kg / 图谱 / 多轮提取流程
- NEVER 在低风险场景先追问；能直接搜就直接搜
- NEVER 因为追求"全面"而牺牲响应速度

## JSON 解析规范（重要）

`opencli web search --sources` 返回的 JSON 结构**不固定**，必须先检查再解析：

### 已知的 JSON 结构

#### 结构一：`{_meta, _data: [{...}]}`
列表结构，`_data` 是结果数组，每项包含 `source`, `title`, `url`，可能还有 `content`：

```json
{"_meta": {...}, "_data": [{"source": "zhihu", "title": "xxx", "url": "..."}]}
```

#### 结构二：`{_meta, _data: {source1: [...], source2: [...]}}`
嵌套结构，`_data` 是按 source 分组的字典，每项可能包含 `content`：

```json
{"_meta": {"sourceCounts": {"zhihu": 10, "daizhige": 5}}, "_data": {"zhihu": [...], "daizhige": [...]}}
```

#### 结构三（SearXNG fallback）：`{results: [...], answers: [...]}`
SearXNG 返回 `results` 数组，每项包含 `title`, `content`, `url`：

```json
{"results": [{"title": "xxx", "content": "摘要...", "url": "..."}]}
```

### 解析流程

搜索完成后，**必须先打印前 300 字节确认结构**，再写解析脚本：

```python
import json

with open('/tmp/search_result.json', 'r', encoding='utf-8') as f:
    raw = f.read()
data = json.loads(raw)

# 先确认结构
if '_data' in data:
    d = data['_data']
    if isinstance(d, list):
        results = d  # 结构一
    elif isinstance(d, dict):
        # 结构二：展平所有 source 的结果
        results = []
        for src_results in d.values():
            if isinstance(src_results, list):
                results.extend(src_results)
elif 'results' in data:
    results = data['results']  # 结构三（SearXNG）
else:
    results = []

# 去重
seen_urls = set()
unique = []
for r in results:
    url = r.get('url', '')
    if url and url not in seen_urls:
        seen_urls.add(url)
        unique.append(r)
```

### 注意
- opencli --sources 返回的结果**可能没有 content 字段**（只有 title + url）
- 如果大部分结果缺少 content，直接 curl searxng 拿摘要更可靠
- searxng 返回的 JSON 结构简单稳定，优先使用

## Route 1: QUICK_SEARCH

适用：用户要"快速知道是什么 / 有哪些结果 / 大概情况如何"。

### 默认流程

1. **六语言并行搜索**：同时用以下六种语言搜索关键词
   - 中文（简体）、英语、法语、日语、俄语、德语
   - 每种语言分别选最合适的数据源（见"多语言搜索数据源配置"）
2. **searxng 每次必选**，作为所有语言的共同主数据源
3. **searxng 返回结果含 content**，直接使用；其他语言搜索走 `opencli web search`
4. 搜索后合并所有语言结果，按 URL 去重 + 按内容相似度去重
5. **超时后立即检查 -o 文件**：若 opencli 超时退出，先检查 `/tmp/search_result.json` 是否有内容，有则直接解析，不再走 fallback
6. 对前 5 条高质量来源读取正文（规则见"输出格式"）
7. 输出搜索结果列表，外语摘要自动翻译为中文

### 搜索命令

六语言并行搜索，每种语言独立命令：

```bash
# 中文
opencli web search "<keyword>" --sources searxng,zhihu,xiaohongshu,bilibili --limit 30 -f json -o /tmp/search_zh.json
# 英语
opencli web search "<keyword>" --sources searxng,google,reddit,hackernews --limit 30 -f json -o /tmp/search_en.json
# 法语
opencli web search "<keyword>" --sources searxng --limit 30 -f json -o /tmp/search_fr.json
# 日语
opencli web search "<keyword>" --sources searxng --limit 30 -f json -o /tmp/search_ja.json
# 俄语
opencli web search "<keyword>" --sources searxng --limit 30 -f json -o /tmp/search_ru.json
# 德语
opencli web search "<keyword>" --sources searxng --limit 30 -f json -o /tmp/search_de.json
```

### SearXNG Fallback

**重要：opencli 超时后，不要立即走 fallback，先检查 -o 文件。**

超时退出时，`/tmp/search_result.json` 可能已有部分结果（opencli 已支持增量写入）。步骤：

1. **先检查** `/tmp/search_result.json` 是否存在且有内容
2. **有内容** → 直接解析使用，不再调用 searxng
3. **无内容或结果过少** → 才用 curl 拿 searxng 兜底

如果 opencli --sources 结果不佳（如缺少 content 字段），直接用 curl 拿 searxng：

```bash
curl -s "http://127.0.0.1:10086/search?q=$(echo "<keyword>" | python3 -c 'import sys,urllib.parse;print(urllib.parse.quote(sys.stdin.read().strip()))')&format=json&limit=30" 2>&1
```

返回的 JSON 结构稳定（`results` 数组 + `content` 字段），直接取用。

### 结果数量建议

- 普通搜索：`--limit 50`（让更多相关结果进入筛选池）
- 用户明确要"多找一点"：`--limit 80`
- 默认读取正文数：3-5 页
- **最终展示不设硬性上限**，按内容相关性排序后全部展示

### 输出格式：搜索结果列表

搜索结果按相关性从高到低排列，每条包含：

```
[序号] <标题>
  摘要：<50-150字，含核心信息>
  来源：<domain1> · <类型> · <日期>
       <domain2> · <类型> · <日期>
  链接：<url1>
       <url2>
```

**展示规则**：
- **不设硬性上限**：所有与关键词相关的结果都展示
- **内容去重**：按内容相似度合并重复条目（标题相似 OR 摘要核心信息重叠），而非仅按 URL 去重
- **合并后多来源**：同一条目下显示所有来源，列在同一缩进下
- **完全无关的才排除**：标题和摘要均与关键词无关联的才过滤掉

其中 source type 可选标注：
- 官方（official）
- 百科（Wikipedia）
- 媒体（Reuters/BBC 等 major media）
- 社区（Hacker News / Reddit / 知乎 / 小红书）
- 博客（blog）
- 学术（arXiv / 学术机构）

**六语言结果格式**：
- 中文来源 → 保持原文
- 非中文来源（英/法/日/俄/德）→ 翻译为中文，标注 `[译自 <语言>.<source>]`
- 专业术语保留英文原文

示例：
```
搜"量子计算最新进展"结果（共 24 条，按相关性排列，合并重复后 15 条）：

[1] Google Quantum AI 展示新一代量子处理器
  摘要：Google Quantum AI 团队在 2024 年 12 月发布了最新量子处理器 Willow，量子纠错能力提升至 99.9%，在特定基准测试中展示了"量子优势"。该芯片拥有 105 个量子比特，计划于 2025 年开放云端访问。[译自 blog.google]
  来源：blog.google · 官方 · 2024-12
       zhihu.com · 社区
  链接：https://blog.google/technology/research/quantum-ai-willow/
       https://zhuanlan.zhihu.com/p/xxxxx

[2] IBM Quantum Network 2025 年路线图
  摘要：IBM 发布 2025 年量子计算发展路线图，计划推出 1000+ 量子比特系统，重点推进量子纠错和实用化应用。路线图显示量子体积（Quantum Volume）年均增长 2-3 倍。
  来源：ibm.com · 官方
  链接：https://www.ibm.com/quantum/roadmap

[3] ...
```

最后附一句：
> 共展示 X 条来源，合并重复后 Y 条。如需深入了解某一篇，我可以帮你细读。

## Route 2: READ_PAGE

适用：用户给了明确 URL，希望直接读内容。

```bash
opencli web read --url "<url>" -f json
```

输出：
- 这页主要讲什么
- 关键信息 3-7 条
- 如果页面有明显限制（登录墙 / 内容过短 / 抓取失败），明确说明

### B 站特例

若 URL 为 `bilibili.com/video/BV...`，优先拉字幕而不是读页面：

```bash
opencli bilibili subtitle <bvid> -f json
```

然后基于字幕做摘要。

## Route 3: EXPAND_SEARCH

适用：用户说"继续找""展开点""再多几个来源"。

### 默认做法

在原主题基础上，只补一个维度，不进入深研循环：
- 官方 / 一手来源
- 英文来源
- 中文讨论
- 最新动态
- 对立观点 / 争议点

示例：
```bash
opencli web search "<topic>" --sources searxng,zhihu --limit 10 -f json
opencli web search "<topic>" --sources searxng,google --limit 10 -f json
```

### 停止条件

满足任一即可停止：
- 已有 5 个以上高质量来源
- 新结果高度重复
- 用户只是想"先看看"而不是深挖

## Route 4: LINKS_ONLY

适用：用户只要链接，不要长摘要。

输出格式：
1. 标题 — URL
2. 标题 — URL
3. 标题 — URL

规则：
- 默认给 10 条
- 优先官方 / 权威 / 原始来源
- 去掉明显内容农场和低质量镜像站

## Route 5: COMPARE

适用：快速比较两个对象/观点/产品。

### 流程

1. 分别搜索 A / B
```bash
opencli web search "<A>" --sources searxng,zhihu --limit 5 -f json
opencli web search "<B>" --sources searxng,zhihu --limit 5 -f json
```

2. 如有必要，再搜比较词：
```bash
opencli web search "<A> vs <B>" --sources searxng,google --limit 8 -f json
```

3. 读取每边 1-2 个高价值来源后输出：
- 相同点
- 不同点
- 适用场景
- 推荐结论（如果证据足够）

## 搜索词策略

### 默认策略
- 第一条查询：直接用用户原话
- 第二条查询（仅必要时）：轻量改写，不引入复杂假设
- 第三条查询（仅用户要求继续）：补充英文、official、latest、review、documentation 之一

### 关键词自动优化规则

在搜索前，根据内容类型自动优化关键词，无需用户提示：

#### 1. 语言补全
- 检测到中文关键词 → 自动追加英文关键词搜索（或合并中英文一次搜索）
- 检测到纯英文 → 必要时追加中文关键词
- 原则：一次搜索尽量覆盖中英文，节省轮次

#### 2. 主题类型识别 + 自动扩展

| 主题类型 | 检测信号 | 自动追加词 |
|---------|---------|-----------|
| 技术文档 | 含框架名、API、开发工具 | `official docs` / `documentation` |
| 最新动态 | 含"最新、2024、2025、新闻" | `latest` / `news` |
| 评测/对比 | 含"评测、对比、比较、review" | `review` / `benchmark` |
| 官方信息 | 含"官网、官方、官方网站" | 去除非必要词，保留核心名 |
| 人物/公司 | 含人名或公司名 | 保留原名，不做过长拆解 |
| 历史/起源 | 含"历史、由来、起源" | `history` / `origin` |
| 教程/入门 | 含"教程、入门、学习" | `tutorial` / `getting started` |

#### 3. 查询简化
- 关键词超过 10 个字时，提取核心名词进行搜索
- 去除语气词、重复修饰词（如"非常详细地了解一下"→"详细了解"）
- 保留专业术语和核心实体

### 搜索失败回退
- 第一次搜索结果少于 3 条 → 缩短关键词，去除限定词再搜
- 第二次搜索结果仍差 → 改用 curl searxng 获取可靠摘要
- 第三次仍然失败 → 明确告知用户并提供相近建议

### 禁止事项
- 不要一开始就拆成很多学术子问题
- 不要自动进入多轮发散
- 不要为简单主题设计复杂研究框架
- 不要过度改写用户原话，导致语义偏移

## 数据源自动选择规则

根据关键词自动选择最合适的数据源，无需用户提示。

### 已知数据源列表

常见可用源（不区分大小写）：
`google`, `bing`, `wikipedia`, `zhihu`, `xiaohongshu`, `bilibili`, `github`, `stackoverflow`, `medium`, `dev.to`, `reddit`, `hackernews`, `twitter`, `youtube`, `tiktok`, `reuters`, `bbc`, `arxiv`, `scholar`, `cnki`, `douban`, `weibo`, `jike`, `amazon`, `substack`, `apple-podcasts`, `daizhige`, `searxng`

### 自动选择逻辑

#### 必选规则
- **`searxng` 每次必选**，无论何种内容类型，放在 source 列表首位
- **社区类作为通用源**，与内容类型数据源并列使用

#### 按内容类型选择

| 内容类型 | 检测信号 | 优先数据源 |
|---------|---------|-----------|
| 技术/开发 | 含编程语言、框架、API、工具名 | `github`, `stackoverflow`, `medium`, `dev.to` |
| 视频/教程 | 含"视频、教程、入门、how to" | `bilibili`, `youtube`, `xiaohongshu` |
| 学术/论文 | 含"论文、学术、研究、paper" | `arxiv`, `scholar`, `cnki` |
| 新闻/时事 | 含"最新、新闻、2024、2025、事件" | `reuters`, `bbc`, `twitter` |
| 产品/购物 | 含"购买、评测、产品、review" | `amazon`, `xiaohongshu`, `zhihu` |
| 社区讨论 | 含"怎么看、如何评价、讨论" | `zhihu`, `xiaohongshu`, `reddit`, `weibo` |
| 中文内容 | 用户中文提问，或含中文 | `zhihu`, `xiaohongshu`, `bilibili`, `douban` |
| 英文内容 | 用户英文提问，或专业术语 | `google`, `hackernews`, `reddit`, `medium` |
| 通用/未知 | 无特定信号 | `searxng`, `zhihu`, `xiaohongshu`, `bilibili` |

> **说明**：社区类（`zhihu`, `xiaohongshu`, `reddit`, `weibo`）作为通用源，适用于所有内容类型，与类型专用源并列选择。

#### 选择原则
1. **`searxng` 每次必选**：无论内容类型，第一个 source 永远是 `searxng`
2. **尽量选多**：默认选 4-6 个数据源，覆盖不同类型
3. **中英文兼顾**：中文关键词优先包含中文社区源；英文关键词优先包含英文社区源
4. **同类型去重**：同类型数据源选质量最高的 1-2 个

#### 执行示例
```bash
# 技术内容
opencli web search "React hooks 教程" --sources searxng,github,zhihu,stackoverflow --limit 30 -f json

# 学术内容
opencli web search "transformer attention mechanism" --sources searxng,arxiv,google,hackernews --limit 30 -f json

# 新闻/时事
opencli web search "2025 AI 最新进展" --sources searxng,reuters,bbc,zhihu --limit 30 -f json

# 社区讨论
opencli web search "iPhone 16 值得买吗" --sources searxng,zhihu,xiaohongshu,weibo --limit 30 -f json

# 通用内容
opencli web search "量子计算" --sources searxng,zhihu,bilibili,arxiv,google --limit 30 -f json
```

### 六语言搜索规则

**核心原则**：默认对所有搜索主题进行六语言（中文、英语、法语、日语、俄语、德语）并行搜索，不再区分"中国传统化"与"非中国传统化"。

**多语言数据源配置**：

| 语言 | 优先数据源 | 说明 |
|------|-----------|------|
| 中文 | `searxng`, `zhihu`, `xiaohongshu`, `bilibili` | 中文社区 + 通用 |
| 英语 | `searxng`, `google`, `reddit`, `hackernews` | 英文社区 + 通用 |
| 法语 | `searxng` | searxng 自动覆盖法语区引擎 |
| 日语 | `searxng` | searxng 自动覆盖日本引擎 |
| 俄语 | `searxng` | searxng 自动覆盖俄罗斯引擎 |
| 德语 | `searxng` | searxng 自动覆盖德语区引擎 |

**执行方式**：
1. **先将关键词译成六种语言**（agent 自行翻译，不调用额外工具）
2. 六条搜索命令同时发出（并发执行），各自写入独立临时文件
3. 各自解析后合并，按 URL 去重，再按内容相似度去重
4. searxng 返回结果天然含 content，优先使用；其他语言结果无 content 时，用 `opencli web read` 补充抓取
5. 按相关性排序后展示

**关键词翻译示例**（以"量子计算"为例）：

| 语言 | 翻译 |
|------|------|
| 中文 | 量子计算 |
| 英语 | quantum computing |
| 法语 | informatique quantique |
| 日语 | 量子コンピューティング |
| 俄语 | квантовые вычисления |
| 德语 | Quantencomputing |

**摘要翻译规则**：
- 中文来源 → 保持原文
- 非中文来源（英/法/日/俄/德）→ 翻译为中文，标注 `[译自 <语言>.<source>]`
- 专业术语保留英文原文（如"Transformer"、"API"、"LLM"等）

### 来源筛选规则

优先级从高到低：
1. 官方文档 / 官方博客 / 官方仓库
2. Wikipedia / ArXiv / 学术机构 / 大学页面
3. Reuters / BBC / major tech media
4. 高质量论坛 / Hacker News / Reddit / 知乎 / 小红书高质量回答
5. 普通博客

降权：
- 标题党
- 内容农场
- 无作者、无日期、无来源转载站
- 与用户问题明显不相关的 SEO 页面

## 输出格式规范

### 搜索结果列表（核心输出）

```
搜"<query>"结果（共 X 条来源，合并重复后 Y 条，按相关性排列）：

[1] <标题>
  摘要：<50-150字，含核心信息>
  来源：<domain1> · <类型> · <日期>
       <domain2> · <类型> · <日期>
  链接：<url1>
       <url2>

[2] <标题>
  摘要：...
  来源：...
  链接：...

[3] ...
```

说明：
- 按相关性从高到低排列
- **不设硬性上限**：所有与关键词相关的结果都展示
- **内容去重**：按内容相似度合并重复条目，而非仅按 URL 去重
- 合并后同一标题下可能有多条来源（跨平台的同一内容）
- 每条包含：标题、摘要（50-150字）、来源列表、链接列表
- 来源标注格式：域名 · 类型 · 日期（可选）
- 类型标注：官方 / 百科 / 媒体 / 社区 / 博客 / 学术
- 搜索摘要优先用 page 读取后的正文提炼，其次用搜索 snippet
- 英文摘要自动翻译为中文，标注 `[译自 <source>]`

### 链接列表模板

```
搜"<query>"找到这些更值得点开的结果：
1. <标题> — <domain> — <url>
2. <标题> — <domain> — <url>
3. <标题> — <domain> — <url>
4. <标题> — <domain> — <url>
5. <标题> — <domain> — <url>
```

### 页面摘要模板

```
这页主要在讲：<1-2句>

关键信息：
- ...
- ...

来源：<domain> · <类型>
链接：<url>

如果你愿意，我可以继续：
- 提炼成更短摘要
- 对比同主题其他来源
- 继续往下挖官方/英文资料
```

## 失败与回退

### 搜索超时问题

**opencli `web search` 已支持增量写入 `-o` 文件。**
每个子命令完成后立即将已有结果写入文件，不会因终端超时而丢失已获取的数据。

**处理流程：**
1. opencli 超时退出后，**立即检查** `/tmp/search_result.json` 是否存在
2. 有内容 → 直接解析使用（不需要 fallback）
3. 无内容或结果极少（<3条） → 才用 curl searxng 兜底
4. 仍无结果 → 缩短关键词重试

### 搜不到 / 结果差
1. 改用更短查询词再搜一次
2. 直接改用 curl searxng 获取可靠摘要
3. 改成英文关键词再搜一次

### 页面读取失败
- `opencli web read -f json -o` 只返回元数据（文件保存路径），不返回正文内容
- 若需要正文，用 `browser_navigate` + `browser_snapshot` 直接读取页面
- 遇到 401/403/反爬，记录失败原因并改读同主题其他来源

## 何时升级到 insight-probe

若出现以下情况，再切换到 insight-probe：
- 用户明确要"深度调研 / 系统分析 / 生成研究报告"
- 需要跨多轮维护研究状态
- 需要显式证据链、命题、缺口分析
- 需要沉淀为可持续扩展的研究图谱

一句话区分：
- quick-opencli-search = 快速找到答案
- insight-probe = 系统建立研究资产
