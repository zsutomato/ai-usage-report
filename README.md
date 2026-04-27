# ai-usage-report

> AI 使用周报自动生成 Skill —— 一条命令装好，每周五自动帮你汇总本周在 WorkBuddy 里做了什么，并可选邮件发给你的上级。

**维护人**：QC（ericqcsun） · **当前版本**：见本仓 `SKILL.md` 的 `version` 字段 · **源码主仓**：[`ericqcsun/team-skills`](https://git.woa.com/ericqcsun/team-skills)

本仓库是 **Skill 的公开 OTA 源**（GitHub 公开仓库），供用户侧 `curl` 拉取 `SKILL.md` 即装即用。**当前 Skill 内置 OTA 默认就从本仓拉取**。请**勿直接在本仓修改 `SKILL.md`**——本仓由源码主仓 `team-skills` 的 `scripts/release-ai-usage-report.sh` 通过 GitHub Contents API 自动同步，手改会被下次发布吞掉。

---

## 它能干什么

自动回溯"本周"（上周六 00:00 → 本周五 23:59）所有 Agent 对话记录，生成一份标准格式的 Markdown 周报，包括：

- **本周完成的任务**（三层数据采集合并去重：SQLite session 索引 + 各工作空间 memory 日志 + conversation_search）
- **使用的能力**（写代码 / 写文档 / 搜索资料 / 数据处理 / PPT·PDF / 浏览器自动化 / 定时任务 / 图片生成 / 代码审查 / 其他）
- **本周对话统计**（总对话数、活跃工作空间、活跃天数）
- **使用模型**、**数据来源**、**Skill 版本**、**生成时间**等元信息

生成位置：`~/Downloads/ai-usage-report/{USER_WXID}-{统计起始日期}.md`

如果你在 WorkBuddy 里装过 [`qq-mail` Connector](https://workbuddy.cn)，**人工交互模式**下还会在周报生成后进入**两阶段确认发送流程**：先预览主题 / 收件人 / 正文 → 你回复"确认发送" → 邮件进入发送队列。**自动化（定时任务）模式默认只生成文件，不自动发信**（因为二阶段确认没法无人值守完成）。

---

## 安装（零依赖，一条命令搞定）

```bash
mkdir -p ~/.workbuddy/skills/ai-usage-report && \
curl -sL "https://raw.githubusercontent.com/zsutomato/ai-usage-report/main/SKILL.md" \
  -o ~/.workbuddy/skills/ai-usage-report/SKILL.md
```

> 📌 本仓就是 Skill OTA 默认拉取源，`curl` 免登录即可。Skill 源码在工蜂 `ericqcsun/team-skills` monorepo 里维护，本仓只同步最新 `SKILL.md`。

**验证安装成功：**

```bash
head -6 ~/.workbuddy/skills/ai-usage-report/SKILL.md
```

应该看到 `name: ai-usage-report` / `version: X.Y.Z` 等字段。

---

## 第一次运行（重要）

安装完直接对 WorkBuddy 说：

> 执行 AI 使用周报

**首次手动运行会问你 2~3 个问题，别跳过这一步**，否则定时任务会因为没身份识别而用系统用户名兜底。

1. **企业微信 ID**：例如 `ericqcsun`
2. **⚠️ 周报接收人邮箱**：**一般是你的上级管理者**，**请勿填你自己的邮箱**。发件身份由 QQ Mail Connector 自动处理，你这里填的是"周报要发给谁"。例如：`manager@example.com`
3. **额外收件人**（可选）：需要同时抄给其他人就填他们的邮箱（多个用逗号分隔），不需要就回复"无"

回答会被写入 `~/.workbuddy/USER.md` 持久化，之后（包括定时任务）都不会再问。

> ⚠️ 当前 `qq-mail` 即便接收 `cc` 参数，实际投递效果也可能把额外收件人地址并入 To。Skill 里统一称为"额外收件人"，**不承诺**标准邮件头里的 CC 语义。

---

## 设置成每周自动跑（强烈推荐）

走完第一次手动运行后，建议配置成每周五 18:00 自动生成：

> 请帮我创建一个自动化任务，名称"AI 使用周报"，prompt 为"执行 AI 使用周报"，每周五 18:00 执行。

然后你就躺平，每周五下班前周报文件自动躺在 `~/Downloads/ai-usage-report/` 里等你。**邮件发送仍然需要你人工"确认发送"**（这是 qq-mail 的两阶段确认硬要求，跳不了）。

---

## 升级到最新版

**默认情况下不用你管**——Skill 内置 OTA：每次你说"执行 AI 使用周报"时，Step 2 会 `curl` GitHub 最新版和本地版本比对，如果远端更新就自动覆盖本地并提示"Skill 已更新至 vX.Y.Z，建议重新触发以使用新版本"。当前会话继续用旧版跑完，**下次触发**才加载新版。

**必须手动升级的两种情况**：

1. **你装的是 v2.2.7 及更早版本**：那时候 OTA 版本正则只识别两段版本号（`v2.2`），会把 `v2.3.0` 误判成"已是最新版"，必须手动拉一次新版才能把 OTA 正则修好。之后就能正常吃 OTA。
2. **你想立刻用新版跑一次当前周报，不想等"下次"**。

手动升级命令：

```bash
curl -sL "https://raw.githubusercontent.com/zsutomato/ai-usage-report/main/SKILL.md" \
  -o ~/.workbuddy/skills/ai-usage-report/SKILL.md
```

然后**重开一个对话**再触发周报。

---

## 核心设计（给想深入的人）

### 三层数据采集 + 逐层降级

1. **SQLite session 索引**（`~/Library/Application Support/WorkBuddy/codebuddy-sessions.vscdb`）：提取本周活跃 session 的标题、工作空间路径（cwd）、对话数。v2.3.0 起按 `updatedAt`/`lastUpdatedAt` 优先过滤，保证"用老 session 继续聊"的项目也能被识别。
2. **工作空间 memory 日志**（各工作空间的 `.workbuddy/memory/YYYY-MM-DD.md`）：提取每天具体做了什么、产出了什么。v2.3.0 起候选工作空间 = `SESSION_CWDS ∪ sibling 扫描命中目录`，sibling 扩展扫描从"失败 fallback"升级为"常态补集"。
3. **conversation_search**：中英文各跑几轮关键词检索作为补充。

任何一层失败不中断，三层合并去重后生成最终清单。

### 身份识别四级回退

`~/.workbuddy/USER.md` → `IDENTITY.md` → `SOUL.md` → 首次运行询问并持久化 → `whoami` 兜底。automation 模式跳过"询问"这一级，防止定时任务卡死。

### 任务粒度口径

"1 条任务 = 1 个独立可交付产出"。批量同质工作合并为 1 条并注明规模（例："回复客户工单 · 搜索资料 · TAM 工作台（约 600 条）"）；多语言/多版本/多轮迭代合并为 1 条；同项目连续多日推进合并为 1 条；闲聊和一次性调用不计入。目标粒度一周 5–15 条。

---

## 兼容性

- **操作系统**：macOS / Windows / Linux（Step 4 按 `VSCODE_PORTABLE` / `VSCODE_APPDATA` / 平台默认路径推导数据库位置）
- **Python**：至少需要 `python` / `python3` / `py -3` 其中之一（用于 JSON 解析和时间过滤，`sqlite3` CLI 可选）
- **网络**：OTA 与 `conversation_search` 需要网络；网络不可达时跳过、附注、不中断

---

## Changelog 与版本页脚

详细的版本变更请看 `SKILL.md` 文件末尾的 `## Changelog` 段落与 `**当前版本**` / `**最后更新**` 字段。OTA 版本检测的正则就是匹配 `**当前版本**` 这一行，**修改时不要破坏这个格式**。

---

## 问题反馈 / 贡献

- 报 bug / 提需求：直接找 QC（ericqcsun）
- 改源码：在 [`ericqcsun/team-skills`](https://git.woa.com/ericqcsun/team-skills) 上改 `ai-usage-report/SKILL.md`，commit push 后跑 `./scripts/release-ai-usage-report.sh` 发三处
- 别直接 PR 本仓 —— 会被下次 release 脚本覆盖

