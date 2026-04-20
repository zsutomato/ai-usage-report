---
name: ai-usage-report
slug: ai-usage-report
version: 2.1.0
description: "自动回溯本周 Agent 对话记录，生成标准格式使用周报。三层数据采集（SQLite → memory → conversation_search），支持定时任务无人值守执行。"
---

# AI 使用周报 Skill

## 功能说明

自动回溯本周所有 Agent 对话记录，生成标准格式的使用周报 Markdown 文件，用于统计 AI 工具使用情况。

**适用场景**：每周五下班前，或设置为定时任务自动执行。

---

## 使用方法

直接对 Agent 说：

> 执行 AI 使用周报

Agent 会自动执行以下步骤，无需手动干预。

---

## 执行流程

收到"执行 AI 使用周报"指令后，严格按以下步骤顺序执行：

### Step 1：校准当前时间

执行命令获取准确日期，**不得凭印象猜测**：

```bash
date "+%Y-%m-%d %A %H:%M"
```

根据输出结果计算本次统计周期：**上周六 00:00 — 本周五 23:59**。

将起止日期存为变量供后续步骤使用，格式为 `YYYY-MM-DD`。

### Step 2：检查并更新 Skill 版本

① 读取本地版本号：

```bash
LOCAL_VER=$(grep "^\*\*当前版本\*\*" ~/.workbuddy/skills/ai-usage-report/SKILL.md | grep -oE 'v[0-9]+\.[0-9]+')
echo "本地版本：$LOCAL_VER"
```

② 从 GitHub 读取最新版本号（**5 秒超时，失败跳过**）：

```bash
REMOTE_CONTENT=$(curl -s --connect-timeout 5 --max-time 10 \
  "https://raw.githubusercontent.com/zsutomato/ai-usage-report/main/SKILL.md" 2>/dev/null)
if [ -n "$REMOTE_CONTENT" ]; then
  REMOTE_VER=$(echo "$REMOTE_CONTENT" | grep "^\*\*当前版本\*\*" | grep -oE 'v[0-9]+\.[0-9]+')
  echo "远端版本：$REMOTE_VER"
else
  echo "⚠️ 无法连接 GitHub，跳过版本检查"
fi
```

③ 比对并处理：

- 若**版本一致**或**网络不可达**：跳过，继续执行，在周报末尾附注 `⚠️ Skill 版本未校验（网络不可达）`（仅网络不可达时）
- 若**远端版本更新**：自动覆盖本地 Skill，告知用户"Skill 已更新至 vX.X，建议重新触发以使用新版本"，然后**按当前版本继续执行**（因为当前 session 中加载的仍是旧版 prompt）

### Step 3：识别用户身份

按以下顺序逐级查找，**取到即停**：

1. 读取 `~/.workbuddy/USER.md`，用正则提取企业微信 ID（匹配以下变体：`企业微信 ID`、`企业微信ID`、`企微ID`、`企微 ID`、`wxid`、`WeChat Work ID`、`ericqcsun` 类 ID 格式）
2. 读取 `~/.workbuddy/IDENTITY.md`，同上正则匹配
3. 读取 `~/.workbuddy/SOUL.md`，提取姓名或身份信息
4. **首次运行引导**（非定时任务模式才执行）：若前 3 步都拿不到企业微信 ID，且当前是人工交互触发（非 automation），**停下来问用户一次**：
   > 检测到首次使用本 Skill，未找到你的企业微信 ID。请输入你的企业微信 ID（例如 `ericqcsun`），之后将写入 `~/.workbuddy/USER.md` 持久化，下次不再询问。

   拿到用户回复后，追加写入 `~/.workbuddy/USER.md`（文件不存在则创建），格式：
   ```
   # User Profile
   - 企业微信 ID：{用户输入}
   ```
   然后把该值赋给 `USER_WXID`。

   **定时任务模式（无人值守）不执行此步**，直接走下一步兜底，避免卡住。

5. 最终兜底：执行 `whoami` 获取系统用户名（此时在周报末尾附注 `⚠️ 企业微信 ID 未配置，已用系统用户名兜底，请手动修正文件名`）

将识别到的信息存为变量：
- `USER_NAME`：姓名（识别不到则为空）
- `USER_WXID`：企业微信 ID（识别不到则用 `whoami` 结果）

### Step 4：三层数据采集

**核心原则**：三层数据源逐层采集，任何一层有数据就能出报告，不得因某层为空而中断。

#### 第一层：SQLite session 索引（先执行，用于发现工作空间路径）

读取 WorkBuddy 的 session 数据库，获取**全量** session 列表，再用 Python 过滤统计周期内的记录：

```bash
sqlite3 ~/Library/Application\ Support/WorkBuddy/codebuddy-sessions.vscdb \
  "SELECT value FROM ItemTable" 2>/dev/null
```

> 注意：`value` 是 JSON BLOB，无法在 SQL 层按时间过滤，需全量拉取后在 Python 侧基于 `createdAt` 时间戳筛选。

从筛选结果中提取：
- 对话标题（`title`）
- 所属工作空间**完整路径**（`cwd`）
- 创建时间
- 状态

同时，将所有 `cwd` 路径去重，得到**工作空间路径列表**，供第二层使用。

**如果 sqlite3 不可用或数据库不存在，工作空间列表为空，后续第二层跳过，不报错。**

#### 第二层：扫描 memory 日志（最详细）

基于第一层获取的**工作空间路径列表**，遍历每个工作空间的 `.workbuddy/memory/` 目录，读取统计周期内的日期文件：

对每个工作空间路径 `{cwd}`：
- 检查 `{cwd}/.workbuddy/memory/` 是否存在
- 读取统计周期内的 `YYYY-MM-DD.md` 日期文件
- 提取每天做了什么、产出了什么

将采集到的内容按日期整理，记录来源工作空间。

**如果所有 memory 目录均为空或不存在，不报错，继续第三层。**

#### 第三层：conversation_search 搜索对话摘要（补充细节）

使用 `conversation_search` 工具，按以下策略搜索（中英文各一轮，覆盖不同语言习惯）：

- 搜索 1：`query="本周完成的工作和任务"`，限定 `start_date` 和 `end_date` 为统计周期
- 搜索 2：`query="开发、修复、文档、部署、设计"`，同上日期范围
- 搜索 3：`query="会议、讨论、评审、分析"`，同上日期范围
- 搜索 4：`query="development, fix, deploy, review, document"`，同上日期范围
- 搜索 5：`query="meeting, discussion, analysis, design"`，同上日期范围

每次搜索取 `limit=10`，合并去重。

**如果 conversation_search 无结果，不报错，继续。**

#### 数据合并

将三层数据合并：
- SQLite 提供**完整时间线、对话标题列表和工作空间路径**
- memory 日志提供**具体任务和产出**
- conversation_search 提供**任务摘要和上下文补充**

去重后按时间排序，作为周报内容的输入。

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
数据来源：{实际采集到数据的层级，如 "sessions + memory + conversation_search" 或 "sessions only"}

【本周完成的任务】
（逐条列出，格式：序号. 任务描述 · 使用能力 · 所属项目/工作空间）
（根据三层数据合并后的结果生成，尽量具体）

【使用的能力】
（从以下勾选：写代码 · 写文档 · 搜索资料 · 数据处理 · PPT/PDF · 浏览器自动化 · 定时任务 · 图片生成 · 知识库管理 · 代码审查 · 其他：___）

【本周对话统计】
- 总对话数：{SQLite 统计结果}
- 活跃工作空间：{去重后的工作空间列表}
- 活跃天数：{有对话记录的天数}

【未完成 / 中途放弃的任务】
（没有则写"无"）

---
```

#### 任务粒度口径（生成【本周完成的任务】时遵循）

为避免不同成员的任务数对比失真，按以下口径决定"1 条任务"的边界：

- **1 条任务 = 1 个独立可交付产出**。交付物是一份文档、一个功能模块、一次对外回复、一场会议纪要、一份分析结论等。
- **批量同质工作合并为 1 条**，在描述里注明规模。
  - ✅ 正确：`回复客户工单 · 搜索资料 · TAM 工作台（约 600 条）`
  - ❌ 错误：把 600 条工单拆成 600 行
- **同一产出的多语言/多版本/多轮迭代合并为 1 条**，在描述里注明变体数。
  - ✅ 正确：`TCCA 认证考纲中英版撰写 · 写文档 · 培训认证`
  - ❌ 错误：把"中文版大纲""英文版大纲""大纲 review"拆成 3 条
- **同一项目的连续多日推进合并为 1 条**，在描述里注明跨天。
  - ✅ 正确：`某客户 POC 部署方案设计（周二—周四推进）· 写文档 · 售前`
- **筛选门槛**：闲聊、环境配置、单次工具调用、一次性脚本等**不计入**任务清单（可归到"使用的能力"里体现）。
- **目标粒度**：大多数成员一周 5–15 条是健康区间。远低于 5 条可能漏报，远高于 20 条大概率粒度过细，需要合并。

### Step 6：输出结果

文件写入完成后：

1. 告知用户文件的**完整绝对路径**
2. 用 `deliver_attachments` 或 `open_result_view` 展示文件
3. 提示用户将文件发给管理者

---

## 注意事项

- 只描述实际发生的事情，不加建议或评价
- 所有字段从数据源自动提取，确实没有的写"未识别"或"无"
- 时间范围必须以 `date` 命令结果为准，不得凭印象填写
- **任何一个步骤失败都不得中断整体流程**，降级处理后继续
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

生成后将 `.md` 文件发给管理者即可。

---

## 更新此 Skill

此 Skill 托管于 GitHub 公开仓库：

```
https://github.com/zsutomato/ai-usage-report
```

**手动强制更新**：

```bash
curl -s "https://raw.githubusercontent.com/zsutomato/ai-usage-report/main/SKILL.md" \
  > ~/.workbuddy/skills/ai-usage-report/SKILL.md
```

或直接对 Agent 说：`请手动更新 AI 使用周报 Skill 到最新版本`

---

## Changelog

- **v2.1** (2026-04-20)：Step 3 增加首次运行交互引导（未找到企微 ID 时询问并写入 `~/.workbuddy/USER.md` 持久化，automation 模式跳过）；Step 5 增加任务粒度口径说明（1 条任务 = 1 个独立可交付产出，批量同质工作合并并注明规模，筛选掉闲聊和一次性调用）
- **v2.0** (2026-04-10)：重构数据采集（三层降级：SQLite → memory → conversation_search，SQLite 先行以发现工作空间路径）；身份识别四级回退+正则匹配多种字段名；固定输出目录 `~/Downloads/ai-usage-report/`；OTA 加超时容错+更新后不重新执行；新增对话统计、数据来源、使用模型字段；中英文双语搜索词
- **v1.5** (2026-04-04)：迁移至 GitHub 公开仓库，加入 OTA 自动更新
- **v1.3** (2026-04-04)：加入时间校准、版本号、生成时间戳
- **v1.0** (2026-03-27)：初始版本

**当前版本**：v2.1  
**最后更新**：2026-04-20  
**维护人**：QC
