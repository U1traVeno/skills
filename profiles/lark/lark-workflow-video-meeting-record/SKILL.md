---
name: lark-workflow-video-meeting-record
version: 1.0.0
description: "自定义飞书视频会议整理工作流：将用户给定时段内的多段飞书视频会议或用户直接提供的多个会议/妙记链接视为同一次会议，基于逐字稿自行合并生成会议纪要，并按飞书会议分段整理逐字稿。当用户需要把长视频会议整理到指定飞书知识库、合并多段会议纪要、保留逐段逐字稿和妙记原文时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 多段视频会议纪要与逐字稿整理

> **前置条件：** 开始前先阅读 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，再按需阅读 [`../lark-vc/SKILL.md`](../lark-vc/SKILL.md)、[`../lark-doc/SKILL.md`](../lark-doc/SKILL.md)、[`../lark-wiki/SKILL.md`](../lark-wiki/SKILL.md)。

## 适用场景

- 用户要求整理飞书视频会议、会议录制、妙记、逐字稿，并输出到飞书知识库。
- 用户说明一场长会议因为飞书一小时会议/录制限制被拆成多段，需要视为同一次会议。
- 用户给出一个时间段，要求把该时间段内相关视频会议合并整理。
- 用户直接给出多个会议链接、会议 ID、妙记链接或 minute_token，要求合并成一份纪要。

## 必要输入

用户必须给出总结输出所在的飞书知识库，通常是知识库 URL 或 space_id。

还需要至少一种会议定位方式：

- 时间段：例如 `5.29 22:00 到 5.30 02:00`。
- 会议链接或会议 ID：来自飞书视频会议记录。
- 妙记链接或 minute_token：来自飞书妙记。

如果缺少知识库位置，先向用户索要；不要默认写入个人知识库。

## 核心规则

1. **会议纪要不要使用妙记 AI 智能纪要。**
   - `vc +notes` 返回的 `summary`、`todos`、`chapters` 只能作为参考线索，不得直接作为最终会议纪要。
   - 最终“大纲纪要”和“待办任务”必须由你读取并理解逐字稿后自行总结。

2. **多段飞书会议按语义视为同一次会议。**
   - 用户描述的同一时段内会议，或用户直接提供的多个会议/妙记链接，默认合并为同一次会议。
   - 大纲纪要合并成一份，不按飞书会议机械拆分。
   - 逐字稿文档仍按每一次飞书视频会议分成几个大段。

3. **逐字稿保留双层结构。**
   - 每段飞书视频会议都包含：
     - AI 优化语意逐字稿：保留主要发言顺序、说话人和技术含义，删除明显口癖、重复和 ASR 错误。
     - 妙记原文：完整保留飞书导出的原始逐字稿，标明原始妙记链接。

4. **输出到同一个父文档的两个子文档。**
   - 在目标知识库中创建或复用一个父文档，标题必须以会议开始日期开头，格式为 `YY.MM.DD <会议主题>`，例如 `26.05.29 给 Modo 快速过一下 Paramer 的应用功能`。
   - 如果会议跨自然日，父文档标题日期使用第一段会议的开始日期。
   - 父文档下放两个子文档：
     - `总结纪要：<会议主题>`
     - `逐字稿：<会议主题>`
   - 父文档正文放两个子文档入口链接和内容范围说明。

## 工作流

### Step 1: 解析目标知识库

如果用户给的是知识库 URL，优先解析 space_id 或直接使用 URL 中的 space id。

```bash
lark-cli wiki +node-get --as user --node-token "<wiki_or_doc_url>" --format json
lark-cli wiki +node-list --as user --space-id "<space_id>" --format json --page-all --page-limit 10
```

知识库操作优先显式使用 `--as user`。

### Step 2: 定位会议

如果用户给时间段，先用系统 `date` 命令确认年份和日期，不要心算日期。

```bash
date '+%Y-%m-%d %Z'
lark-cli vc +search --as user --start "<YYYY-MM-DD>" --end "<YYYY-MM-DD>" --format json --page-size 30
```

如果结果有分页，继续带 `--page-token` 查询，直到没有更多数据。

如果用户直接给会议 ID 或链接，提取 meeting_id。

如果用户直接给妙记 URL，提取 URL 末尾 minute_token。

### Step 3: 获取 minute_token

会议 ID 需要先转成 minute_token：

```bash
lark-cli vc +recording --as user --meeting-ids "<meeting_id_1>,<meeting_id_2>" --format json
```

直接给 minute_token 的场景跳过本步。

### Step 4: 获取逐字稿

用 minute_token 获取产物和下载原始逐字稿：

```bash
lark-cli vc +notes --as user --minute-tokens "<minute_token_1>,<minute_token_2>" --format json
```

该命令会把原始逐字稿下载到：

```text
minutes/<minute_token>/transcript.txt
```

如果缺 scope，按提示用 split-flow 授权，不要阻塞当前轮：

```bash
lark-cli auth login --scope "minutes:minutes:readonly minutes:minutes.artifacts:read minutes:minutes.transcript:export" --no-wait --json
```

拿到 `verification_url` 后必须用 `lark-cli auth qrcode` 生成并展示二维码；用户授权后再执行：

```bash
lark-cli auth login --device-code "<device_code>"
```

### Step 5: 整理内容

按会议开始时间排序所有 minute_token。读取每个 `transcript.txt`。

确定父文档标题日期：取第一段会议开始时间，格式化为 `YY.MM.DD`。父文档标题必须是：

```text
YY.MM.DD <会议主题>
```

生成两份内容：

#### 总结纪要文档

固定结构：

```markdown
# 总结纪要：<会议主题>

> 整理范围：<时间范围>。本文不使用飞书妙记智能纪要，而是基于逐字稿合并整理。

## 会议信息

- 主题：
- 形式：同一场长会议分 <N> 段录制
- 时间：
- 参会人：
- 原始妙记：
  - [第一段：<标题>](<minute_url>)
  - [第二段：<标题>](<minute_url>)
- 逐字稿：[查看逐字稿](<wiki_url>)

## 会议大纲

### 一、<合并后的主题 1>

- ...

### 二、<合并后的主题 2>

- ...

## 待办任务

- [ ] ...
- [ ] ...
```

整理要求：

- 大纲按实际讨论主题合并，不要按“第一段/第二段/第三段”机械分章。
- 只保留有行动价值或决策价值的内容。
- 待办要尽量包含对象、动作和交付物；不确定负责人时不要编造。

#### 逐字稿文档

固定结构：

````markdown
# 逐字稿：<会议主题>

> 整理范围：同一场会议分 <N> 段飞书录制。每段均包含“AI 优化语意逐字稿”和“妙记原文”。

- [第一段原始妙记](<minute_url>)
- [第二段原始妙记](<minute_url>)

## 第一段：<会议标题>（<开始时间>，约 <时长>）

### AI 优化语意逐字稿

**说话人**：...

### 妙记原文

```text
<完整 transcript.txt 内容>
```

## 第二段：...
````

整理要求：

- AI 优化语意逐字稿要保留说话人和关键发言顺序。
- 允许合并连续碎片、删除明显重复、修正 ASR 错字，但不要改写事实。
- 妙记原文必须完整保留飞书导出的文本，不要删减。

### Step 6: 写入知识库

先检查目标知识库是否已有相同主题父文档或两个子文档。若已有，优先复用并覆盖更新对应内容；若没有，创建：

```bash
lark-cli docs +create --api-version v2 --as user --title "YY.MM.DD <会议主题>" --content "<title>YY.MM.DD <会议主题></title>"
lark-cli wiki +move --as user --obj-token "<docx_token>" --obj-type docx --target-space-id "<space_id>"
```

创建两个子文档或复用已有子文档：

```bash
lark-cli docs +create --api-version v2 --as user --title "总结纪要：<会议主题>" --content "<title>...</title>"
lark-cli wiki +move --as user --obj-token "<summary_docx_token>" --obj-type docx --target-space-id "<space_id>" --target-parent-token "<parent_node_token>"

lark-cli docs +create --api-version v2 --as user --title "逐字稿：<会议主题>" --content "<title>...</title>"
lark-cli wiki +move --as user --obj-token "<transcript_docx_token>" --obj-type docx --target-space-id "<space_id>" --target-parent-token "<parent_node_token>"
```

更新正文时优先使用 Markdown 文件传参，避免 shell 转义和长度限制：

```bash
lark-cli docs +update --api-version v2 --as user --doc "<summary_docx_token>" --command overwrite --doc-format markdown --content @summary.md
lark-cli docs +update --api-version v2 --as user --doc "<transcript_docx_token>" --command overwrite --doc-format markdown --content @transcript.md
```

`--content @file` 要求文件路径是当前工作目录下的相对路径；必要时切换到临时目录再执行。

### Step 7: 验证

写入完成后必须验证：

```bash
lark-cli wiki +node-list --as user --space-id "<space_id>" --parent-node-token "<parent_node_token>" --format json
lark-cli docs +fetch --api-version v2 --as user --doc "<summary_docx_token>" --format json --jq '.data.document.content | contains("不使用飞书妙记智能纪要")'
lark-cli docs +fetch --api-version v2 --as user --doc "<transcript_docx_token>" --format json --jq '.data.document.content | contains("妙记原文")'
```

最终回复用户父文档、总结纪要子文档、逐字稿子文档链接，并说明已验证的关键点。

## 权限

| 操作 | 所需 scope |
|------|-----------|
| 搜索历史会议 | `vc:meeting.search:read` |
| 查询会议录制 | `vc:record:readonly` |
| 读取会议/妙记产物 | `vc:note:read`, `minutes:minutes:readonly`, `minutes:minutes.artifacts:read`, `minutes:minutes.transcript:export` |
| 读取/创建/更新文档 | 文档相关 user 授权，通常可用 `lark-cli auth login --domain drive` |
| 读取/移动知识库节点 | `wiki:node:read`, `wiki:node:retrieve`, `wiki:node:move` |

遇到缺 scope 时，按 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md) 的 user 身份增量授权流程处理。
