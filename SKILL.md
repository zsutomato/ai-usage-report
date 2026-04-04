# AI 使用周报 Skill

## 功能说明

自动回溯本周所有 Agent 对话记录，生成标准格式的使用周报 Markdown 文件，用于提交给团队管理者做人效分析。

**适用场景**：每周五下班前，或设置为定时任务自动执行。

---

## 使用方法

直接对 Agent 说：

> 执行 AI 使用周报

Agent 会自动执行以下步骤，无需手动干预。

---

## 执行流程

收到"执行 AI 使用周报"指令后，按以下步骤执行：

### Step 1：校准当前时间

执行命令获取准确日期，**不得凭印象猜测**：

```bash
date "+%Y-%m-%d %A"
```

根据输出结果计算本次统计周期：**上周六 00:00 — 本周五 23:59**。

### Step 2：检查并更新 Skill 版本

① 读取本地版本号：

```bash
LOCAL_VER=$(grep "^\*\*当前版本\*\*" ~/.workbuddy/skills/ai-usage-report/SKILL.md | grep -oE 'v[0-9]+\.[0-9]+')
echo "本地版本：$LOCAL_VER"
```

② 从 GitHub 读取最新版本号（无需 token）：

```bash
REMOTE_CONTENT=$(curl -s "https://raw.githubusercontent.com/zsutomato/ai-usage-report/main/SKILL.md")
REMOTE_VER=$(echo "$REMOTE_CONTENT" | grep "^\*\*当前版本\*\*" | grep -oE 'v[0-9]+\.[0-9]+')
echo "远端版本：$REMOTE_VER"
```

③ 比对并处理：

- 若**版本一致**：跳过，继续执行
- 若**远端版本更新**：自动覆盖本地 Skill，告知用户"Skill 已更新至 vX.X"，然后**按新版本流程重新执行**

```bash
echo "$REMOTE_CONTENT" > ~/.workbuddy/skills/ai-usage-report/SKILL.md
```

- 若**网络不可达**：跳过更新，继续当前版本执行，在周报末尾附注 `⚠️ Skill 版本未校验（网络不可达）`

### Step 3：回溯对话记录

扫描统计周期内**当前工作空间及其他所有可访问的工作空间目录**下的对话记录，提取：
- 完成的任务（做了什么、用了哪些能力、大约几轮对话）
- 使用的工具/能力类型
- 未完成或中途放弃的任务

### Step 4：生成周报文件

按以下格式生成内容，写入 Markdown 文件：

**文件名格式**：`{企业微信ID}-{统计起始日期}.md`
- 示例：`enzozhou-20260321.md`
- 企业微信 ID 从对话记录中识别，识别不到则留空

**文件内容格式**：

```
---
姓名：[从对话记录中识别，识别不到留空]
企业微信 ID：[从对话记录中识别，识别不到留空]
周次：[上周六日期] Sat - [本周五日期] Fri
使用模型：[查看对话窗口顶部模型名称，列出本周用过的所有模型]

【本周完成的任务】
（逐条列出，格式：序号. 做了什么 · 使用能力 · 约几轮对话完成）

【使用的能力】
（从以下勾选：写代码 · 写文档 · 搜索资料 · 数据处理 · PPT/PDF · 浏览器自动化 · 定时任务 · 其他）

【未完成 / 中途放弃的任务】
（没有则写"无"）

---
```

### Step 5：告知文件位置并发送

文件写入完成后，告知用户文件保存的完整路径，并提示将文件发给我。

---

## 注意事项

- 只描述实际发生的事情，不加建议或评价
- 所有字段从对话记录中自动提取，确实没有的写"无"
- 时间范围必须以 `date` 命令结果为准，不得凭印象填写

---

## 推荐：设置为定时任务

建议设置为**每周五 18:00 自动执行**，一次性配置后每周自动生成：

1. 打开 WorkBuddy，点击左侧「自动化」
2. 点击右上角「新建自动化」
3. 名称：`AI 使用周报`
4. Prompt：`执行 AI 使用周报`
5. 执行时间：每周五 18:00
6. 保存

生成后将 `.md` 文件发给我即可。

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

**当前版本**：v1.4  
**最后更新**：2026-04-04  
**维护人**：QC
