---
name: ai-usage-report
slug: ai-usage-report
version: 2.5.1
description: "自动回溯本周 Agent 对话记录，生成标准格式使用周报；若当前环境已配置 qq-mail Connector，则在手动模式下可进入确认发送流程。三层数据采集（SQLite → memory → conversation_search），支持定时任务无人值守生成。"
---

# AI 使用周报 Skill

## 功能说明

自动回溯本周所有 Agent 对话记录，生成标准格式的使用周报 Markdown 文件，用于统计 AI 工具使用情况；若当前环境已配置可用的 `qq-mail` Connector，则在人工交互模式下可继续进入邮件确认发送流程。

**适用场景**：每周五下班前手动执行，或设置为定时任务自动生成周报文件。

---

## 使用方法

直接对 Agent 说：

> 执行 AI 使用周报

Agent 会自动执行以下步骤；若进入 `qq-mail` 发送流程，则会在最终发信前等待一次用户确认。

---

## 执行流程

收到"执行 AI 使用周报"指令后，严格按以下步骤顺序执行：

### Step 0：识别运行时 host（v2.5.0 起）

**目的**：本 Skill 同时支持安装在 **WorkBuddy** 和 **CodeBuddy** 两个产品下运行。Step 0 用于探测当前 Skill 加载自哪一端，得到变量 `SKILL_HOST`（取值 `workbuddy` 或 `codebuddy`），供后续 Step 2（OTA 写回路径）、Step 3（身份识别优先级）使用。

**探测逻辑（按顺序）**：

1. 若 `~/.codebuddy/skills/ai-usage-report/SKILL.md` 存在且内容与当前执行中的 Skill 一致 → `SKILL_HOST = codebuddy`
2. 若 `~/.workbuddy/skills/ai-usage-report/SKILL.md` 存在且内容与当前执行中的 Skill 一致 → `SKILL_HOST = workbuddy`
3. 两个都存在、都一致（极罕见）→ 默认 `SKILL_HOST = workbuddy`（WorkBuddy 为 Skill 原生环境）
4. 都不存在（Skill 被直接以 adhoc 方式加载）→ 默认 `SKILL_HOST = workbuddy`，并附注 `⚠️ 无法识别 Skill 安装路径，已默认按 WorkBuddy host 运行`

**简化的 bash 探测脚本**：

```bash
WB_SKILL="$HOME/.workbuddy/skills/ai-usage-report/SKILL.md"
CB_SKILL="$HOME/.codebuddy/skills/ai-usage-report/SKILL.md"

if [ -f "$CB_SKILL" ] && ! [ -f "$WB_SKILL" ]; then
  SKILL_HOST=codebuddy
elif [ -f "$WB_SKILL" ] && ! [ -f "$CB_SKILL" ]; then
  SKILL_HOST=workbuddy
elif [ -f "$WB_SKILL" ] && [ -f "$CB_SKILL" ]; then
  # 两个都存在，默认 workbuddy（也可通过对比哪个 SKILL.md 更新来精细化判断）
  SKILL_HOST=workbuddy
else
  SKILL_HOST=workbuddy
fi
echo "SKILL_HOST=$SKILL_HOST"
```

同时记录下两个**实际存在**的 Skill 安装副本路径（供 Step 2 OTA 双端同步使用）：

```bash
SKILL_PATHS=()
[ -f "$WB_SKILL" ] && SKILL_PATHS+=("$WB_SKILL")
[ -f "$CB_SKILL" ] && SKILL_PATHS+=("$CB_SKILL")
```

### Step 1：校准当前时间

执行命令获取准确日期，**不得凭印象猜测**：

```bash
date "+%Y-%m-%d %A %H:%M"
```

根据输出结果计算本次统计周期：**上周六 00:00 — 本周五 23:59**。

将起止日期存为变量供后续步骤使用，格式为 `YYYY-MM-DD`。

**v2.5.1 起：对 end_date 做 clamp，确保不超过今天**（防止第三层 `conversation_search` 后端因未来日期 400）：

```bash
TODAY=$(date "+%Y-%m-%d")
# 若周五还没到（如周四下班提前触发、或周五的 0 点 automation），end_date 原本算的是"本周五"，可能晚于 today
# 后端对 end_date > today 会返回 HTTP 400，必须 clamp
if [[ "$END_DATE" > "$TODAY" ]]; then
  END_DATE="$TODAY"
fi
echo "统计周期：$START_DATE — $END_DATE（today=$TODAY）"
```

> **为什么必须 clamp**：2026-04-30（周四）的手测证明，`end_date=2026-05-01` 会让 `conversation_search` 后端返回 HTTP 400。V2 vs V3 对比实验（同一字不差的 query，只改日期参数就从 success 变 400）已排除了 query 形态和其他变量。Step 1 末尾 clamp 到 today 是最小代价的根治手段。

### Step 2：检查并更新 Skill 版本

**OTA 源策略**：Skill 支持两个 OTA 源并行，按顺序尝试，任一源拿到合法内容即停：

1. 工蜂（内网优先，未来主源）：`https://git.woa.com/ericqcsun/ai-usage-report/raw/main/SKILL.md`
2. GitHub（当前默认可匿名访问，兜底源）：`https://raw.githubusercontent.com/zsutomato/ai-usage-report/main/SKILL.md`

**为什么是这个顺序**：工蜂更贴近公司内网环境（低延迟、无需翻墙、未来可能承载更多团队 skill），优先使用；但当前工蜂 public 语义为"登录后内网可见"，匿名 `curl` 会被重定向到登录页返回 HTML，所以必须校验响应是否为合法 Markdown，失败就降级到 GitHub。哪天工蜂真正支持匿名 raw 了，这一段无需改动即自动生效。

① 读取本地版本号（使用 Step 0 探测到的当前 host 对应副本；若有多个副本存在则取其中一个，版本号应一致）：

```bash
# SKILL_HOST / SKILL_PATHS 由 Step 0 设置
case "$SKILL_HOST" in
  workbuddy) LOCAL_SKILL="$HOME/.workbuddy/skills/ai-usage-report/SKILL.md" ;;
  codebuddy) LOCAL_SKILL="$HOME/.codebuddy/skills/ai-usage-report/SKILL.md" ;;
  *)         LOCAL_SKILL="$HOME/.workbuddy/skills/ai-usage-report/SKILL.md" ;;
esac
LOCAL_VER=$(grep "^\*\*当前版本\*\*" "$LOCAL_SKILL" 2>/dev/null | grep -oE 'v[0-9]+\.[0-9]+(\.[0-9]+)?')
echo "本地版本：$LOCAL_VER（host: $SKILL_HOST）"
```

② 依次尝试两个 OTA 源，拿合法 Markdown 即停（**每个源 5 秒连接超时，总 10 秒超时**）：

```bash
fetch_skill_md() {
  local url="$1"
  local content
  content=$(curl -s --connect-timeout 5 --max-time 10 "$url" 2>/dev/null)
  # 合法性校验：必须以 YAML frontmatter 起始（`---` 开头），且能抽到 `**当前版本**` 行
  if [ -z "$content" ]; then
    return 1
  fi
  if ! printf '%s\n' "$content" | head -1 | grep -q '^---$'; then
    return 1
  fi
  if ! printf '%s\n' "$content" | grep -q '^\*\*当前版本\*\*'; then
    return 1
  fi
  printf '%s\n' "$content"
  return 0
}

REMOTE_CONTENT=""
REMOTE_SOURCE=""
for src in \
  "https://git.woa.com/ericqcsun/ai-usage-report/raw/main/SKILL.md|woa" \
  "https://raw.githubusercontent.com/zsutomato/ai-usage-report/main/SKILL.md|github"; do
  url="${src%|*}"
  name="${src##*|}"
  if candidate=$(fetch_skill_md "$url"); then
    REMOTE_CONTENT="$candidate"
    REMOTE_SOURCE="$name"
    break
  fi
done

if [ -n "$REMOTE_CONTENT" ]; then
  REMOTE_VER=$(echo "$REMOTE_CONTENT" | grep "^\*\*当前版本\*\*" | grep -oE 'v[0-9]+\.[0-9]+(\.[0-9]+)?')
  echo "远端版本：$REMOTE_VER（来源：$REMOTE_SOURCE）"
else
  echo "⚠️ 所有 OTA 源都不可用，跳过版本检查"
fi
```

> **合法性校验硬要求**：任何 OTA 源返回的内容，必须同时满足：
> - 第一行是 `---`（YAML frontmatter 起始）
> - 正文中存在 `**当前版本**` 行
>
> 任何一条不满足立即视为"该源不可用"，继续下一个源。这是防止把登录页 HTML 或其他错误响应误写回本地 SKILL.md 的最后一道防线。

③ 比对并处理：

- 若**版本一致**或**所有 OTA 源都不可达**：跳过，继续执行，在周报末尾附注 `⚠️ Skill 版本未校验（所有 OTA 源不可达）`（仅所有源都不可达时）
- 若**远端版本更新**：**双端同步覆盖**——把新版 SKILL.md 写入 `SKILL_PATHS` 里的**所有存在的副本路径**（可能同时包含 `~/.workbuddy/skills/ai-usage-report/SKILL.md` 和 `~/.codebuddy/skills/ai-usage-report/SKILL.md`）。告知用户 `Skill 已更新至 vX.Y.Z（来源：{REMOTE_SOURCE}，已同步到 N 个副本），建议重新触发以使用新版本`，然后**按当前版本继续执行**（因为当前 session 中加载的仍是旧版 prompt）。

  ```bash
  # $REMOTE_CONTENT 是拉到的新版 Markdown，$SKILL_PATHS 是 Step 0 发现的实际存在副本列表
  for p in "${SKILL_PATHS[@]}"; do
    mkdir -p "$(dirname "$p")"
    printf '%s\n' "$REMOTE_CONTENT" > "$p"
    echo "已同步 SKILL.md → $p"
  done
  ```

  **要点**：不主动创建另一端不存在的副本目录。如果用户只装了 WorkBuddy 端，CodeBuddy 端不存在，不要自动帮他装；等他手动 curl 一次再说。这条规则防止 Skill 越权。

### Step 3：识别用户身份、周报接收邮箱与额外收件人

**v2.5.0 起按 host 差异化读取身份**。读取优先级如下，**取到即停**：

**读取阶段（两端都扫描，按 host 优先）**：

1. **host 对应端的官方身份文件**：
   - 若 `SKILL_HOST = workbuddy`：依次读 `~/.workbuddy/USER.md` → `~/.workbuddy/IDENTITY.md` → `~/.workbuddy/SOUL.md`（WorkBuddy 社区约定的四件套结构）
   - 若 `SKILL_HOST = codebuddy`：优先读 `~/.codebuddy/CODEBUDDY.md`（CodeBuddy 官方用户级记忆文件，字段以自然语言格式存在于 `## CodeBuddy Added Memories` 等段落中），尝试从中提取企微 ID / 邮箱；读不到继续
2. **跨端兜底**（无论 host 是哪端，都再扫一遍另一端的文件，因为用户可能在两端都配置过）：
   - 若 host=workbuddy 且前面没拿到，再扫 `~/.codebuddy/CODEBUDDY.md`
   - 若 host=codebuddy 且前面没拿到，再扫 `~/.workbuddy/USER.md` → `~/.workbuddy/IDENTITY.md` → `~/.workbuddy/SOUL.md`
3. 每个文件都尝试提取以下字段（正则变体）：
   - 企业微信 ID：`企业微信 ID`、`企业微信ID`、`企微ID`、`企微 ID`、`wxid`、`WeChat Work ID`、`ericqcsun` 类 ID 格式
   - 周报接收邮箱：`周报接收邮箱`、`管理者邮箱`、`收件邮箱`、`report_to_email`、`report email`
   - 额外收件人列表（兼容历史字段）：`周报抄送邮箱`、`抄送邮箱`、`cc_email`、`cc emails`、`report_cc_email`；若值为 `无` / `none` / 空字符串，则视为"已配置但不额外发送给其他人"

**首次运行引导**（非定时任务模式才执行）：

- 若读取阶段都拿不到企业微信 ID，且当前是人工交互触发（非 automation），**停下来问用户一次**：
  > 检测到首次使用本 Skill，未找到你的企业微信 ID。请输入你的企业微信 ID（例如 `ericqcsun`），之后将写入本地持久化（具体路径见下），下次不再询问。
- 若读取阶段都拿不到周报接收邮箱，且当前是人工交互触发（非 automation），**再问用户一次**。文案必须强调"接收人"，避免用户误填成自己的邮箱：
  - 若当前对话主要使用中文，直接问（**严格使用下面这句原文，不要改写、不要删减**）：
    > ⚠️ 请输入**周报接收人**的邮箱（一般是你的上级管理者）。**请勿输入你自己的邮箱地址**——发件身份由 QQ Mail Connector 自动处理，你这里要填的是"周报要发给谁"。例如：`manager@example.com`
  - 若当前对话主要使用英文，直接问（**严格使用下面这句原文，不要改写、不要删减**）：
    > ⚠️ Please enter the **recipient's email address** for the weekly report (typically your manager). **Do NOT enter your own email address** — the sender identity is handled by the QQ Mail Connector. You should fill in "who will receive the report". For example: `manager@example.com`
- **仅在本次运行属于"首次补齐周报发送配置"时**（即刚刚通过交互拿到了 `REPORT_TO_EMAIL`），若还没有额外收件人配置，则顺带问一次是否还要发给其他人。同样要强调"收件人"语义：
  - 若当前对话主要使用中文，直接问：
    > 需要同时发给其他人吗？如果需要，请输入**其他收件人**的邮箱（同样是收件人，不是你自己的发件邮箱），多个用逗号分隔；如果不需要，回复"无"。
  - 若当前对话主要使用英文，直接问：
    > Do you want to send it to anyone else as additional recipients? If yes, enter the **recipients'** email addresses (not your own sender address), separated by commas; otherwise reply "none".
- **如果 `REPORT_TO_EMAIL` 已经存在，只是 `REPORT_CC_EMAILS` 缺失，则不要额外追问**，默认按"不额外发送给其他人"处理，避免每次发信都新增交互。

**写回阶段**（只写 host 对应的官方规范位置，不越权跨写另一端）：

- 若 `SKILL_HOST = workbuddy`：更新 `~/.workbuddy/USER.md`（文件不存在则创建）。按 WorkBuddy 社区约定的 Markdown 字段格式写入：
  ```
  # User Profile
  - 企业微信 ID：{用户输入或已识别值}
  - 周报接收邮箱：{用户输入或已识别值}
  - 周报抄送邮箱：{逗号分隔邮箱列表；若用户明确表示不需要额外发送给其他人，则写"无"}
  ```
- 若 `SKILL_HOST = codebuddy`：追加到 `~/.codebuddy/CODEBUDDY.md` 的 `## CodeBuddy Added Memories` 段落（CodeBuddy 官方约定的记忆段落；若不存在则创建）。格式保持自然语言可读：
  ```
  ## CodeBuddy Added Memories

  - 企业微信 ID：{用户输入或已识别值}
  - 周报接收邮箱：{用户输入或已识别值}
  - 周报抄送邮箱：{逗号分隔邮箱列表；若不需要则写"无"}
  ```
  若段落已存在相关字段，用增量更新方式替换旧值而不是重复追加。

然后把这些值分别赋给 `USER_WXID` / `REPORT_TO_EMAIL` / `REPORT_CC_EMAILS`。

**定时任务模式（无人值守）不执行此步**，直接走下一步兜底，避免卡住。

**最终兜底**：
- 企业微信 ID 仍然缺失时，执行 `whoami` 获取系统用户名（此时在周报末尾附注 `⚠️ 企业微信 ID 未配置，已用系统用户名兜底，请手动修正文件名`）
- 周报接收邮箱若仍缺失，则保留为空字符串；后续发送步骤根据该变量决定是提示配置还是继续发送
- 额外收件人若缺失、为空字符串，或明确写为 `无` / `none`，则统一视为空列表，不阻塞发送流程

将识别到的信息存为变量：
- `USER_NAME`：姓名（识别不到则为空）
- `USER_WXID`：企业微信 ID（识别不到则用 `whoami` 结果）
- `REPORT_TO_EMAIL`：周报接收邮箱（识别不到则为空）
- `REPORT_CC_EMAILS`：额外收件人列表（沿用历史 cc/抄送字段名兼容；识别不到或明确为"无"则为空列表）

### Step 4：三层数据采集

**核心原则**：三层数据源逐层采集，任何一层有数据就能出报告，不得因某层为空而中断。

#### 第一层：SQLite session 索引（先执行，用于发现工作空间路径）

目标：优先从 WorkBuddy 的 session 数据库提取**全量**会话列表，再用 Python 过滤统计周期内的记录；**不得假设只有 macOS，也不得假设系统一定有 `sqlite3` 命令。**

##### 1. 先确定数据库路径

会话库文件名固定为：

```text
codebuddy-sessions.vscdb
```

数据库路径应按 **`userDataPath + codebuddy-sessions.vscdb`** 的逻辑推导，而不是只写死一个绝对路径。

**v2.4.0 起，必须同时采集 WorkBuddy 和 CodeBuddy 两个产品的数据**。两个产品共享完全一致的数据库 schema（表名 `ItemTable`、字段 `conversationId / cwd / userId / title / status / createdAt / updatedAt`），采集逻辑可以完全复用。

构造**数据源列表 `DATA_SOURCES`**，按产品分别推导 userDataPath；每个数据源后续独立读库，结果最后按 `conversationId` 合并去重。

##### WorkBuddy 的 userDataPath

1. 若存在 `VSCODE_PORTABLE`，使用：`{VSCODE_PORTABLE}/user-data`
2. 若存在 `VSCODE_APPDATA`，使用：`{VSCODE_APPDATA}/WorkBuddy`
3. 若以上都没有，再按平台默认路径兜底：
   - macOS：`~/Library/Application Support/WorkBuddy`
   - Windows：`%APPDATA%\WorkBuddy`
   - Windows 若 `APPDATA` 不存在：`%USERPROFILE%\AppData\Roaming\WorkBuddy`
   - Linux：`$XDG_CONFIG_HOME/WorkBuddy`
   - Linux 若 `XDG_CONFIG_HOME` 不存在：`~/.config/WorkBuddy`

##### CodeBuddy 的 userDataPath（v2.4.0 新增）

CodeBuddy 存在**国内版（CN）和国际版**两种发行，目录名不同，且国内版目录带空格。按以下顺序逐个探测，**所有存在对应 `codebuddy-sessions.vscdb` 的路径都纳入 `DATA_SOURCES`**（多个版本并存时都采集）：

1. 若存在 `CODEBUDDY_APPDATA` 环境变量，使用：`{CODEBUDDY_APPDATA}`
2. 再按平台默认路径逐个探测：
   - macOS：
     - `~/Library/Application Support/CodeBuddy CN`（国内版，注意目录名带空格）
     - `~/Library/Application Support/CodeBuddy`（国际版 / 无后缀兜底）
   - Windows：
     - `%APPDATA%\CodeBuddy CN`
     - `%APPDATA%\CodeBuddy`
     - `%APPDATA%` 不存在时回退 `%USERPROFILE%\AppData\Roaming\` 同名目录
   - Linux：
     - `$XDG_CONFIG_HOME/CodeBuddy CN`
     - `$XDG_CONFIG_HOME/CodeBuddy`
     - `$XDG_CONFIG_HOME` 不存在时回退 `~/.config/` 同名目录

> 注意：`CodeBuddyExtension` 目录（有些版本会同时存在）**不是** CodeBuddy 独立客户端的 userDataPath，只是插件缓存，不包含 session DB，**不要**把它加入 `DATA_SOURCES`。

##### 汇总数据源列表

最终构造 `DATA_SOURCES` 数组，每项至少包含：

```json
{"product": "workbuddy", "user_data_path": "...", "db_path": "{user_data_path}/codebuddy-sessions.vscdb"}
{"product": "codebuddy", "user_data_path": "...", "db_path": "{user_data_path}/codebuddy-sessions.vscdb"}
```

**过滤原则**：
- 对每个候选路径，先检查 `{user_data_path}/codebuddy-sessions.vscdb` 是否真实存在
- 存在则纳入 `DATA_SOURCES`，不存在则静默跳过（不算错误、不附注）
- 若最终 `DATA_SOURCES` 为空（用户两个产品都没装，或数据目录都被清理过），第一层整体视为失败，降级到第二层

补充说明：少数高级场景可能通过 `--user-data-dir` 或其他方式覆盖默认目录；**只有在当前环境能明确识别到实际目录时才使用该值，否则不要猜测，继续按上述默认路径处理。**

##### 2. 对 DATA_SOURCES 每个源独立读库 + Python 过滤

对 `DATA_SOURCES` 列表中的每个数据源（`{product, user_data_path, db_path}`），独立执行读库流程；一个源失败不影响其他源。

先探测可用的 Python 命令：`python`、`python3`；Windows 额外尝试 `py -3`。

**第一层完整成功的必要条件**：必须至少有一个可用的 Python 命令，因为 `value` 字段里的 JSON 解析、活跃时间（`updatedAt` 优先、`createdAt` 兜底）过滤、`cwd` 提取与去重都要在 Python 侧完成；`sqlite3` CLI 只负责"读库"，**不等于**第一层已经成功。

对每个源按以下顺序尝试读取：

**方式 A：优先尝试 `sqlite3` CLI + Python 过滤**
- 先探测 `sqlite3` 命令是否存在
- 若 `sqlite3` 和 Python **都**可用，则先执行：

```bash
sqlite3 "{DB_PATH}" "SELECT value FROM ItemTable" 2>/dev/null
```

- 再把导出的结果交给 Python 做 JSON 解析与时间过滤

**方式 B：若 `sqlite3` 不可用，但 Python 可用，则改用 Python 内置 `sqlite3` 模块一站式读取与过滤**
- 使用同一个 Python 脚本完成数据库读取、JSON 解析、时间过滤与 `cwd` 提取：

```python
import sqlite3
conn = sqlite3.connect(DB_PATH)
cur = conn.cursor()
cur.execute("SELECT value FROM ItemTable")
rows = cur.fetchall()
```

**方式 C：若没有任何可用的 Python 命令**
- 即使存在 `sqlite3` CLI，也视为第一层整体失败（所有源都处理不了）
- 原因：无法可靠完成 JSON 解析、时间过滤和工作空间路径提取
- 不得中断整体流程，直接降级到第二层 memory 扫描

> 注意：`value` 是 JSON BLOB，无法在 SQL 层按时间过滤；**无论走方式 A 还是方式 B，最终都必须在 Python 侧基于时间戳筛选。**
>
> **时间字段选取（重要）**：
> - 优先使用 `updatedAt` / `lastUpdatedAt` / `updated_at` 等"最近活跃时间"字段做过滤窗口判断；
> - 只有在所有活跃时间字段都缺失时，才回退到 `createdAt`；
> - 原因：周报需要的是"本周真实发生的工作"，而不是"本周新建 session 的工作"。如果一个项目本周继续用上周/更早创建的 session 推进，其 `createdAt` 可能落在窗口外，但 `updatedAt` 仍在本周，**必须纳入**。

从每个源的筛选结果中提取（**每条记录额外打上 `product` 标签，标明来自 workbuddy 还是 codebuddy**）：
- 产品标签（`product`：`workbuddy` 或 `codebuddy`）
- 会话 ID（`conversationId`，用作跨源去重 key）
- 对话标题（`title`）
- 所属工作空间**完整路径**（`cwd`）
- 创建时间（`createdAt`）
- 最近活跃时间（`updatedAt` / `lastUpdatedAt`，若存在）
- 状态

##### 3. 跨源合并 + 去重 + 统计

所有源独立处理完后，把各自的筛选结果合并成一个总列表：

- **跨源去重 key**：`conversationId`。若同一个 `conversationId` 同时出现在两个产品的 DB 里（罕见，理论上可能出现在产品数据迁移场景），保留一条；保留哪个产品的标签可选"最近活跃的那个"。
- **跨源对话统计**：按 `product` 分组统计条数，供 Step 5 报告拆分显示（如 `WorkBuddy 132 + CodeBuddy 3 = 总 135`）。
- **跨源 cwd 合并**：将所有筛选结果的 `cwd` 去重合并，得到**session 层工作空间路径列表**（下文称 `SESSION_CWDS`），供第二层使用。

记录**实际贡献数据的产品列表 `ACTIVE_PRODUCTS`**：

- 若 WorkBuddy 源采到了至少 1 条记录，`ACTIVE_PRODUCTS` 包含 `workbuddy`
- 若 CodeBuddy 源采到了至少 1 条记录，`ACTIVE_PRODUCTS` 包含 `codebuddy`
- 两个都采到：`ACTIVE_PRODUCTS = ["workbuddy", "codebuddy"]`
- 只有一个产品有数据：`ACTIVE_PRODUCTS = ["workbuddy"]` 或 `["codebuddy"]`
- 第一层整体失败：`ACTIVE_PRODUCTS = []`

**此列表不代表"本周全部活跃工作空间"**。仍可能存在一种项目：本周有 memory 产出，但 session 的 `createdAt` 和 `updatedAt` 都在窗口外（例如 agent 基于更早 session 只做了少量交互但写了 memory 日志）；或系统时间字段格式不规范被 Python 解析失败；或数据库本身被清理过。第二层必须对这种情况做兜底扩展，不得只相信 `SESSION_CWDS`。

若出现以下任一情况，某个单源视为失败（**其他源继续处理**）：
- 该源的数据库路径不存在
- 该源的数据库读取报错
- 该源的 JSON 解析 / 时间过滤脚本执行失败

若出现以下情况，第一层**整体失败**但不得中断：
- Python 完全不可用
- 所有源都失败（包括"`DATA_SOURCES` 为空"这种情况）

**若第一层整体失败，继续第二层；并在最终报告中附注 `⚠️ session 索引层未采集成功，本次统计可能偏低`。**

#### 第二层：扫描 memory 日志（最详细）

先构造 memory 扫描用的**候选工作空间列表**，采用**并集语义**，不得只相信第一层 `SESSION_CWDS`：

1. **Session 层贡献**：若第一层成功，把 `SESSION_CWDS` 纳入候选集合（可能为空集，也可能是第一层拿到的若干 cwd）。
2. **Sibling 扩展扫描（常态补集，不是 fallback）**：只要当前工作空间路径可确定，**无论第一层是否成功、是否非空，都必须执行**一次受控 sibling 扩展扫描，把下面的目录加入候选集合：
   - 以当前工作空间的父目录作为候选容器
   - 只检查父目录下一层的直接子目录，**不递归扫描**
   - 对每个直接子目录 `{candidate}`，只有同时满足以下条件才纳入（**两个 memory 目录任一存在即可**）：
     - `{candidate}/.workbuddy/memory/` 或 `{candidate}/.codebuddy/memory/` 存在
     - 上面存在的目录里，统计周期内至少存在 1 个 `YYYY-MM-DD.md` 日期文件
3. **最终候选集合 = `SESSION_CWDS ∪ Sibling 扫描命中的目录`**，去重后作为第二层扫描范围。
4. 若最终候选集合仍为空（第一层失败且 sibling 一个候选都没命中），跳过第二层，直接进入第三层。
5. 若当前工作空间路径无法确定，则只能依赖 `SESSION_CWDS`；若此时 `SESSION_CWDS` 也为空，直接进入第三层。

**数据来源标记规则**（写入最终报告 `数据来源` 字段）：

- 仅使用了 `SESSION_CWDS`：`memory-session-cwds`
- 仅使用了 sibling 扫描命中的目录（session 层为空或失败）：`memory-expanded-siblings`
- 两者都有贡献（最常见）：`memory-session-cwds + expanded-siblings`
- sibling 扫描因命中集为空未贡献，但当前工作空间本身纳入：`memory-current-workspace`

> 注意：
> - sibling 扩展扫描是**常态补集**，不再只在第一层失败时触发。原因：即使第一层成功，`SESSION_CWDS` 也只反映"本周 session 活跃时间落在窗口内的工作空间"。若某项目本周推进了工作但 session 时间戳字段异常、或全部基于更早 session 完成零星交互、且 memory 日志在窗口内 —— 这种情况 `SESSION_CWDS` 会漏掉，必须靠 sibling 扫描把它兜回来。
> - sibling 扩展扫描只允许看父目录下一层；不得继续递归扫描子孙目录。
> - sibling 门槛是"本周有 memory 日志"（`.workbuddy/memory/` 或 `.codebuddy/memory/` 任一有日志即算），不会把本周没产出的无关项目误纳入。
> - 若 sibling 扫描额外贡献了候选目录（即 `memory-expanded-siblings` 或 `memory-session-cwds + expanded-siblings`），报告中应附注 `ℹ️ memory 层已按父目录下一层扩展扫描 sibling workspace，包含本周有 memory 日志但不在 session cwd 列表中的项目`。
> - 若最终只依赖 `SESSION_CWDS`（sibling 扫描无贡献），且 session 层本身失败或为空，报告中应附注 `⚠️ 未拿到全量工作空间列表，memory 层仅扫描当前工作空间或为空，本次统计可能偏低`。

对每个候选工作空间路径 `{cwd}`，**同时扫描两个 memory 目录**（v2.4.0 起）：

- `{cwd}/.workbuddy/memory/YYYY-MM-DD.md`（WorkBuddy 约定位置）
- `{cwd}/.codebuddy/memory/YYYY-MM-DD.md`（CodeBuddy 约定位置）

两个目录均存在时全部读取，采集到的内容合并。**文件绝对路径作为去重 key**，防止同一个文件在合并环节被重复读取（例如 `SESSION_CWDS` 和 sibling 扫描都命中同一个 cwd）。

对每个 memory 文件：
- 只读取统计周期内的 `YYYY-MM-DD.md` 日期文件
- 提取每天做了什么、产出了什么
- 可选：记录该条目来源于 `.workbuddy/memory/` 还是 `.codebuddy/memory/`，供调试，不影响最终报告文案

将采集到的内容按日期整理，记录来源工作空间。

**如果所有 memory 目录均为空或不存在，不报错，继续第三层。**

#### 第三层：conversation_search 搜索对话摘要（补充细节）

使用 `conversation_search` 工具，按以下策略搜索（中英文各若干轮，覆盖不同语言习惯）：

- 搜索 1：`query="本周完成的工作和任务"`，限定 `start_date` 和 `end_date` 为统计周期
- 搜索 2：`query="开发、修复、文档、部署、设计"`，同上日期范围
- 搜索 3：`query="会议、讨论、评审、分析"`，同上日期范围
- 搜索 4：`query="development, fix, deploy, review, document"`，同上日期范围
- 搜索 5：`query="meeting, discussion, analysis, design"`，同上日期范围

每次搜索取 `limit=10`，合并去重。

**日期参数约束（v2.5.1 硬约束）**：
- `start_date` / `end_date` 统一使用 Step 1 clamp 过的 `START_DATE` / `END_DATE`，格式严格为 `YYYY-MM-DD`，不得携带时间 / 时区 / `T` 分隔符
- 严禁向 `end_date` 传入晚于 `today` 的日期（Step 1 clamp 已兜底，但 agent 手动改写时也不得违反）
- 若怀疑日期参数是后端失败的根因，允许单次调试时**不传** `start_date` / `end_date`，让工具使用默认窗口（today-7 / today）

**并发要求（v2.5.0 起硬约束）**：以上 5 个搜索**必须在同一个 function_calls block 内并发发起**，不得串行等待每一个响应。原因：每个 `conversation_search` 查询是独立的、无依赖的网络 IO 调用，串行执行会累积成 15-25 秒的等待；并发执行总耗时接近单个查询的耗时（5-10 秒），吞吐提升 3-5 倍。

实现要点：
- Agent 在一次回复中一次性输出 5 个 `conversation_search` 工具调用（同一个 `<function_calls>` 块内）
- 等全部响应返回后再做合并去重
- 任意一个查询失败不影响其他查询的结果采用
- **禁止**为了"逐个看结果决定是否继续"而串行发起

若运行时限制一次最多 N 个并发工具调用，按 N 个一批分组发起；仍然要在同一批内并发，不得单个串行。

**如果 conversation_search 无结果或运行时不提供此工具，不报错，继续。**

**全失败降级（v2.5.1 新增硬约束）**：若 5 个并发查询**全部返回失败**（`success: false` 或抛异常），不得静默吞掉错误，必须：

1. 收集每个调用的 `message` / `queryHash`（如果有），作为结构化信息保留
2. 在第三层数据合并阶段**跳过**这 5 个空结果（不影响 sessions + memory 的输出）
3. 在 Step 5 生成周报时，frontmatter `数据来源` 字段明确写 `sessions + memory (tier3 all-failed)`
4. 并在周报末尾附一条**简洁的**降级附注（不要把 5 条原始错误日志原样塞进报告）：

   ```
   ⚠️ 第三层 conversation_search 本次 5 个查询全部失败（HTTP 错误或工具异常），已降级为 sessions + memory only。
      典型原因：start_date / end_date 参数不合规（如 end_date 超过 today）、后端服务暂不可用。
      首条失败信息：{results[0].message}
   ```

   只展示首条 `message` 即可，避免报告被 5 条近似错误日志淹没；完整 5 条错误保留在 agent 内部日志供归因，不输出到周报。

5. 降级附注的目标：让用户一眼看到"第三层失败 + 一句可操作的归因提示"，不误导成"skill 坏了"。

#### 数据合并

将三层数据合并：
- SQLite 提供**完整时间线、对话标题列表和工作空间路径**
- memory 日志提供**具体任务和产出**
- conversation_search 提供**任务摘要和上下文补充**

去重后按时间排序，作为周报内容的输入。

如果最终只有 memory 层有数据，数据来源写 `memory only`，并在报告中补充：

```text
⚠️ session 索引层未采集成功，本次任务数、活跃天数和对话数可能被低估
```

### Step 5：生成周报文件

**输出目录**：`~/Downloads/ai-usage-report/`（不存在则自动创建）

```bash
mkdir -p ~/Downloads/ai-usage-report
```

**文件名格式**：`{USER_WXID}-{统计起始日期}.md`
- 示例：`enzozhou-20260404.md`
- 如果 USER_WXID 为 whoami 结果，同样使用，如 `ericsun-20260404.md`

**文件内容格式**：

```
---
姓名：{USER_NAME，识别不到写"未识别"}
企业微信 ID：{USER_WXID}
周次：{上周六日期} Sat — {本周五日期} Fri
生成时间：{当前时间，格式 YYYY-MM-DD HH:mm}
Skill 版本：{当前本地 SKILL.md 中的版本号}
使用模型：{查看对话窗口顶部模型名称，列出本周用过的所有模型；识别不到则写"未识别"}
数据来源产品：{实际采到数据的产品列表，逗号分隔。`workbuddy` / `codebuddy` / `workbuddy + codebuddy`；若两个产品都没采到则写"无"}
数据来源：{实际采集到数据的层级，如 "sessions + memory(session-cwds + expanded-siblings) + conversation_search"、"sessions only"、"memory-session-cwds"、"memory-expanded-siblings" 或 "memory-current-workspace"}

【本周完成的任务】
（逐条列出，格式：序号. 任务描述 · 使用能力 · 所属项目/工作空间）
（根据三层数据合并后的结果生成，尽量具体）
（**不标注任务来自哪个产品**——用户视角关心"我做了啥"，不关心在 WorkBuddy 还是 CodeBuddy 里做的）

【使用的能力】
（从以下勾选：写代码 · 写文档 · 搜索资料 · 数据处理 · PPT/PDF · 浏览器自动化 · 定时任务 · 图片生成 · 知识库管理 · 代码审查 · 其他：___）

【本周对话统计】
- 总对话数：{合并后总数}{若两个产品都有数据，附加拆分括号：（WorkBuddy X + CodeBuddy Y）}
- 活跃工作空间：{合并去重后的工作空间列表}
- 活跃天数：{有对话记录的天数，跨产品合并去重}

【未完成 / 中途放弃的任务】
（没有则写"无"）

---
```

**对话统计字段的拆分显示规则**：

- 若 `ACTIVE_PRODUCTS = ["workbuddy", "codebuddy"]`：`总对话数：135（WorkBuddy 132 + CodeBuddy 3）`
- 若 `ACTIVE_PRODUCTS = ["workbuddy"]`：`总对话数：132`（不加括号）
- 若 `ACTIVE_PRODUCTS = ["codebuddy"]`：`总对话数：3`（不加括号）
- 若 `ACTIVE_PRODUCTS = []`：`总对话数：0`（附注"⚠️ session 索引层未采集成功"）

#### 任务粒度口径（生成【本周完成的任务】时遵循）

为避免不同成员的任务数对比失真，按以下口径决定"1 条任务"的边界：

- **核心原则：产出驱动，不是 session 驱动**。**session 数量不是任务数的代理指标**。同一项目的多个 session（包括跨天、跨产品、同日多次触发、用老 session 继续聊）都应按"交付物"合并——5 个 session 做 1 件事就是 1 条任务；1 个 session 做 5 件独立交付物就是 5 条任务。Agent 生成任务清单时**不得按 session 数拆分**，也不得拿对话数 / session 数当任务数的近似值。
- **1 条任务 = 1 个独立可交付产出**。交付物是一份文档、一个功能模块、一次对外回复、一场会议纪要、一份分析结论等。
- **批量同质工作合并为 1 条**，在描述里注明规模。
  - ✅ 正确：`回复客户工单 · 搜索资料 · TAM 工作台（约 600 条）`
  - ❌ 错误：把 600 条工单拆成 600 行
- **同一产出的多语言/多版本/多轮迭代合并为 1 条**，在描述里注明变体数。
  - ✅ 正确：`TCCA 认证考纲中英版撰写 · 写文档 · 培训认证`
  - ❌ 错误：把"中文版大纲""英文版大纲""大纲 review"拆成 3 条
- **同一项目的连续多日推进合并为 1 条**，在描述里注明跨天。
  - ✅ 正确：`某客户 POC 部署方案设计（周二—周四推进）· 写文档 · 售前`
- **同一项目的多 session 合并为 1 条**（v2.4.1 起明确）。即便你在同一个 workspace 里开了 5 个 session 推进同一件事，任务清单里也只出现 1 条；session 数只影响【本周对话统计】段，不影响任务条数。
- **筛选门槛**：闲聊、环境配置、单次工具调用、一次性脚本等**不计入**任务清单（可归到"使用的能力"里体现）。
- **目标粒度**：大多数成员一周 5–15 条是健康区间。远低于 5 条可能漏报，远高于 20 条大概率粒度过细，需要合并。

### Step 6：输出结果并尝试邮件发送

文件写入完成后：

#### Step 6A：始终执行的结果输出

1. 告知用户文件的**完整绝对路径**
2. 用 `deliver_attachments` 或 `open_result_view` 展示文件
3. 明确说明本次是否进入邮件发送流程

#### Step 6B：检测 `qq-mail` 发送能力（以 `GetMe` 成功为准）

仅当以下条件**同时满足**时，才继续发送：
- 当前是人工交互模式（非 automation）
- `REPORT_TO_EMAIL` 不为空
- `qq-mail` MCP server 可调用
- 调用 `GetMe` 成功，且拿到至少一个可用 alias_id

具体规则：

1. 先调用 `qq-mail` 的 `GetMe`
   - 若调用失败、server 不存在、未授权、没有 alias，视为 `qq-mail` 当前不可用
   - 降级处理：告诉用户“已生成周报文件，但当前环境未配置可用的 QQ Mail Connector，请先完成配置后重新执行”；**不要中断整个 Skill**

2. 若 `REPORT_TO_EMAIL` 为空：
   - 人工交互模式下：提示用户先在 `~/.workbuddy/USER.md` 中补 `周报接收邮箱`，或重新执行一次让 Skill 记录该字段
   - automation 模式下：直接跳过发送，并在输出中附注 `⚠️ 周报接收邮箱未配置，已跳过邮件发送`

3. 若 `GetMe` 成功：
   - 默认选用 `is_primary: true` 的 alias；若有多个 primary，优先取第一个
   - **邮件主题必须严格等于以下字面模板**（方括号、空格、连字符、日期格式**一个字符都不得变**）：

     ```
     [AI Weekly Report] {统计起始日期 YYYY-MM-DD} - {USER_WXID}
     ```

     ✅ **正确格式（这是唯一允许的格式）**：
     - `[AI Weekly Report] 2026-04-18 - ericqcsun`
     - `[AI Weekly Report] 2026-04-20 - kaihongfu`

     ❌ **禁止的变体**（agent 若产出以下任一，即为违规，必须重算后再发送）：
     - `AI 使用周报 | kaihongfu | 2026-04-20 ~ 2026-04-24`（禁止中文前缀 / 管道符 / 日期范围）
     - `AI Usage Report 2026-04-18 to 2026-04-24`（禁止改写前缀 / 用 "to" / 缺失用户名）
     - `[AI 周报] 2026-04-20 - kaihongfu`（禁止方括号内用中文）
     - `[AI Weekly Report] 2026-04-20~2026-04-24 - kaihongfu`（禁止日期范围，**只能用起始日期这一个日期**）
     - `[AI Weekly Report] - kaihongfu`（禁止缺失日期）
     - `[AI Weekly Report] 2026-04-20`（禁止缺失用户名）

     **即使**用户在当前对话里主要使用中文、或 agent 觉得"中文标题对用户更友好"，邮件主题**仍必须使用上述英文 ASCII 字面模板**，不得以任何理由扩展或替换。**不得把报告正文 Markdown 的 h1 标题（"AI 使用周报"）当作邮件主题来源**。

     **主题自检（必须执行）**：生成主题字符串后、构造 Phase 1 请求之前，必须用下面的正则验证一次：

     ```
     ^\[AI Weekly Report\] \d{4}-\d{2}-\d{2} - [a-zA-Z0-9_.-]+$
     ```

     若不匹配，说明主题被改写了，**必须重算**，不得进入 Phase 1。

   - 邮件正文直接使用刚生成的 Markdown 文件全文，`body_format` 使用 `PLAIN`
   - 若 `REPORT_CC_EMAILS` 为非空列表，则在发送参数中追加 `cc`；若为空，则**不要传 `cc` 参数**
   - **能力边界说明**：`qq-mail` 当前即便传入 `cc`，实际投递效果也可能把这些地址并入 To；因此 Skill 仅把 `REPORT_CC_EMAILS` 作为"额外收件人"输入，不对标准邮件头里的 CC 语义做承诺

#### Step 6C：发起 Phase 1 发送预览

以选中的 alias_id、`REPORT_TO_EMAIL`、主题、正文调用一次 `SendMessage`，**不要带 `confirmation_token`**；若 `REPORT_CC_EMAILS` 非空，则同一次请求里附带 `cc` 参数。

预期结果：
- 服务端返回 `428` / `42801`
- 返回 `operation_summary`
- 返回 `confirmation_token`

拿到后，向用户展示简短确认信息（**主题必须用反引号包起来，让用户能一眼看清是否符合标准格式**）：

- 若 `REPORT_CC_EMAILS` 为空：
  > 即将发送，请确认以下信息：
  > - 主题：`{邮件主题}`
  > - 发件：`{发件邮箱}`
  > - 收件：`{REPORT_TO_EMAIL}`
  >
  > 如确认发送，请回复"确认发送"。
- 若 `REPORT_CC_EMAILS` 非空：
  > 即将发送，请确认以下信息：
  > - 主题：`{邮件主题}`
  > - 发件：`{发件邮箱}`
  > - 收件：`{REPORT_TO_EMAIL}`
  > - 额外收件人：`{REPORT_CC_EMAILS}`
  >
  > 如确认发送，请回复"确认发送"。

#### Step 6D：等待用户明确确认后完成 Phase 2

只有当用户在对话中明确回复以下任一表达时，才允许继续：
- `确认发送`
- `确认`
- `发送`

然后**立即**用与 Phase 1 **完全相同**的参数重试一次 `SendMessage`，只额外加上 `confirmation_token`。

成功标准：
- 服务端返回 `queued: true` 或其他成功结果
- 告知用户“邮件已进入发送队列”

#### Step 6E：automation 模式下的固定行为

若当前是 automation 模式：
- **始终跳过 `qq-mail` 发送步骤**
- 原因要写清楚：`qq-mail` 的 `SendMessage` 需要用户显式确认，当前模式下无法完成二阶段确认
- 输出提示：
  > 周报文件已生成；由于 `qq-mail` 发送需要人工确认，本次 automation 不执行邮件发送。

---

## 注意事项

- 只描述实际发生的事情，不加建议或评价
- 所有字段从数据源自动提取，确实没有的写"未识别"或"无"
- 时间范围必须以 `date` 命令结果为准，不得凭印象填写
- **任何一个步骤失败都不得中断整体流程**，降级处理后继续
- `qq-mail` 是否可用，以 `GetMe` 调用成功并拿到 alias 为准；不要用“本地 skill 文件存在”代替真实可用性判断
- `qq-mail` 发送必须遵守两阶段确认：先拿 `confirmation_token`，再等待用户显式批准后重发；Skill 不得替用户自动确认
- `qq-mail` 当前即使接收 `cc` 参数，实际投递效果也可能把这些地址并入 To；因此 `REPORT_CC_EMAILS` 只应按“额外收件人输入”理解，不要对标准 CC 语义做承诺
- 如果三层数据源全部为空，生成一份只包含元信息的空白周报，并在报告中注明"⚠️ 未检测到本周使用记录"

---

## 推荐：设置为定时任务

建议设置为**每周五 18:00 自动执行**，一次性配置后每周自动生成：

1. 打开 WorkBuddy，点击左侧「自动化」
2. 点击右上角「新建自动化」
3. 名称：`AI 使用周报`
4. Prompt：`执行 AI 使用周报`
5. 执行时间：每周五 18:00
6. 保存

生成后会优先输出 `.md` 文件；若当前环境已配置可用的 `qq-mail` Connector，则手动执行时可继续进入确认发送流程。自动化模式下仍只生成文件，不执行邮件发送。

---

## 更新此 Skill

此 Skill 同时托管在**工蜂（内网优先）**和 **GitHub（公开兜底）**两个 OTA 源，内容自动同步：

- 工蜂：`https://git.woa.com/ericqcsun/ai-usage-report`
- GitHub：`https://github.com/zsutomato/ai-usage-report`

Skill 内置 OTA 会按"工蜂 → GitHub"顺序自动尝试，任一源拿到合法内容即停。

**手动强制更新**（v2.5.0 起支持双端安装，把 SKILL.md 写到你实际在用的那端；两端都用就两个命令都跑一遍）：

```bash
# —— 源 × 目标端 矩阵（总共 4 种组合，按你实际情况选）——

# 1. GitHub 源 → WorkBuddy 端（最常见）
mkdir -p ~/.workbuddy/skills/ai-usage-report && \
curl -sL "https://raw.githubusercontent.com/zsutomato/ai-usage-report/main/SKILL.md" \
  -o ~/.workbuddy/skills/ai-usage-report/SKILL.md

# 2. GitHub 源 → CodeBuddy 端
mkdir -p ~/.codebuddy/skills/ai-usage-report && \
curl -sL "https://raw.githubusercontent.com/zsutomato/ai-usage-report/main/SKILL.md" \
  -o ~/.codebuddy/skills/ai-usage-report/SKILL.md

# 3. 工蜂源 → WorkBuddy 端（内网环境可尝试；匿名 raw 当前受限，失败换 GitHub）
mkdir -p ~/.workbuddy/skills/ai-usage-report && \
curl -sL "https://git.woa.com/ericqcsun/ai-usage-report/raw/main/SKILL.md" \
  -o ~/.workbuddy/skills/ai-usage-report/SKILL.md

# 4. 工蜂源 → CodeBuddy 端
mkdir -p ~/.codebuddy/skills/ai-usage-report && \
curl -sL "https://git.woa.com/ericqcsun/ai-usage-report/raw/main/SKILL.md" \
  -o ~/.codebuddy/skills/ai-usage-report/SKILL.md
```

或直接对 Agent 说：`请手动更新 AI 使用周报 Skill 到最新版本`

---

## Changelog

- **v2.5.1** (2026-04-30)：修复 Step 4 第三层 `conversation_search` 在"周五非工作时间触发周报"场景下 5 个查询全部 HTTP 400 的 bug。① **方案 A（根治）**：Step 1 末尾对 `END_DATE` 做 clamp —— `if END_DATE > today then END_DATE = today`。根因：v2.5.0 算出的 `end_date=2026-05-01`（本周五 23:59）在今天是周四时属于未来日期，`conversation_search` 后端会硬校验 `end_date <= today` 返回 400。QC 做的 V2 vs V3 隔离实验（同一字不差的 query，只改日期参数就从 success 变 400）锁定此为唯一根因。② **方案 C（防御+归因）**：第三层 5 个并发查询全失败时不得静默吞掉，必须在 `数据来源` 字段标记 `sessions + memory (tier3 all-failed)`，并在周报末尾附一条简洁降级附注（含首条失败 `message`），方便下次撞类似 bug 时直接归因。③ 明示日期参数格式硬约束：严格 `YYYY-MM-DD`，不得带 T 分隔符 / 时区 / 时间。
- **v2.5.0** (2026-04-28)：**Skill 正式双端适配**——可同时安装/运行在 WorkBuddy 和 CodeBuddy 下，互不干扰。① 新增 Step 0 "运行时 host 探测"，输出 `SKILL_HOST` 变量；② Step 2 OTA 从单端改为**双端同步**：更新当前 host 对应副本 + 同步已存在的另一端副本，不主动越权创建另一端；③ Step 3 身份识别分 host 差异化：WorkBuddy host 读 `~/.workbuddy/{USER,IDENTITY,SOUL}.md` 四件套，CodeBuddy host 优先读 `~/.codebuddy/CODEBUDDY.md`（官方）再跨读 `.workbuddy/` 四件套作为兜底；写回只写 host 对应的官方规范文件；④ Step 4 第三层 `conversation_search` 5 个查询改为**并发发起**（同一 function_calls block 内），总耗时从 ~20s 降到 ~5s，整条流水线提速约 3-4 倍；⑤ 文件末尾"手动强制更新"命令扩展为"源 × 目标端"四种组合矩阵
- **v2.4.2** (2026-04-28)：加固 Step 6 邮件主题约束，防止 agent 生成"AI 使用周报 \| xxx \| 日期范围"、"AI Usage Report xxx to xxx" 等非标准格式。新增 6 条反例禁止清单、硬规定主题字面模板不得改写、禁止把报告 h1 标题当主题来源；Phase 1 发送前必须通过正则自检 `^\[AI Weekly Report\] \d{4}-\d{2}-\d{2} - [a-zA-Z0-9_.-]+$` 才能进入确认流程；Phase 1 预览文案改为多行列表 + 主题反引号显示，便于用户在确认前一眼识别异常主题
- **v2.4.1** (2026-04-27)：加固 Step 5 任务粒度口径——显式声明"**session 数量不是任务数的代理指标**，同一项目多个 session 按交付物合并"，防止未来 agent 把 session 数当任务数拆分。文档加固，行为不变。
- **v2.4.0** (2026-04-27)：新增 **CodeBuddy 数据源**，周报同时覆盖 WorkBuddy 和 CodeBuddy 两个产品的使用记录。① Step 4 第一层把单源 `userDataPath` 推导改为多产品 `DATA_SOURCES` 列表（WorkBuddy + CodeBuddy CN + CodeBuddy 国际版），两个产品共享相同 DB schema 所以采集逻辑完全复用；② 每条 session 记录打 `product` 标签，跨源按 `conversationId` 去重合并，某个单源失败不影响其他源；③ Step 4 第二层 memory 扫描同时覆盖 `.workbuddy/memory/` 和 `.codebuddy/memory/` 两个目录，按文件绝对路径去重；④ Step 5 报告元信息新增 `数据来源产品` 字段；对话统计支持拆分显示（`总数：135（WorkBuddy 132 + CodeBuddy 3）`）；⑤ 任务清单**不标注产品来源**，保持用户视角"本周我做了啥"干净输出
- **v2.3.1** (2026-04-27)：Step 2 OTA 源策略升级为双源并行：按"工蜂（内网优先）→ GitHub（公开兜底）"顺序尝试，任一源拿到合法内容即停；新增响应合法性校验（必须以 `---` frontmatter 起始 + 包含 `**当前版本**` 行），防止登录页 HTML 等错误响应被误写回本地；后续工蜂匿名 raw 就绪时 Skill 无需修改即自动切换；"更新此 Skill"段同步双源手动升级命令
- **v2.3.0** (2026-04-27)：修复"本周用老 session 继续聊的项目被整体漏掉"的采集 bug：① Step 4 第一层时间过滤改为优先使用 `updatedAt` / `lastUpdatedAt` 等活跃时间字段，只在所有活跃时间字段缺失时才回退 `createdAt`；② Step 4 第二层候选工作空间列表改为**并集语义**——`SESSION_CWDS ∪ sibling 扫描命中的目录`，sibling 扩展扫描从"失败 fallback"升级为"常态补集"，只要当前工作空间路径可确定就一定执行；③ 新增 `memory-session-cwds` / `memory-session-cwds + expanded-siblings` 数据来源标记；④ sibling 门槛不变（仅父目录下一层 + 本周有 memory 日志）
- **v2.2.9** (2026-04-27)：强化 Step 3 周报接收邮箱的首次交互文案，显式强调“接收人/一般是你的上级管理者”、“请勿输入你自己的邮箱”，避免用户误把自己当成收件人；额外收件人追问同步强调“收件人，不是你自己的发件邮箱”
- **v2.2.8** (2026-04-26)：根据手测结果收敛 `qq-mail` 的多人收件口径：保留历史 `cc` / “周报抄送邮箱”字段兼容，但用户可见文案统一改为“额外收件人”；并明确说明当前实际投递效果可能把这些地址并入 To，不承诺标准 CC 语义
- **v2.2.7** (2026-04-26)：在 `qq-mail` 发送链路中加入固定 To + 可选 CC 配置：Step 3 支持识别/持久化“周报抄送邮箱”，首次补齐周报发送配置时可顺带询问一次 CC；Step 6 在发送参数与确认文案中支持展示 To + CC 摘要
- **v2.2.6** (2026-04-26)：重写 Step 4 第二层 fallback 语义：不再默认只扫当前 workspace，而是优先尝试受控 sibling 扩展扫描（仅父目录下一层、仅纳入存在 `.workbuddy/memory/` 且统计周期内有日志的目录）；同时把数据来源区分为 `memory-current-workspace` 与 `memory-expanded-siblings`
- **v2.2.5** (2026-04-26)：继续收口 Step 4：明确第一层完整成功必须有 Python 参与 JSON 解析与 `createdAt` 过滤，`sqlite3` CLI 仅是读库手段；同时补齐第二层 fallback 语义，若拿不到全量 `cwd` 列表则只扫描当前工作空间，并要求显式提示统计可能偏低
- **v2.2.4** (2026-04-26)：收窄 Step 4 的路径推导主叙事：默认按 `VSCODE_PORTABLE` / `VSCODE_APPDATA` / 平台默认目录处理，`--user-data-dir` 降为补充说明，仅在当前环境能明确识别时才使用，避免把少数高级覆盖场景写成常规路径
- **v2.2.3** (2026-04-26)：重写 Step 4 第一层 session 索引逻辑：按平台与覆盖项推导 WorkBuddy userDataPath，并改为能力探测式读取（`sqlite3` CLI → Python `sqlite3` 模块 → `memory only`）；若降级为 `memory only`，要求显式提示统计结果可能被低估
- **v2.2.2** (2026-04-26)：统一邮件主题为纯英文 ASCII，格式固定为 `[AI Weekly Report] {统计起始日期 YYYY-MM-DD} - {USER_WXID}`，避免姓名或各国语言字符进入 Subject
- **v2.2.1** (2026-04-26)：补齐收件邮箱首次交互文案，要求按当前对话语言显式直接提问；中文使用“接收周报的邮箱地址是？”，英文使用对应英文问句
- **v2.2** (2026-04-26)：新增 `qq-mail` 发送集成路径：Step 3 增加“周报接收邮箱”识别与首次交互持久化；Step 6 改为“输出结果 + `qq-mail` 可用性探测 + 两阶段确认发送”；明确 `qq-mail` 以 `GetMe` 成功为准，automation 模式只生成文件不发信
- **v2.1** (2026-04-20)：Step 3 增加首次运行交互引导（未找到企微 ID 时询问并写入 `~/.workbuddy/USER.md` 持久化，automation 模式跳过）；Step 5 增加任务粒度口径说明（1 条任务 = 1 个独立可交付产出，批量同质工作合并并注明规模，筛选掉闲聊和一次性调用）
- **v2.0** (2026-04-10)：重构数据采集（三层降级：SQLite → memory → conversation_search，SQLite 先行以发现工作空间路径）；身份识别四级回退+正则匹配多种字段名；固定输出目录 `~/Downloads/ai-usage-report/`；OTA 加超时容错+更新后不重新执行；新增对话统计、数据来源、使用模型字段；中英文双语搜索词
- **v1.5** (2026-04-04)：迁移至 GitHub 公开仓库，加入 OTA 自动更新
- **v1.3** (2026-04-04)：加入时间校准、版本号、生成时间戳
- **v1.0** (2026-03-27)：初始版本

**当前版本**：v2.5.1
**最后更新**：2026-04-30
**维护人**：QC
