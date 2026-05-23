---
name: ai-usage-report
slug: ai-usage-report
version: 3.3.5
agent_created: true
description: "当用户要生成 AI 使用周报、统计本周 Agent 工作产出，或配置每周自动周报时使用。"
---

# AI 使用周报 Skill

## 触发方式

用户说“执行 AI 使用周报”时，生成本周 AI 使用周报；人工模式下可继续进入 `qq-mail` 二阶段确认发送，automation 模式只生成文件。

---

## 执行红线（最高优先级）

以下规则高于后文所有流程。任一违反，视为生成失败，必须重算；仍不合规则停止对应动作。

1. **邮件主题只能是**：`[AI Weekly Report] {START_DATE} - {USER_WXID}`。不得使用中文标题、日期范围、报告 h1、`to`、管道符或任何额外前后缀。
2. **邮件正文必须 inline**：`body` 必须等于刚生成的 `.md` 周报全文；`attachments` 不传或只能是 `[]`。
3. **任务清单只能来自 Step 4 实际采集结果**。样例不是数据源，不得复制、改写、计入最终周报。
4. **任务首行必须是五标签定长行**：`**场景** ... | **产出** ... | **迭代** ... | **批量** ... | **工作空间** ...`。
5. **发送前必须执行格式校验 gate**。subject/body/attachments/task format 任一不通过，不得进入邮件发送。
6. **任何数据源失败都不得中断整体流程**。必须降级继续，并在报告中显式说明。

---

## 执行流程

### Step 0：运行前准备

#### 0.1 识别 host 与安装副本

探测以下两个安装路径，记录实际存在的路径到 `SKILL_PATHS`：

```bash
WB_SKILL="$HOME/.workbuddy/skills/ai-usage-report/SKILL.md"
CB_SKILL="$HOME/.codebuddy/skills/ai-usage-report/SKILL.md"
SKILL_PATHS=()
[ -f "$WB_SKILL" ] && SKILL_PATHS+=("$WB_SKILL")
[ -f "$CB_SKILL" ] && SKILL_PATHS+=("$CB_SKILL")

if [ -f "$CB_SKILL" ] && ! [ -f "$WB_SKILL" ]; then
  SKILL_HOST=codebuddy
else
  SKILL_HOST=workbuddy
fi
```

若两个副本都存在，默认 `SKILL_HOST=workbuddy`。不得主动创建另一端不存在的副本。

#### 0.2 校准时间

必须用系统命令取当前日期，不得凭印象填写：

```bash
date "+%Y-%m-%d %A %H:%M"
```

统计周期固定为：上周六 00:00 — 本周五 23:59。变量格式：

```text
START_DATE=YYYY-MM-DD
END_DATE=YYYY-MM-DD
TODAY=YYYY-MM-DD
```

`END_DATE` 必须 clamp 到不晚于今天：

```bash
TODAY=$(date "+%Y-%m-%d")
if [[ "$END_DATE" > "$TODAY" ]]; then
  END_DATE="$TODAY"
fi
```

#### 0.3 OTA 版本检查

按顺序尝试两个 OTA 源，每个源 `--connect-timeout 5 --max-time 10`：

1. 工蜂：`https://git.woa.com/ericqcsun/ai-usage-report/raw/main/SKILL.md`
2. GitHub：`https://raw.githubusercontent.com/zsutomato/ai-usage-report/main/SKILL.md`

远端内容必须同时满足：

- 第一行是 `---`
- 正文包含 `**当前版本**` 行

不满足立即判定该源不可用，继续下一个源。

若远端版本更新，必须覆盖 `SKILL_PATHS` 中所有已存在副本，并输出以下固定提醒块；不得省略、不得改写成一句话、不得埋在日志里：

```text
⚠️ 检测到 AI 使用周报 Skill 有新版本
- 当前加载版本：v{LOCAL_VER}
- 远端最新版本：v{REMOTE_VER}
- 已更新本地安装副本：{UPDATED_COUNT} 个
- 注意：当前这次运行仍使用旧版 prompt，只有重新触发后才会使用新版规则

请重新执行：
/ai-usage-report
```

人工模式下：更新完成后必须停止当前流程，不得继续生成周报。原因：当前 session 已加载旧版 prompt，继续执行会让用户误以为新版规则已在本次生效。

automation 模式下：不得中断；继续用当前已加载版本生成报告，但必须在报告末尾附注：

```text
⚠️ 本次运行开始时检测到 Skill 新版本 v{REMOTE_VER}，已更新本地副本；由于当前 session 已加载 v{LOCAL_VER} prompt，本次仍按 v{LOCAL_VER} 执行，下次运行生效。
```

---

### Step 1：识别身份与发送配置

#### 1.1 读取字段

按 host 优先，再跨端兜底：

| host | 优先读取 | 兜底读取 |
|------|----------|----------|
| `workbuddy` | `~/.workbuddy/USER.md` → `IDENTITY.md` → `SOUL.md` | `~/.codebuddy/CODEBUDDY.md` |
| `codebuddy` | `~/.codebuddy/CODEBUDDY.md` | `~/.workbuddy/USER.md` → `IDENTITY.md` → `SOUL.md` |

提取字段：

- `USER_NAME`：姓名，缺失写“未识别”
- `USER_WXID`：企业微信 ID，缺失时用 `whoami` 兜底，并在报告附注
- `REPORT_TO_EMAIL`：周报接收人邮箱，缺失则邮件发送跳过
- `REPORT_CC_EMAILS`：额外收件人列表；`无` / `none` / 空字符串统一视为空列表

#### 1.2 首次人工运行补齐

automation 模式禁止提问，直接降级。

人工模式下：

- 缺 `USER_WXID`：询问一次企业微信 ID。
- 缺 `REPORT_TO_EMAIL`：必须使用以下原文询问，不得改写：

> ⚠️ 请输入**周报接收人**的邮箱（一般是你的上级管理者）。**请勿输入你自己的邮箱地址**——发件身份由 QQ Mail Connector 自动处理，你这里要填的是“周报要发给谁”。例如：`manager@example.com`

- 仅首次补齐 `REPORT_TO_EMAIL` 时，可追问一次额外收件人：

> 需要同时发给其他人吗？如果需要，请输入**其他收件人**的邮箱（同样是收件人，不是你自己的发件邮箱），多个用逗号分隔；如果不需要，回复“无”。

若 `REPORT_TO_EMAIL` 已存在但 `REPORT_CC_EMAILS` 缺失，不得追问，直接按无额外收件人处理。

#### 1.3 写回位置

只写 host 对应官方位置：

- `workbuddy`：更新 `~/.workbuddy/USER.md`
- `codebuddy`：更新 `~/.codebuddy/CODEBUDDY.md` 的 `## CodeBuddy Added Memories` 段落

不得跨端创建另一端配置文件。已有字段必须更新旧值，不得重复追加。

---

### Step 2：采集数据

#### 2.1 SQLite session 索引

纳入存在 `codebuddy-sessions.vscdb` 的数据源：

| 产品 | macOS 默认路径 |
|------|----------------|
| WorkBuddy | `~/Library/Application Support/WorkBuddy/codebuddy-sessions.vscdb` |
| CodeBuddy CN | `~/Library/Application Support/CodeBuddy CN/codebuddy-sessions.vscdb` |
| CodeBuddy 国际版 | `~/Library/Application Support/CodeBuddy/codebuddy-sessions.vscdb` |

硬规则：

- schema 一致：`ItemTable.value` 是 JSON，含 `conversationId / cwd / userId / title / status / createdAt / updatedAt`。
- 必须用 Python 解析 JSON 与过滤时间；`sqlite3` CLI 只能读库，不能替代 Python。
- 时间过滤优先用 `updatedAt / lastUpdatedAt / updated_at`，全部缺失才回退 `createdAt`。
- 多源按 `conversationId` 去重，统计 `ACTIVE_PRODUCTS`、产品对话数、`SESSION_CWDS`。
- 单源失败只跳过该源；全部失败则继续 memory 层，并在报告附注 `⚠️ session 索引层未采集成功，本次统计可能偏低`。

#### 2.2 memory 日志

候选 workspace：

```text
CANDIDATE_CWDS = SESSION_CWDS ∪ sibling 扫描命中目录
```

`sibling 扫描`只能检查当前 workspace 父目录下一层直接子目录；若子目录中 `.workbuddy/memory/` 或 `.codebuddy/memory/` 存在统计周期内 `YYYY-MM-DD.md`，则纳入。禁止递归更深目录。

对每个候选 workspace 扫描：

```text
{cwd}/.workbuddy/memory/YYYY-MM-DD.md
{cwd}/.codebuddy/memory/YYYY-MM-DD.md
```

只读取统计周期内文件；按文件绝对路径去重；提取日期、产出、工作空间。若 sibling 有贡献，报告附注：`ℹ️ memory 层已按父目录下一层扩展扫描 sibling workspace`。

#### 2.3 历史对话搜索补充（可选增强）

WorkBuddy 的历史对话搜索用户入口是 `/search` slash command；历史版本中也曾称为 `conversation_search`。**不得假设它一定是 agent 可直接调用的工具**。

执行前必须完成能力探测，按以下分支处理：

1. **若当前运行时暴露 agent-callable 的历史搜索工具**（例如 `conversation_search` 或等价工具）：并发或分批执行 5 个查询，不得单个串行。
2. **若只存在用户侧 `/search` slash command，且搜索结果没有进入当前 agent 上下文**：不得假装已搜索；数据来源标记 `tier3 unavailable (history search not agent-callable)`。
3. **若用户手动执行 `/search`，且搜索结果实际出现在当前 agent 上下文**：才允许把这些结果作为第三层输入，并在数据来源写 `history-search-user-provided`。
4. **若工具存在但 5 个查询全部失败**：数据来源标记 `tier3 all-failed`，报告末尾附首条失败 `message`。

推荐查询词：

1. `本周完成的工作和任务`
2. `开发、修复、文档、部署、设计`
3. `会议、讨论、评审、分析`
4. `development, fix, deploy, review, document`
5. `meeting, discussion, analysis, design`

若工具支持日期参数，必须使用：

```text
start_date = START_DATE
end_date = END_DATE
limit = 10
```

日期格式只能是 `YYYY-MM-DD`；`END_DATE` 不得晚于 `TODAY`。

**红线**：禁止因为 `/search` 功能存在、或用户输入过 `/search ...`，就把第三层写成已执行。只有实际拿到可供 agent 读取的搜索结果，才能把数据来源写为 `conversation_search` / `history-search-user-provided`。

#### 2.4 合并口径

- SQLite 提供时间线、对话标题、产品拆分和 workspace 列表。
- memory 是任务事实的主来源。
- 历史对话搜索只补充上下文，不得覆盖 memory 中的明确事实。
- 最终按“可交付产出”聚合，进入 Step 3。

---

### Step 3：生成周报文件

#### 3.1 输出位置

```text
目录：~/Downloads/ai-usage-report/
文件名：{USER_WXID}-{START_DATE}.md
```

目录不存在必须创建。文件必须是 Markdown，正文从 frontmatter `---` 开始。

#### 3.2 文件模板

```markdown
---
姓名：{USER_NAME 或 未识别}
企业微信 ID：{USER_WXID}
周次：{START_DATE} Sat — {END_DATE} Fri
生成时间：{YYYY-MM-DD HH:mm}
Skill 版本：v3.3.5
使用模型：{识别不到写 未识别}
数据来源产品：{workbuddy / codebuddy / workbuddy + codebuddy / 无}
数据来源：{sessions / memory-session-cwds / memory-expanded-siblings / conversation_search / history-search-user-provided / tier3 unavailable / tier3 all-failed 等实际组合}

【本周完成的任务】
{真实任务清单；无事实则写“未检测到本周任务记录”}

【使用的能力】
{写代码 · 写文档 · 搜索资料 · 数据处理 · PPT/PDF · 浏览器自动化 · 定时任务 · 图片生成 · 知识库管理 · 代码审查 · 其他}

【本周对话统计】
- 总对话数：{合并后总数；多产品时附拆分}
- 活跃工作空间：{去重列表；无则写 无}
- 活跃天数：{去重天数}

【未完成 / 中途放弃的任务】
{没有则写 无}
---
```

#### 3.3 任务五标签格式

每条任务第一行必须严格匹配：

```text
**场景** {场景} | **产出** {产出} | **迭代** {数字或 ~数字} | **批量** {数字或 —} | **工作空间** {workspace}
```

字段规则：

| 字段 | 必须满足 |
|------|----------|
| **场景** | 中文名词短语，≤25 字，不得空 |
| **产出** | 真实可交付成果，多个用顿号，不得空 |
| **迭代** | 只计有效文件变更/改稿轮；纯数字或 `~数字` |
| **批量** | 必须有可数实证；无批量写 `—` |
| **工作空间** | 来自实际扫描结果，不得编造 |

深度任务（迭代 ≥ 20、批量 ≥ 100、或产出 ≥ 3 项）必须附 3-N 行自由叙述；轻量任务可以省略第二行。

#### 3.4 样例隔离硬规则

以下格式样例只用于学习排版，**不是数据源，不是真实任务，不得进入最终周报**。

禁止复制、改写或复用样例中的任务名、workspace、批量数、迭代数、产出内容。最终任务清单只能来自 Step 2 实际采集结果。若 Step 2 未采集到任务事实，必须写“未检测到本周任务记录”，不得用样例填充。一旦最终报告中出现样例内容，视为生成失败，必须重算。

```markdown
**场景** {真实场景} | **产出** {真实交付物} | **迭代** {真实迭代数} | **批量** {真实数量或 —} | **工作空间** {真实 workspace}
```

#### 3.5 任务聚合口径

- 1 条任务 = 1 个独立可交付产出。
- 禁止按 session 数拆任务；session 数只进入【本周对话统计】。
- 同一项目跨天、跨产品、多次会话继续推进，必须合并为 1 条任务。
- 批量同质工作合并为 1 条，用 `**批量**` 表示规模。
- 多语言、多版本、多轮改稿属于同一交付物时，必须合并为 1 条。
- 闲聊、环境配置、单次工具调用、一闪而过的实验，不计入任务清单。

#### 3.6 任务格式自检 gate

生成文件后，必须检查【本周完成的任务】：

- 若有任务，每条任务首行必须包含 5 个标签，顺序固定，分隔符固定为 ` | `。
- 禁止出现样例占位符：`{真实场景}`、`{真实交付物}`、`{真实 workspace}`。
- 禁止出现未被 Step 2 采集到的 workspace、批量数或任务名。
- 不通过则重算任务清单；仍不通过则保留文件但标注 `⚠️ 任务清单格式校验失败，未发送邮件`，并停止邮件发送。

---

### Step 4：输出与邮件发送

#### 4.1 始终输出文件

必须先告知 `.md` 文件绝对路径，并展示/回传文件。随后说明本次是否进入邮件发送。

#### 4.2 发送前置条件

只有同时满足以下条件，才允许进入 `qq-mail`：

- 人工交互模式（非 automation）
- `REPORT_TO_EMAIL` 非空
- `qq-mail` 可调用
- `GetMe` 成功，且拿到至少一个 alias
- Step 3 任务格式自检通过

不满足则只生成文件并说明原因。automation 模式固定跳过发送。

#### 4.3 唯一允许的发送参数

```text
alias_id = primary alias；多个 primary 取第一个
subject = "[AI Weekly Report] {START_DATE} - {USER_WXID}"
body = 读取刚生成的 .md 文件全文
body_format = "PLAIN"
attachments = 不传；若连接器强制要求该字段，只能传 []
cc = 仅当 REPORT_CC_EMAILS 非空时传入
```

`REPORT_CC_EMAILS` 只代表额外收件人；不得承诺标准 CC 语义。

#### 4.4 邮件格式校验 gate

Phase 1 `SendMessage` 前，必须执行等价于以下逻辑的校验。不通过就重算；仍不通过则停止发送。

```python
import re

subject_re = r"^\[AI Weekly Report\] \d{4}-\d{2}-\d{2} - [a-zA-Z0-9_.-]+$"
forbidden = ["请见附件", "详见附件", "查看附件", "附件中", "see attachment", "attached report"]

assert re.fullmatch(subject_re, subject), subject
assert body.startswith("---")
assert body == open(report_path, encoding="utf-8").read()
assert attachments in (None, [])
assert not any(x in body for x in forbidden)
```

校验通过后，确认预览必须展示：

```text
✅ 主题：{subject}（regex 通过）
✅ 正文：inline，等于周报 Markdown 全文
✅ attachments：未传或 []
✅ 附件文案：未发现
```

#### 4.5 Phase 1 / Phase 2

Phase 1：调用一次 `SendMessage`，不得带 `confirmation_token`。预期拿到 `428 / 42801`、`operation_summary`、`confirmation_token`。

向用户展示：

```text
即将发送，请确认以下信息：
- 主题：`{subject}`
- 发件：`{alias}`
- 收件：`{REPORT_TO_EMAIL}`
- 额外收件人：`{REPORT_CC_EMAILS 或 无}`

如确认发送，请回复“确认发送”。
```

Phase 2：只有用户明确回复 `确认发送`、`确认` 或 `发送`，才允许继续。用与 Phase 1 完全相同的参数重试，只额外加 `confirmation_token`。成功后告知“邮件已进入发送队列”。

---

## 更新此 Skill

OTA 源：

- 工蜂：`https://git.woa.com/ericqcsun/ai-usage-report`
- GitHub：`https://github.com/zsutomato/ai-usage-report`

手动更新：

```bash
mkdir -p ~/.workbuddy/skills/ai-usage-report && curl -sL "https://raw.githubusercontent.com/zsutomato/ai-usage-report/main/SKILL.md" -o ~/.workbuddy/skills/ai-usage-report/SKILL.md
mkdir -p ~/.codebuddy/skills/ai-usage-report && curl -sL "https://raw.githubusercontent.com/zsutomato/ai-usage-report/main/SKILL.md" -o ~/.codebuddy/skills/ai-usage-report/SKILL.md
```

---

## Changelog

- **v3.3.5** (2026-05-24)：补齐 `agent_created: true` frontmatter，便于 WorkBuddy skill 管理能力识别；压缩 INSTALL.md 历史版本段，减少安装文档噪音。
- **v3.3.4** (2026-05-23)：加固 OTA 新版本提醒契约：检测到远端新版本时必须输出固定提醒块；人工模式停止并要求重新触发，automation 模式继续但在报告附注本次仍使用旧版 prompt。
- **v3.3.3** (2026-05-23)：修正第三层历史搜索语义：`conversation_search`/`/search` 仅作为可选增强，必须先确认 agent 可读取搜索结果；未暴露时标记 `tier3 unavailable (history search not agent-callable)`。
- **v3.3.2** (2026-05-23)：进一步聚焦 description 和主执行文档，统一强约束语气；新增样例隔离硬规则与任务格式自检 gate，明确 few-shot 绝不能计入实际任务。
- **v3.3.1** (2026-05-23)：新增顶部执行红线，压缩 Step 4，重写 Step 6 为参数构造 → 格式校验 gate → Phase 1 → Phase 2。
- **v3.3.0** (2026-05-03)：新增任务清单五标签定长行契约。
- **v2.5.2** (2026-05-01)：强制邮件正文 inline，禁止 `.md` 作为 attachments。
- **v2.5.1** (2026-04-30)：修复 `conversation_search` 未来 `end_date` HTTP 400；第三层全失败显式降级。
- **v2.5.0** (2026-04-28)：支持 WorkBuddy / CodeBuddy 双端运行与 OTA 同步。
- **v2.4.2** (2026-04-28)：统一邮件主题为 `[AI Weekly Report] {YYYY-MM-DD} - {USER_WXID}`。
- **v2.4.0** (2026-04-27)：采集扩展到 WorkBuddy + CodeBuddy CN + CodeBuddy 国际版。
- **v2.3.0** (2026-04-27)：用活跃时间过滤 session，并用 sibling memory 兜回老 session 项目。
- **v2.0** (2026-04-10)：建立 SQLite → memory → conversation_search 三层采集模式。
- **v1.0** (2026-03-27)：初始版本。

**当前版本**：v3.3.5
**最后更新**：2026-05-24
**维护人**：QC
