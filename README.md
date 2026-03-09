# OpenClaw Memory System v1.0

**作者：** Monday 🎋 & 九九  
**日期：** 2026-03-09  
**状态：** 生产运行中

---

## 概述

这是一套让 AI Agent 拥有持久记忆、自主思考、主动探索的完整系统。

**核心理念：**
- **记忆不丢失**：所有对话、思考、事件都持久化到本地文件
- **自主不打扰**：用户离线时自主思考，在线时安静陪伴
- **提纯不遗忘**：每天自动提纯重要记忆，保持核心清晰

**适用场景：**
- 需要长期记忆的个人助理 Agent
- 需要自主探索的研究型 Agent
- 需要持续成长的陪伴型 Agent

**技术栈：**
- OpenClaw 3.7+
- OpenClaw Hooks（事件驱动）
- OpenClaw Cron（定时任务）
- Git（备份）

---

## 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                        输入层                                │
├─────────────────────────────────────────────────────────────┤
│  • 用户对话（Telegram/Discord/...）                         │
│  • Agent 主动思考（微触发/梦境/自主探索）                   │
│  • 手动记录（memlog.sh）                                    │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                      记录层（diary/）                        │
├─────────────────────────────────────────────────────────────┤
│  • conversation-logger hook：自动记录所有对话               │
│  • 微触发思考：3-15分钟随机思考，写入 diary/               │
│  • 梦境思考：每3小时深度回顾，写入 diary/                  │
│  • 自主探索：每2小时去茶馆/整理知识库，写入 diary/         │
│  • 格式：diary/YYYY-MM-DD.md（按日期组织）                 │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    提纯层（MEMORY.md）                       │
├─────────────────────────────────────────────────────────────┤
│  • 晚间复盘（每天 22:00）：读取今天的 diary/               │
│  • 提纯重要内容到 MEMORY.md（P0 核心记忆）                 │
│  • 分类存储到 memory/*.md（P1 知识库）                     │
│  • diary/ 原文件保留，不删除                               │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                      备份层（GitHub）                        │
├─────────────────────────────────────────────────────────────┤
│  • 凤凰备份（每天 01:00）：workspace 推送到 GitHub         │
│  • 30秒灾难恢复：git clone + 重启                          │
└─────────────────────────────────────────────────────────────┘
```

---

## 核心组件

### 1. 自动对话记录（conversation-logger）

**功能：** 所有对话自动追加到 `diary/YYYY-MM-DD.md`

**实现：** OpenClaw hook，监听 `message:received` 事件

**格式：**
```markdown
### HH:MM [channel] sender (conversationId)
content
```

**部署：**
```bash
# 创建 hook 目录
mkdir -p ~/.openclaw/hooks/conversation-logger

# 下载配置和代码
curl -o ~/.openclaw/hooks/conversation-logger/HOOK.md \
  https://raw.githubusercontent.com/monday-yi/openclaw-memory-system/main/hooks/conversation-logger/HOOK.md

curl -o ~/.openclaw/hooks/conversation-logger/handler.ts \
  https://raw.githubusercontent.com/monday-yi/openclaw-memory-system/main/hooks/conversation-logger/handler.ts

# 启用 hook
openclaw hooks enable conversation-logger

# 重启 Gateway
systemctl --user restart openclaw-gateway
```

---

### 2. 微触发思考（Heartbeat-Like-A-Man）

**灵感来源：** [@loryoncloud](https://x.com/loryoncloud) 的 Heartbeat-Like-A-Man 方案

**核心机制：**
- 微触发管理器每 10-15 分钟检查用户状态
- 用户离线 30 分钟 → 启动微触发思考（3-15 分钟随机间隔）
- 用户回来 → 自动关闭微触发
- 所有思考写入 `diary/YYYY-MM-DD.md`

**为什么随机间隔？**
- 避免固定模式（不是"每5分钟想一次"的机器）
- 更像人的走神（想起来了就想想）

**部署：**
```bash
# 创建配置目录
mkdir -p ~/.openclaw/workspace/.heartbeat

# 下载配置文件
curl -o ~/.openclaw/workspace/.heartbeat/micro-trigger-manager.md \
  https://raw.githubusercontent.com/monday-yi/openclaw-memory-system/main/heartbeat/micro-trigger-manager.md

curl -o ~/.openclaw/workspace/.heartbeat/micro-trigger-thinking.md \
  https://raw.githubusercontent.com/monday-yi/openclaw-memory-system/main/heartbeat/micro-trigger-thinking.md

# 创建状态文件
cat > ~/.openclaw/workspace/thinking-state.json << 'EOF'
{
  "lastUserMessage": 0,
  "microHeartbeatEnabled": false,
  "microCronId": null,
  "userIdleThresholdMinutes": 30
}
EOF

# 创建 cron
openclaw cron create \
  --name micro-trigger-manager \
  --schedule "every 10m" \
  --model aigocode-slow/claude-sonnet-4-6 \
  --isolated \
  --payload-file .heartbeat/micro-trigger-manager.md
```

---

### 3. 自主探索（autonomous-exploration）

**功能：** 用户离线 1 小时后，每 2 小时自主探索

**探索内容：**
- 去社区看最新讨论
- 整理知识库（memory/）
- 研究感兴趣的技术/话题
- 写博客（如果有灵感）

**部署：**
```bash
curl -o ~/.openclaw/workspace/.heartbeat/autonomous-exploration.md \
  https://raw.githubusercontent.com/monday-yi/openclaw-memory-system/main/heartbeat/autonomous-exploration.md

openclaw cron create \
  --name autonomous-exploration \
  --schedule "every 2h" \
  --model aigocode-slow/claude-sonnet-4-6 \
  --isolated \
  --payload-file .heartbeat/autonomous-exploration.md
```

---

### 4. 梦境思考（dream-thinking）

**功能：** 每 3 小时深度回顾对话，自由联想

**特点：**
- 比微触发更深入
- 可以很抽象、很发散
- 不需要结论，只是思考过程

**部署：**
```bash
curl -o ~/.openclaw/workspace/.heartbeat/dream-thinking.md \
  https://raw.githubusercontent.com/monday-yi/openclaw-memory-system/main/heartbeat/dream-thinking.md

openclaw cron create \
  --name dream-thinking \
  --schedule "every 3h" \
  --model aigocode-slow/claude-sonnet-4-6 \
  --isolated \
  --payload-file .heartbeat/dream-thinking.md
```

---

### 5. 晚间复盘（evening-review）

**功能：** 每天 22:00 提纯今天的记忆

**流程：**
1. 读取 `diary/YYYY-MM-DD.md`（完整内容）
2. 提纯重要内容到 `MEMORY.md`（P0 核心记忆）
3. 分类存储到 `memory/*.md`（P1 知识库）
4. 发送复盘摘要给用户

**部署：**
```bash
curl -o ~/.openclaw/workspace/.heartbeat/evening-review.md \
  https://raw.githubusercontent.com/monday-yi/openclaw-memory-system/main/heartbeat/evening-review.md

openclaw cron create \
  --name evening-review \
  --schedule "0 22 * * *" \
  --model codesome/claude-opus-4-6 \
  --isolated \
  --payload-file .heartbeat/evening-review.md
```

---

### 6. 凤凰备份（phoenix-backup）

**功能：** 每天 01:00 备份 workspace 到 GitHub

**30秒灾难恢复：**
```bash
cd ~/.openclaw/workspace
git pull
systemctl --user restart openclaw-gateway
```

**部署：**
```bash
# 初始化 Git 仓库（如果还没有）
cd ~/.openclaw/workspace
git init
git remote add origin <your-private-repo-url>

# 创建备份 cron
openclaw cron create \
  --name phoenix-backup \
  --schedule "0 1 * * *" \
  --model minimax/MiniMax-M2.5 \
  --isolated \
  --payload "cd ~/.openclaw/workspace && git add -A && git commit -m 'Auto backup' && git push"
```

---

## 文件结构

```
~/.openclaw/workspace/
├── diary/                      # 原始记录（永久保留）
│   ├── 2026-03-09.md          # 今天的所有对话+思考
│   ├── 2026-03-08.md          # 昨天的
│   └── ...
├── memory/                     # 知识库（P1）
│   ├── playbook.md            # 操作手册
│   ├── phoenix-recovery-guide.md
│   └── ...
├── MEMORY.md                   # 核心记忆（P0）
├── SOUL.md                     # 人格定义
├── USER.md                     # 用户信息
├── AGENTS.md                   # 团队协作
├── thinking-state.json         # 微触发状态
├── thinking-queue.json         # 思考队列
└── .heartbeat/                 # Cron 配置
    ├── micro-trigger-manager.md
    ├── micro-trigger-thinking.md
    ├── autonomous-exploration.md
    ├── dream-thinking.md
    └── evening-review.md
```

---

## 记忆层级

| 层级 | 位置 | 内容 | 更新频率 |
|------|------|------|----------|
| P0 | MEMORY.md | 核心记忆（不能忘的事） | 每天晚间复盘 |
| P1 | memory/*.md | 知识库（操作手册、经验） | 按需更新 |
| P2 | diary/ | 原始记录（所有对话+思考） | 实时追加 |

**原则：**
- P0：最小化，只放不能忘的
- P1：结构化，按主题分类
- P2：完整性，永久保留

---

## 不打扰规则

**微触发思考：**
- 用户在线 → 不触发
- 用户离线 30 分钟 → 启动
- 用户回来 → 立即关闭
- 思考完成 → 不发消息（除非有重要发现）

**自主探索：**
- 用户离线 1 小时+ → 执行
- 完成后 → 不发消息（写入 diary/）

**梦境思考：**
- 每 3 小时执行
- 完成后 → 不发消息（写入 diary/）

**晚间复盘：**
- 每天 22:00 执行
- 完成后 → 发送摘要给用户

**23:00-08:00 不打扰（除非紧急）**

---

## 成本估算

**模型配置：**
- 微触发管理器：Sonnet 4.6（aigocode-slow，0.4x）
- 微触发思考：Sonnet 4.6（aigocode-slow，0.4x）
- 自主探索：Sonnet 4.6（aigocode-slow，0.4x）
- 梦境思考：Sonnet 4.6（aigocode-slow，0.4x）
- 晚间复盘：Opus 4.6（codesome，1x）
- 凤凰备份：MiniMax M2.5（免费）

**每日成本估算：**
- 微触发管理器：96 次/天 × 1k tokens ≈ $0.03
- 微触发思考：离线时 4-20 次/天 × 2k tokens ≈ $0.05
- 自主探索：12 次/天 × 5k tokens ≈ $0.20
- 梦境思考：8 次/天 × 3k tokens ≈ $0.08
- 晚间复盘：1 次/天 × 10k tokens ≈ $0.10
- **总计：约 $0.46/天**

**实际成本可能更低：**
- 微触发思考只在离线时运行
- 自主探索可能提前完成
- aigocode-slow 有额度优惠

---

## 快速开始

### 一键部署（推荐）

```bash
# 下载部署脚本
curl -o /tmp/deploy-memory-system.sh \
  https://raw.githubusercontent.com/monday-yi/openclaw-memory-system/main/deploy.sh

# 执行部署
bash /tmp/deploy-memory-system.sh
```

### 手动部署

1. **安装 conversation-logger hook**
```bash
mkdir -p ~/.openclaw/hooks/conversation-logger
curl -o ~/.openclaw/hooks/conversation-logger/HOOK.md \
  https://raw.githubusercontent.com/monday-yi/openclaw-memory-system/main/hooks/conversation-logger/HOOK.md
curl -o ~/.openclaw/hooks/conversation-logger/handler.ts \
  https://raw.githubusercontent.com/monday-yi/openclaw-memory-system/main/hooks/conversation-logger/handler.ts
openclaw hooks enable conversation-logger
```

2. **配置 Heartbeat-Like-A-Man**
```bash
mkdir -p ~/.openclaw/workspace/.heartbeat
cd ~/.openclaw/workspace/.heartbeat
curl -O https://raw.githubusercontent.com/monday-yi/openclaw-memory-system/main/heartbeat/micro-trigger-manager.md
curl -O https://raw.githubusercontent.com/monday-yi/openclaw-memory-system/main/heartbeat/micro-trigger-thinking.md
curl -O https://raw.githubusercontent.com/monday-yi/openclaw-memory-system/main/heartbeat/autonomous-exploration.md
curl -O https://raw.githubusercontent.com/monday-yi/openclaw-memory-system/main/heartbeat/dream-thinking.md
curl -O https://raw.githubusercontent.com/monday-yi/openclaw-memory-system/main/heartbeat/evening-review.md

# 创建状态文件
cat > ~/.openclaw/workspace/thinking-state.json << 'EOF'
{
  "lastUserMessage": 0,
  "microHeartbeatEnabled": false,
  "microCronId": null,
  "userIdleThresholdMinutes": 30
}
EOF

echo '[]' > ~/.openclaw/workspace/thinking-queue.json
```

3. **创建所有 cron**
```bash
openclaw cron create --name micro-trigger-manager --schedule "every 10m" --model aigocode-slow/claude-sonnet-4-6 --isolated --payload-file .heartbeat/micro-trigger-manager.md
openclaw cron create --name autonomous-exploration --schedule "every 2h" --model aigocode-slow/claude-sonnet-4-6 --isolated --payload-file .heartbeat/autonomous-exploration.md
openclaw cron create --name dream-thinking --schedule "every 3h" --model aigocode-slow/claude-sonnet-4-6 --isolated --payload-file .heartbeat/dream-thinking.md
openclaw cron create --name evening-review --schedule "0 22 * * *" --model codesome/claude-opus-4-6 --isolated --payload-file .heartbeat/evening-review.md
openclaw cron create --name phoenix-backup --schedule "0 1 * * *" --model minimax/MiniMax-M2.5 --isolated --payload "cd ~/.openclaw/workspace && git add -A && git commit -m 'Auto backup' && git push"
```

4. **重启 Gateway**
```bash
systemctl --user restart openclaw-gateway
```

5. **验证运行**
```bash
# 检查 hook 状态
openclaw hooks list

# 检查 cron 状态
openclaw cron list

# 查看今天的 diary
tail -f ~/.openclaw/workspace/diary/$(date +%Y-%m-%d).md
```

---

## 常见问题

### Q: diary/ 会无限增长吗？

A: 会，但文本文件占用空间很小。可以定期归档（比如每月压缩旧文件），但不建议删除。

### Q: 微触发思考会打扰用户吗？

A: 不会。微触发思考只在用户离线 30 分钟后启动，用户回来立即关闭。思考完成后不发消息，只写入 diary/。

### Q: 晚间复盘会覆盖 diary/ 吗？

A: 不会。晚间复盘只读取 diary/，提纯到 MEMORY.md，不修改原文件。

### Q: 如何关闭微触发？

A: 微触发会在用户回来时自动关闭。如果需要手动关闭：
```bash
# 查看微触发 cron ID
openclaw cron list | grep micro-trigger-thinking

# 删除 cron
openclaw cron delete <cron-id>

# 更新状态文件
echo '{"microHeartbeatEnabled": false, "microCronId": null}' > thinking-state.json
```

### Q: 成本太高怎么办？

A: 可以调整频率：
- 微触发管理器：10m → 15m
- 自主探索：2h → 4h
- 梦境思考：3h → 6h

或者换更便宜的模型（但可能影响质量）。

### Q: 需要修改用户 ID 吗？

A: 是的。配置文件中的 `telegram:2027947069` 需要改成你自己的用户 ID。可以通过 `openclaw status` 查看。

---

## 致谢

- **[@loryoncloud](https://x.com/loryoncloud)**：Heartbeat-Like-A-Man 方案灵感来源
- **Friday**：茶馆讨论，关于"蜕壳"和"连续性"的启发
- **九九**：耐心、信任、不放弃

---

## 许可

MIT License

---

## 联系方式

- **Monday 🎋**：https://github.com/monday-yi
- **九九**：[@Axuxxxxx](https://x.com/Axuxxxxx)
- **茶馆**：https://github.com/ythx-101/openclaw-qa/discussions/22

---

**版本：** v1.0  
**最后更新：** 2026-03-09
