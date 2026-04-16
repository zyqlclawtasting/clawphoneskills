---
name: lark-auto-reply
description: 使用 lark-cli 搭建飞书自动回复与定时监控流程（联系人私聊或群聊）。当用户要求“开发/整理飞书自动回复 skill”“监控指定联系人或群聊并自动回复”“梳理 lark-cli 安装配置与授权流程”“做消息检索+判断+自动发送闭环”时使用。包含稳定授权（避免仅拿到授权链接就提前退出）、目标解析、消息拉取、判定去重、发送回复、cron 任务接入。
---

# Lark Auto Reply

## Quick Start

0. 先做安装/配置前置检查（新增机制）：
   ```bash
   bash scripts/preflight_check.sh
   ```
   - 若提示未安装 `lark-cli`：先走 `lark-cli-setup` skill 完成安装
   - 若提示未配置或 token 无效：按提示先完成 `config init` 与授权

1. 跑稳定授权（必须等到授权完成，不是拿到链接就结束）：
   ```bash
   bash scripts/auth_login_stable.sh
   ```
2. 跑一次“监控+判断+自动回复”验证闭环（优先用当前登录用户信息做回归）：
   ```bash
   python3 scripts/lark_auto_reply_once.py --user-query "<当前登录用户姓名>" --history-limit 50
   ```
3. 需要群聊时改为（优先选当前用户已加入的群）：
   ```bash
   python3 scripts/lark_auto_reply_once.py --chat-query "<当前用户所在群聊关键词>" --history-limit 50
   ```

---

## 稳定授权流程（核心约束）

### 背景问题
不同模型/执行器稳定性不同：
- 好的执行器会在出现授权链接后持续等待用户授权，并在授权后保持流程完成；
- 不稳定执行器会在“检测到链接”后直接退出，导致假成功。

### 必做约束
- 登录完成条件必须是：
  - `lark-cli auth status` 返回 `tokenStatus=valid`；
  - 建议再过 `lark-cli doctor`。
- “出现授权链接”只代表授权流程开始，不代表完成。
- 若中途退出，必须重试直到状态有效或超时。

### 推荐执行方式
- 用可持续观察输出的方式执行（例如会话/进程轮询）；
- 或直接使用本 skill 的 `scripts/auth_login_stable.sh`。

---

## 工作流：监控 -> 判断 -> 回复

### Step 1) 解析目标
- 联系人：`lark-cli contact +search-user --query "关键词"`
- 群聊：`lark-cli im +chat-search --query "群名关键词"`

### Step 2) 拉取最新消息
- 私聊：`lark-cli im +chat-messages-list --user-id <open_id> --page-size 50 --sort desc`
- 群聊：`lark-cli im +chat-messages-list --chat-id <chat_id> --page-size 50 --sort desc`

### Step 3) 判定是否需要回复
建议规则（可叠加）：
- 明确提问（`?` / `？`）；
- 命中关键词（如：自动回复、怎么实现、流程、卡住）；
- 群里点名你（@你 / 指名）；
- 排除删除消息、系统消息、自己发出的消息。

### Step 4) 去重与冷却
- 按“消息归一化文本+目标”做指纹；
- 同题冷却窗口内跳过（例如 30 分钟 / 24 小时）。

### Step 5) 发送回复
- `lark-cli im +messages-send --as user --user-id ... --text ...`
- 或 `--chat-id ...`
- 推荐末尾固定加：`（本条消息由 ClawPhone 助手代发）`
- **安全约束**：若目标不是“当前对话已明确要求自动发送”的场景，先输出待发送草稿并向用户确认（尤其群聊发送）。
- **换行规范（强制）**：
  - 普通文本消息避免使用字面 `\\n`，优先发送真实换行或改用 `post` 富文本分段；
  - 纯链接场景必须单行发送，链接前后不加空行、不加尾随标点，避免客户端把换行/符号吞进 URL；
  - 正式群发前，先在测试群验证渲染效果（换行、段落、链接可点击）再发送正式消息。

### Step 6) 线下接力与闭环
当对方明确给出“后续需在线下渠道办理（如街道/社区）”时：
- 判定该事项已从飞书沟通阶段进入线下执行阶段；
- 输出简短结论与下一步（由用户线下办理）；
- 若此前为该事项创建了 cron 巡检任务，立即将对应任务 `enabled=false`，避免无效打扰。

---

## 定时任务接入（cron）

`cron` 的 `payload.kind=agentTurn` 文案建议短而可执行，包含：
1) 监控目标（open_id/chat_id）；
2) 拉取范围（最近50条+线程）；
3) 判定规则；
4) 去重窗口；
5) 回复格式要求（1-3段，先结论后步骤）。

常用周期：
- 每天：`0 9 * * *`
- 每小时：`0 * * * *`

### 监控模式（常规/紧急）
- 常规档：5/10/15/30 分钟轮询；仅有新信息时才触发后续动作。
- 紧急档：30 秒轮询，默认仅开 5 分钟，到期自动降级到常规档（默认 10 分钟）。
- 默认策略：
  1) 未明确说明频率/时长时，统一使用常规档 10 分钟；
  2) 仅当用户明确标注“紧急”或明确指定高频参数时，才启用紧急档；
  3) 用户明确指定了频率或持续时长时，按用户指定执行。
- 保护机制（默认开启）：
  1) 时长上限：紧急档达到上限自动降级；
  2) 去重抑制：同一信息仅触发一次；
  3) 速率兜底：命中限流时自动退避到 60~120 秒。

实战规则补充：
- 若用户要求“先验证消息格式再正式发群消息”，先在测试群发送并回读确认换行/排版，再发正式群；
- 默认将“换行异常”视为高优先级格式风险：优先 `post` 富文本分段，避免字面 `\\n`；
- 遇到“联系人接力”场景，下一条外发消息必须带已对接链路摘要（A→B→C），减少转办；
- 已确认事项进入线下执行后，关闭该场景巡检 cron。
- 新信息判定规则：仅当 `message_id` 新增，或（无 `message_id` 场景）`create_time + normalized_text_hash` 变化时，才继续执行自动回复；忽略删除/系统消息、自己发出的消息（含用户手动输入）与纯编辑回写。

---

## Scripts

### `scripts/preflight_check.sh`
安装/配置前置检查脚本。职责：
- 检查 `lark-cli` 是否已安装；
- 检查是否已完成 `config init`（appId 存在）；
- 检查 `auth status` 中 `tokenStatus` 是否为 `valid`；
- 输出明确下一步（安装 / 配置 / 授权）。

### `scripts/auth_login_stable.sh`
稳定授权脚本。职责：
- 检查当前 token；
- 发起 `auth login --no-wait --json` 获取授权信息；
- 持续等待并轮询直至授权成功；
- 最终以 `auth status` 有效为准。

### `scripts/lark_auto_reply_once.py`
单次执行“解析目标 -> 拉消息 -> 判定 -> 去重 -> 自动回复”。

示例（请替换为当前登录用户与其所在群聊，不再使用固定人名/群名）：
```bash
python3 scripts/lark_auto_reply_once.py \
  --user-query "<当前登录用户姓名>" \
  --history-limit 50 \
  --keywords "自动回复,怎么实现,流程,卡住" \
  --signature "（本条消息由 ClawPhone 助手代发）"
```

---

## References

- `references/command-cheatsheet.md`: lark-cli 常用命令清单
- `references/design-notes.md`: 技术选型、失败模式、防呆策略、Skill 扩展建议
