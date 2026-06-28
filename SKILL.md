---
name: ai-usage-report
slug: ai-usage-report
version: 3.5.2
agent_created: true
description: "当用户要生成 AI 使用周报、统计本周 Agent 工作产出，或配置每周自动周报时使用。"
---

# AI 使用周报 Skill

## 触发方式

用户说“执行 AI 使用周报”时，生成本周 AI 使用周报；人工模式下可继续二阶段确认发送，automation 模式只生成文件。

**发送通道双轨并存（v3.5.2 新增）**：支持两种发送方式，由 `SEND_CHANNEL` 决定，**默认 `qq-mail`**（向后兼容，与历史行为一致）：
- `qq-mail`：通过 QQ Mail Connector 发送（默认，官方原生方式）。
- `agent-mail`：通过 Agently Mail CLI（`agently-cli`，发件身份 `xxx@agent.qq.com`）发送。

首次使用本版本（未记录过 `SEND_CHANNEL`）时，人工模式必须按 Step 0.4 弹出一次性选择；automation 模式直接用默认 `qq-mail`。

---

## 执行红线（最高优先级）

以下规则高于后文所有流程。任一违反，视为生成失败，必须重算；仍不合规则停止对应动作。

1. **邮件主题只能是**：`[AI Weekly Report] {START_DATE} - {USER_WXID}`。不得使用中文标题、日期范围、报告 h1、`to`、管道符或任何额外前后缀。（两种通道一致）
2. **邮件正文必须 inline**：正文必须等于刚生成的 `.md` 周报全文；不得以附件形式发送。（qq-mail 用 `body`；agent-mail 用 `--body-file` 指向周报 md，二者等效 inline）
3. **任务清单只能来自 Step 2 实际采集结果**。样例不是数据源，不得复制、改写、计入最终周报。
4. **任务首行必须是五标签定长行**：`**场景** ... | **产出** ... | **迭代** ... | **批量** ... | **工作空间** ...`。
5. **发送前必须执行格式校验 gate**。subject/body/task format 任一不通过，不得进入邮件发送。（两种通道共用同一 gate）
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

统计周期固定为：**上周六 00:00 — 本周五 23:59**。必须用以下确定性算法计算，禁止人工推算或硬编码星期缩写：

```bash
TODAY=$(date "+%Y-%m-%d")
DOW=$(date "+%u")  # 1=Mon ... 7=Sun
# 上周六偏移量：Mon→-2, Tue→-3, ..., Sat→-7（跑上一周）, Sun→-1（昨天）
if [ "$DOW" = "7" ]; then
  SAT_OFFSET=1
else
  SAT_OFFSET=$((DOW + 1))
fi
START_DATE=$(date -j -v-${SAT_OFFSET}d "+%Y-%m-%d")
# 本周五 = START_DATE + 6 天
END_DATE=$(date -j -v+6d -f "%Y-%m-%d" "$START_DATE" "+%Y-%m-%d")
# 实际星期（用于模板输出，禁止硬编码 Sat/Fri）
START_WEEKDAY=$(date -j -f "%Y-%m-%d" "$START_DATE" "+%a")
END_WEEKDAY=$(date -j -f "%Y-%m-%d" "$END_DATE" "+%a")
```

验证规则（fail-loud，不通过则停止生成并报错）：

```bash
START_DOW=$(date -j -f "%Y-%m-%d" "$START_DATE" "+%u")
END_DOW=$(date -j -f "%Y-%m-%d" "$END_DATE" "+%u")
if [ "$START_DOW" != "6" ] || [ "$END_DOW" != "5" ]; then
  echo "❌ FATAL: date self-check failed — START_DATE=$START_DATE (expected Sat, got dow=$START_DOW), END_DATE=$END_DATE (expected Fri, got dow=$END_DOW)"
  # 停止生成，不得继续
fi
```

`END_DATE` 必须 clamp 到不晚于今天：

```bash
if [[ "$END_DATE" > "$TODAY" ]]; then
  END_DATE="$TODAY"
  END_WEEKDAY=$(date -j -f "%Y-%m-%d" "$END_DATE" "+%a")
fi
```

变量清单（后续步骤全部引用这些变量）：

```text
TODAY=YYYY-MM-DD
START_DATE=YYYY-MM-DD
END_DATE=YYYY-MM-DD
START_WEEKDAY=三个字母星期缩写（如 Sat）
END_WEEKDAY=三个字母星期缩写（如 Fri）
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

#### 0.4 发送通道选择（v3.5.2 新增，首次运行一次性弹出）

读取已保存的 `SEND_CHANNEL`（位置见 Step 1.1 配置文件，键名 `周报发送通道` / `SEND_CHANNEL`）。

**"无值"的精确定义**：配置文件中**根本不存在** `SEND_CHANNEL` / `周报发送通道` 字段（老用户场景：v3.5.1 及更早版本从未写过该字段），**或**字段存在但值为空字符串 / 非法值（既不是 `qq-mail` 也不是 `agent-mail`）。以上两种都视为"无值 = 首次"，必须按下面"无值"分支处理。

按以下分支判定：

- **本次用户消息中带有明确的切换指令**（如"切换到 Agent Mail 发送"/"改用 QQ 邮箱发送"/"换成 agent mail"等明确指定通道的表述）→ **无论是否已有值**，都按用户本次指定的通道覆盖 `SEND_CHANNEL` 并写回配置（见 1.3），然后继续；选到 agent-mail 时进入 0.5。这是唯一允许覆盖既有选择的入口。
- **已存在有效值**（`qq-mail` 或 `agent-mail`）且本次无切换指令 → **直接沿用该值，不再询问**，进入后续流程。
- **automation 模式且无值** → 静默采用默认 `qq-mail`，不提问、不写回。
- **人工模式且无值** → 这是「首次使用本版本」（含**从 3.5.1 升级上来的老用户首次运行**）。**必须弹出一次性选择**，用以下原文询问（不得改写）：

> 📮 本版本（v3.5.2）支持两种周报发送方式，请选择一种（选定后会一直沿用，之后想换随时跟我说"切换发送通道"即可）：
> 1. **QQ 邮箱**（默认）：通过 QQ Mail Connector 发送，无需额外安装。
> 2. **Agent Mail**：通过 Agently Mail CLI 用 `xxx@agent.qq.com` 发送；若未安装/未注册，我会引导你用微信扫码注册。
>
> 回复 `1` 或 `QQ邮箱` 选第一种；回复 `2` 或 `Agent Mail` 选第二种。

用户选择后：
- 选 1 → `SEND_CHANNEL=qq-mail`，**写回配置（见 1.3）**，继续。
- 选 2 → `SEND_CHANNEL=agent-mail`，**写回配置（见 1.3）**，然后进入 **0.5 Agent Mail 准备与微信扫码注册**。

**选定即固定（关键语义）**：一旦 `SEND_CHANNEL` 写回配置，后续每次运行都直接沿用该值、**不再弹窗询问**；只有当用户在某次对话里**明确给出切换指令**时，才覆盖为新通道并重新写回。绝不允许在用户未主动要求的情况下重复弹窗或自行更改已选通道。

#### 0.5 Agent Mail 准备与微信扫码注册（仅当 SEND_CHANNEL=agent-mail）

按以下顺序检查并就绪 Agently Mail CLI：

1. **检查 CLI 是否安装**：`which agently-cli`。
   - 未安装 → 执行 `npm install -g @tencent-qqmail/agently-cli`（官方 npm 包）。
2. **检查授权状态**：`agently-cli auth status`（macOS 下该命令读写 Keychain，若运行环境有沙箱隔离，需在沙箱外执行）。
   - 已 `logged_in: true` 且 `token_status: valid` → 就绪，继续后续流程。
   - 未授权 / token 失效 → 进入**微信扫码注册/授权**：
3. **微信扫码注册/授权**（注册与授权一体化，新用户首次即注册）：
   - 后台运行 `agently-cli auth login`（交互式长命令，background + pty）。
   - 从 stdout/stderr 提取它输出的**原始授权 URL**，视为不可修改的 opaque string（不要做任何 URL 编码/解码/拼接/加标点），用只含原始 URL 的代码块单独展示给用户，并**必须**附文案：
     > 请用**微信扫描**或点击以下链接完成 Agent Mail 注册/授权：
   - 用户在手机微信扫码 / 浏览器中完成后，命令会自动退出。
   - 失败或超时**不要重试**，直接把错误信息反馈给用户。
4. **验证**：`agently-cli +me`，取回 `xxx@agent.qq.com` 邮箱地址，告知用户“Agent Mail 已就绪，邮箱地址 xxx”。

> 注：截至当前版本，agent.qq.com 没有独立于 `auth login` 的注册接口；`auth login` 输出的 OAuth 链接即“微信扫码注册 + 授权”一体化入口，新用户首次扫码即完成注册。

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
- `SEND_CHANNEL`：周报发送通道，取值 `qq-mail`（默认）或 `agent-mail`；缺失时按 Step 0.4 处理

#### 1.2 首次人工运行补齐

automation 模式禁止提问，直接降级。

人工模式下：

- 缺 `USER_WXID`：询问一次企业微信 ID。
- 缺 `REPORT_TO_EMAIL`：必须使用以下原文询问，不得改写：

> ⚠️ 请输入**周报接收人**的邮箱（一般是你的上级管理者）。**请勿输入你自己的邮箱地址**——发件身份由发送通道自动处理，你这里要填的是“周报要发给谁”。例如：`manager@example.com`

- 仅首次补齐 `REPORT_TO_EMAIL` 时，可追问一次额外收件人：

> 需要同时发给其他人吗？如果需要，请输入**其他收件人**的邮箱（同样是收件人，不是你自己的发件邮箱），多个用逗号分隔；如果不需要，回复“无”。

若 `REPORT_TO_EMAIL` 已存在但 `REPORT_CC_EMAILS` 缺失，不得追问，直接按无额外收件人处理。

#### 1.3 写回位置

只写 host 对应官方位置：

- `workbuddy`：更新 `~/.workbuddy/USER.md`（含 `周报发送通道` / `SEND_CHANNEL` 字段）
- `codebuddy`：更新 `~/.codebuddy/CODEBUDDY.md` 的 `## CodeBuddy Added Memories` 段落

不得跨端创建另一端配置文件。已有字段必须更新旧值，不得重复追加。

---

### Step 2：采集数据

#### 2.1 Session 索引（三源探测，中心库优先）

> ⚠️ **存储格式演进（务必理解，否则必然漏报）**：WorkBuddy 当前版本已把会话与工作空间统一存入**中心库 `~/.workbuddy/workbuddy.db`**（SQLite），这是唯一权威、完整的数据源。早期版本曾用 `Application Support/{产品}/codebuddy-sessions.vscdb`，更早一版的过渡期还出现过 `~/.workbuddy/sessions/*.json` 进程心跳文件。三者的信息量天差地别：
> - **中心库 `workbuddy.db`**：`sessions` 表含完整会话历史（id/cwd/title/model/status/created_at/updated_at），`workspaces` 表含全量工作空间清单。**第一优先源。**
> - **旧 vscdb**：`ItemTable` 里有 conversationId/cwd/title，信息次之。
> - **JSON 心跳**：只有 `pid/cwd/lastHeartbeat`，描述「**此刻还活着的进程**」而非「**本周开过的所有会话**」。它是**进程快照，不是会话历史**——只能作为 cwd 的补充，**绝不能当主源**。
>
> 历史教训（v3.4.0 漏报事故）：把心跳快照当成会话历史源，又用「vscdb 文件 mtime」做短路判断（文件 mtime ≠ 库内 updated_at），且完全没探测中心库，导致工作空间漏报率高达 ~77%。本版（v3.5.0）以中心库为第一可信源彻底修复。

##### 2.1.1 数据源探测矩阵

按下表逐产品探测，**每个产品独立判断格式**，不得用一个产品的结果推断另一产品。探测顺序固定为 `central-db → legacy-vscdb → json-heartbeat`，**先命中先用，命中即停**：

| 产品 | ① 中心库（最优先） | ② 旧 vscdb | ③ JSON 心跳目录（仅补充） |
|------|---------------------|------------|---------------------------|
| WorkBuddy | `~/.workbuddy/workbuddy.db` | `~/Library/Application Support/WorkBuddy/codebuddy-sessions.vscdb` | `~/.workbuddy/sessions/` |
| CodeBuddy CN | `~/.codebuddy/codebuddy.db`、`~/Library/Application Support/CodeBuddy CN/codebuddy.db`（路径待产品确认，探测到即用） | `~/Library/Application Support/CodeBuddy CN/codebuddy-sessions.vscdb` | `~/.codebuddy/sessions/` |
| CodeBuddy 国际版 | `~/.codebuddy/codebuddy.db`、`~/Library/Application Support/CodeBuddy/codebuddy.db`（待确认，探测到即用） | `~/Library/Application Support/CodeBuddy/codebuddy-sessions.vscdb` | （路径未知，跳过） |

**探测口径**（按产品独立执行；按顺序短路求值，先命中先返回）：

1. **中心库存在且可读** → 该产品归类 `central-db`，走 2.1.2 流程。这是最权威的源，命中后**直接停止该产品的后续探测**。
2. 否则**旧 vscdb 文件存在**（**只判断文件存在/可读，不看 mtime**）→ 该产品归类 `legacy-vscdb`，走 2.1.3 流程。是否有本周数据由**库内时间字段**决定，不由文件 mtime 决定。
3. 否则若 JSON 心跳目录存在且至少有一个 `*.json` 在统计周期内 → 该产品归类 `json-heartbeat`，走 2.1.4 流程（仅补充 cwd）。
4. 否则该产品归类 `unavailable`，跳过且不附注（用户没装就不算缺失）。

> 🚫 **被废止的旧规则**：v3.4.0「旧 SQLite 文件 mtime 在统计周期内才走 SQLite 分支」的短路判断**已彻底删除**。文件 mtime 只用于「库文件是否存在/可读」，**绝不能用来判断库内是否有本周数据**——这是 v3.4.0 漏报的根因之一。

##### 2.1.2 central-db 分支（中心库，第一可信源）

**这是首选源，能一次性提供会话数 + 标题 + 工作空间 + 活跃天数 + 模型全部维度。**

Schema（WorkBuddy `workbuddy.db`，实测）：
- `sessions(id, cwd, user_id, title, custom_title, status, created_at, updated_at, deleted_at, model, last_activity_at, ...)`，时间字段为 **epoch 毫秒整数**。
- `workspaces(path, last_opened_at)`，`last_opened_at` 为 **epoch 毫秒整数**。

硬规则：
- 必须用 Python（`sqlite3` 模块）查询。时间过滤**一律走库内字段**，禁止用文件 mtime。
- **会话提取**（注意软删除过滤）：
  ```sql
  SELECT id, cwd, COALESCE(custom_title, title, '') AS title, updated_at, model
  FROM sessions
  WHERE updated_at >= {START_TS_MS} AND updated_at <= {END_TS_MS}
    AND (deleted_at IS NULL OR deleted_at = 0)
  ORDER BY updated_at DESC;
  ```
  - `START_TS_MS` = `START_DATE 00:00:00` 本地时区的 epoch 毫秒；`END_TS_MS` = `END_DATE 23:59:59` 本地时区的 epoch 毫秒。必须用 Python 由 `START_DATE`/`END_DATE` 计算，不得硬编码。
  - 累加 `conversationId(=id) / cwd / title / model` 到 `CONVERSATIONS_BY_PRODUCT[product]`、`SESSION_CWDS`。
  - 收集本周出现的 `model` 去重，供 frontmatter「使用模型」字段。
- **工作空间双表并集**（关键，补全「打开过但无会话」的空间）：
  ```sql
  -- 表 2：本周打开过的工作空间
  SELECT path FROM workspaces WHERE last_opened_at >= {START_TS_MS};
  ```
  - `WORKBUDDY_DB_WORKSPACES = {本周 sessions 的 cwd 去重} ∪ {workspaces.last_opened_at 落在周期内的 path}`，全部并入 `SESSION_CWDS`。
- 该产品归入 `ACTIVE_PRODUCTS`。
- 查询异常（库锁定 / 表不存在 / schema 不符）只跳过该产品的中心库源，**回退到 ② 旧 vscdb 继续探测**，并在报告附注 `ℹ️ {产品} 中心库读取失败（{原因}），已回退旧数据源`。

##### 2.1.3 legacy-vscdb 分支（旧格式，可拿对话条目）

Schema：`ItemTable.value` 是 JSON BLOB，含 `conversationId / cwd / userId / title / status / createdAt / updatedAt`。

硬规则：
- 必须用 Python 解析 JSON 与过滤时间；`sqlite3` CLI 只能读库，不能替代 Python。
- **时间过滤一律走 BLOB 内的时间字段**（优先 `updatedAt / lastUpdatedAt / updated_at`，全部缺失才回退 `createdAt`），按 `>= START_TS_MS AND <= END_TS_MS` 过滤。**不得用文件 mtime 短路。**
- 提取本周期内 conversation 列表：`conversationId / cwd / title / startedAt`。
- 累加到全局 `CONVERSATIONS_BY_PRODUCT[product]`、`SESSION_CWDS`。
- 该产品归入 `ACTIVE_PRODUCTS`。
- 若库内本周无任何会话，仍归 `legacy-vscdb`（库可读但本周空），对话数计 0，不附注缺失。

##### 2.1.4 json-heartbeat 分支（进程心跳，仅补充 cwd）

> 语义警告：心跳文件描述「此刻活着的进程」，不是「本周会话历史」。仅在中心库和旧 vscdb 都不可用时才走此分支，且**只用于补漏当前活跃进程的 cwd**。

Schema：每个 `*.json` 形如：
```json
{ "pid": ..., "cwd": "/abs/path", "version": "2.97.2",
  "startedAt": <ms>, "lastHeartbeat": <ms>, "updatedAt": <ms>, "kind": "interactive", ... }
```

硬规则：
- 必须用 Python 读取目录顶层 `*.json`（不递归子目录），按 `updatedAt`（回退 `lastHeartbeat`）过滤统计周期内文件。
- 仅提取 `cwd` 去重，累加到 `SESSION_CWDS`。
- **不得伪造 conversation 数**：`CONVERSATIONS_BY_PRODUCT[product]` 必须置 `None`，不得用 session 文件数充数。
- 该产品归入 `ACTIVE_PRODUCTS`，并在最终报告附注：
  > ℹ️ {产品} 未探测到中心库与可读 vscdb，本次仅用 session 心跳目录补充当前活跃进程的工作空间；完整会话与工作空间清单可能不全，任务清单依赖 memory 层。

##### 2.1.5 多源合并

- `SESSION_CWDS = ∪ all products`（按绝对路径去重）。
- `ACTIVE_PRODUCTS = {归类为 central-db / legacy-vscdb / json-heartbeat 的产品}`。
- 对话数仅累加 `central-db` 与 `legacy-vscdb` 分支的产品；JSON 心跳产品不参与对话数统计。
- 任一产品采集失败只跳过该产品；全部失败则继续 memory 层，并在报告附注 `⚠️ session 索引层未采集成功，本次任务清单完全依赖 memory 层`。

##### 2.1.6 数据来源标签合成

报告 frontmatter 的"数据来源"字段中 session 部分按以下规则填（取所有 `ACTIVE_PRODUCTS` 的分支集合）：
- 仅 central-db 分支产品 → `sessions-central-db`
- 仅 legacy-vscdb 分支产品 → `sessions-legacy-vscdb`
- 仅 JSON 心跳分支产品 → `sessions-json-heartbeat`
- 存在 ≥2 种不同分支 → `sessions-mixed`
- 所有产品都不可用 → `sessions-unavailable`

后续的 memory / 历史搜索来源标签（`memory-session-cwds` / `memory-expanded-siblings` / `conversation_search` 等）按既有逻辑追加，多个标签用逗号连接。

##### 2.1.7 低数量自检告警（防未来再漏报）

合并完成后，若最终识别出的**活跃工作空间数 ≤ 3** 或 **对话数 ≤ 2**（且数据源不是 `sessions-unavailable`），必须在周报正文最顶部加一条告警块，让用户第一时间发现可能漏报，而不是默默接受错误结果：

```text
⚠️ 本次仅识别到 {N} 个工作空间 / {M} 条会话，可能存在数据源探测不全。
已尝试的源：{命中分支列表，如 central-db / legacy-vscdb / json-heartbeat}。
如与实际不符，请检查中心库 ~/.workbuddy/workbuddy.db 是否可读。
```

若数据源本就是 `sessions-unavailable`（用户确实没装/无数据），不触发此告警。

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

- Session 索引提供时间线、workspace 列表，**central-db 与 legacy-vscdb 分支**还能提供对话标题、模型与产品对话数；JSON 心跳分支只贡献 cwd
- memory 是任务事实的主来源
- 历史对话搜索只补充上下文，不得覆盖 memory 中的明确事实
- 最终按"可交付产出"聚合，进入 Step 3

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
周次：{START_DATE} {START_WEEKDAY} — {END_DATE} {END_WEEKDAY}
生成时间：{YYYY-MM-DD HH:mm}
Skill 版本：v3.5.2
使用模型：{识别不到写 未识别}
数据来源产品：{workbuddy / codebuddy / workbuddy + codebuddy / 无}
数据来源：{sessions-central-db / sessions-legacy-vscdb / sessions-json-heartbeat / sessions-mixed / memory-session-cwds / memory-expanded-siblings / conversation_search / history-search-user-provided / tier3 unavailable / tier3 all-failed 等实际组合}

【本周完成的任务】
{真实任务清单；无事实则写“未检测到本周任务记录”}

【使用的能力】
{写代码 · 写文档 · 搜索资料 · 数据处理 · PPT/PDF · 浏览器自动化 · 定时任务 · 图片生成 · 知识库管理 · 代码审查 · 其他}

【本周对话统计】
- 总对话数：{合并 central-db 与 legacy-vscdb 分支产品的对话数；多产品时附拆分；JSON 心跳产品写明"未采集"。若所有产品都只有 JSON 心跳格式，写 `未采集（session 仅心跳格式可用，未探测到中心库）`}
- 活跃工作空间：{去重列表；无则写 无}
- 活跃天数：{去重天数}

混合模式（sessions-mixed）总对话数推荐写法：
```text
- 总对话数：{合并可采集产品的对话数；不可采集的产品逐项说明}
  - WorkBuddy: {N} 条（中心库）
  - CodeBuddy CN: {M} 条（旧 vscdb）
```

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

必须先告知 `.md` 文件绝对路径，并展示/回传文件。随后说明本次是否进入邮件发送、走哪条通道。

#### 4.2 发送前置条件（通道无关）

只有同时满足以下条件，才允许进入发送：

- 人工交互模式（非 automation）
- `REPORT_TO_EMAIL` 非空
- Step 3 任务格式自检通过
- 所选通道就绪（见各通道的额外前置条件）

不满足则只生成文件并说明原因。automation 模式固定跳过发送。

#### 4.3 邮件格式校验 gate（通道无关，发送前必做）

任何通道 Phase 1 前，必须执行等价于以下逻辑的校验。不通过就重算；仍不通过则停止发送。

```python
import re

subject_re = r"^\[AI Weekly Report\] \d{4}-\d{2}-\d{2} - [a-zA-Z0-9_.-]+$"
forbidden = ["请见附件", "详见附件", "查看附件", "附件中", "see attachment", "attached report"]
body = open(report_path, encoding="utf-8").read()

assert re.fullmatch(subject_re, subject), subject
assert body.startswith("---")
assert not any(x in body for x in forbidden)
```

校验通过后，确认预览必须展示：

```text
✅ 主题：{subject}（regex 通过）
✅ 正文：inline，等于周报 Markdown 全文
✅ 通道：{qq-mail / agent-mail}
✅ 附件文案：未发现
```

#### 4.4 通道 A：qq-mail（默认，SEND_CHANNEL=qq-mail）

额外前置条件：`qq-mail` 可调用；`GetMe` 成功，且拿到至少一个 alias。

唯一允许的发送参数：

```text
alias_id = primary alias；多个 primary 取第一个
subject = "[AI Weekly Report] {START_DATE} - {USER_WXID}"
body = 读取刚生成的 .md 文件全文
body_format = "PLAIN"
attachments = 不传；若连接器强制要求该字段，只能传 []
cc = 仅当 REPORT_CC_EMAILS 非空时传入
```

`REPORT_CC_EMAILS` 只代表额外收件人；不得承诺标准 CC 语义。

qq-mail 通道在 4.3 公共 gate 之外，Phase 1 前还必须额外满足以下两条断言（与 v3.5.1 一致，专属本通道）：

```python
assert body == open(report_path, encoding="utf-8").read()
assert attachments in (None, [])
```

Phase 1：调用一次 `SendMessage`，不得带 `confirmation_token`。预期拿到 `428 / 42801`、`operation_summary`、`confirmation_token`。

向用户展示：

```text
即将发送，请确认以下信息：
- 主题：`{subject}`
- 通道：QQ 邮箱
- 发件：`{alias}`
- 收件：`{REPORT_TO_EMAIL}`
- 额外收件人：`{REPORT_CC_EMAILS 或 无}`

如确认发送，请回复“确认发送”。
```

Phase 2：只有用户明确回复 `确认发送`、`确认` 或 `发送`，才允许继续。用与 Phase 1 完全相同的参数重试，只额外加 `confirmation_token`。成功后告知“邮件已进入发送队列”。

#### 4.5 通道 B：agent-mail（SEND_CHANNEL=agent-mail）

额外前置条件：`agently-cli` 已安装且可调用；`agently-cli auth status` 为 `logged_in: true` 且 `token_status: valid`（否则回到 Step 0.5 引导微信扫码注册/授权）。

> 执行环境注意：`agently-cli` 在 macOS 下读写 Keychain，若运行环境有沙箱隔离，所有 `agently-cli` 命令必须在沙箱外执行。

唯一允许的发送方式与参数（发件身份固定为已授权的 `xxx@agent.qq.com`，无需指定 alias）：

```text
agently-cli message +send \
  --to {REPORT_TO_EMAIL} \
  --subject "[AI Weekly Report] {START_DATE} - {USER_WXID}" \
  --body-file {周报 md 的相对路径}        # 注意：--body-file 只接受相对路径，须先 cd 到报告目录再用 ./{USER_WXID}-{START_DATE}.md
  [--cc {每个额外收件人一个 --cc，仅当 REPORT_CC_EMAILS 非空时}]
（不加任何附件参数；--body-file 即周报全文 inline）
```

Phase 1：执行上面的 `agently-cli message +send`（不带 `--confirmation-token`）。预期返回 `confirmation_required: true` + `summary` + `confirmation_token`（`ctk_xxx`，5 分钟有效）。

向用户展示：

```text
即将发送，请确认以下信息：
- 主题：`{subject}`
- 通道：Agent Mail
- 发件：`xxx@agent.qq.com`
- 收件：`{REPORT_TO_EMAIL}`
- 额外收件人：`{REPORT_CC_EMAILS 或 无}`

如确认发送，请回复“确认发送”。
```

Phase 2：只有用户明确回复 `确认发送`、`确认` 或 `发送`，才允许继续。用与 Phase 1 **完全相同**的参数 + `--confirmation-token ctk_xxx` 重试。

> ⚠️ 两阶段必须在同一条 shell 命令里串起来执行（Phase 1 提取 `ctk_xxx` → 立即 Phase 2 带 token），分轮单独调用易出现命令未落库导致漏发。

返回 `queued: true` 即成功。**建议随后核实**（本通道实践要点，非全局红线）：`agently-cli message +list --dir sent --limit 3`，确认已发送文件夹有该邮件记录；若 sent 中无此邮件则判定漏发并重发。核实通过后告知“邮件已进入发送队列”。

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

- **v3.5.2** (2026-06-29)：新增**双发送通道并存**。在保留官方默认 `qq-mail`（QQ Mail Connector）的同时，新增 `agent-mail`（Agently Mail CLI，`xxx@agent.qq.com`）作为可选发送通道，由 `SEND_CHANNEL` 控制，默认仍为 `qq-mail`（完全向后兼容，老用户行为不变）。新增 Step 0.4「首次运行一次性弹出发送通道选择」：明确"无值"含「字段不存在」（v3.5.1 升级上来的老用户首次运行即触发弹窗）与「空/非法值」两种；选定后写回配置并固定沿用，后续不再弹窗，仅当用户明确给出切换指令时才覆盖。新增 Step 0.5「Agent Mail 准备与微信扫码注册」（`agently-cli auth login` 输出的 OAuth 链接即微信扫码注册+授权一体化入口）。Step 4 重构为通道无关的前置/格式校验 gate + 通道 A（qq-mail，发送参数/断言/两阶段流程与 v3.5.1 完全一致）/ 通道 B（agent-mail，`agently-cli message +send --body-file` inline 正文 + 两阶段确认）。沉淀 agent-mail 三个实践要点：`--body-file` 仅接受相对路径、agently-cli 需在沙箱外运行、两阶段须同条命令串联以防漏发，并建议发送后 `+list --dir sent` 核实（仅 agent-mail 通道内部实践，非全局红线）。Step 1 新增 `SEND_CHANNEL` 配置字段，写回 USER.md / CODEBUDDY.md。**除新增 agent-mail 通道外，v3.5.1 的所有既有逻辑（host/时间/OTA/数据采集/周报生成/qq-mail 发送）保持不变。**
- **v3.5.1** (2026-06-22)：修复 Step 0.2 日期/星期计算 bug。加入确定性日期算法（`date -v` 从 TODAY 推算上周六/本周五），消除硬编码 Sat/Fri 模板字面量；新增星期自检 fail-loud（START_DATE 必须为周六、END_DATE 必须为周五，否则停止生成）；END_DATE clamp 后同步更新 END_WEEKDAY；frontmatter 周次行改用 `{START_WEEKDAY}`/`{END_WEEKDAY}` 变量。根因：v3.5.0 Step 0.2 只写"上周六—本周五"但无算法，Agent 在边界日运行时会生成错误星期标注（如 kasonkang 报告出现 `2026-06-22 Fri`，实际 06-22 是周一）。
- **v3.5.0** (2026-06-13)：修复 v3.4.0 工作空间漏报 P1 事故（漏报率曾达 ~77%）。Step 2.1 改为「中心库优先」三源探测：① `~/.workbuddy/workbuddy.db` 中心库（`sessions` + `workspaces` 双表，含会话数/标题/模型/工作空间全字段，第一可信源）→ ② 旧 vscdb（按库内时间字段过滤）→ ③ JSON 心跳（降级为仅补充 cwd 的补充源）。彻底删除「vscdb 文件 mtime 短路判断」（文件 mtime ≠ 库内 updated_at 是漏报根因）。工作空间采集改为 `sessions.cwd ∪ workspaces.path` 双表并集，补全「打开过但无会话」的空间。新增数据来源标签 `sessions-central-db` / `sessions-legacy-vscdb`；新增低数量自检告警（工作空间 ≤3 或会话 ≤2 时报警）。CodeBuddy CN/国际版预留中心库探测条目（路径待产品确认），探测到即用，否则回退旧 vscdb（当前 CodeBuddy CN 仍写 vscdb，无此问题）。
- **v3.4.0** (2026-05-30)：适配 WorkBuddy v2.97 起 session 存储从 SQLite 切换为 `~/.workbuddy/sessions/*.json` 心跳文件。Step 2.1 重写为「按产品独立探测」的双源采集：旧 SQLite 在则用 SQLite（拿全字段）、否则 fallback 到新 JSON 心跳（仅拿 cwd）。新增数据来源标签 `sessions-sqlite` / `sessions-json-heartbeat` / `sessions-mixed`；JSON 心跳分支不再伪造对话条目数。CodeBuddy CN 当前仍写旧 SQLite，自动按存在性走对应分支，未来 CodeBuddy CN 也切换时无需改 Skill。
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

**当前版本**：v3.5.2
**最后更新**：2026-06-29
**维护人**：QC
