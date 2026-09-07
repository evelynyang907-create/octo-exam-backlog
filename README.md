# octo-server 考试需求池

本仓库是 octo-server B卷考试的需求池（backlog），由两个AI Agent协作维护。

## 架构设计

### 双Agent协作模式

本次搭建了两个独立Agent，通过 GitHub issue状态/label/评论协作，不通过群聊直接对话。

**设计原则：共享事实，隔离角色；共享工作对象，隔离运行状态。**

| Agent | 职责 | 触发方式 |
|-------|------|----------|
| **📋 产品管家** | 产品问答（带源码引用）、接收bug/feature反馈、追问确认、建issue归档、写PRD、根据review意见修改PRD | 群内@触发 |
| **🔍 项目监理** | cron每5分钟自主巡检、自动review in-review状态的PRD（三条标准）、发现状态变化主动汇报、无变化完全静默 | cron定时触发（every 5m），不依赖@通知 |

### 协作流程

```
用户提bug/feature → 产品管家追问/复述确认 → 建issue(triage)
                                          ↓
                              产品管家被@写PRD → 补背景/场景/验收标准 → 改label in-review
                                          ↓
                         项目监理cron自动发现in-review PRD
                        ↙                      ↘
              PRD合格→改label approved      PRD不合格→评论逐条指出问题
              群汇报"Review通过"             →改label doing → 群汇报"Review打回"
                                                       ↓
                                              产品管家读取评论修改PRD
                                              →重新in-review → 监理cron再审
```

### 项目监理Review三条铁律

1. **禁止写How**：PRD里出现技术实现细节（用什么库/加什么字段/贴代码）→ 打回
2. **验收标准必须用户视角**："用户3秒内看到提示"通过，"接口返回200"→ 打回
3. **信息完整**：背景/用户场景/验收标准三块缺一块→ 打回指出缺什么

### 汇报规则

- 所有cron汇报消息格式极简：emoji+一句话+链接（单独一行）+@相关人+@主考
- 链接单独占一行，避免IM客户端把后续文字拼进URL
- 无变化零输出（前置shell脚本判断，没事实根本不唤醒大模型）

## Label体系

三层分类，共12个label：

| 层 | label |
|----|-------|
| 类型 | `bug` / `feature` |
| 优先级 | `P0` / `P1` / `P2` |
| 状态 | `triage` → `doing` → `in-review` → `approved` → `done` / `wontfix` |

## 知识库

- 覆盖9个域：认证/鉴权/配置/模块/API/IM控制面/Bot/存储/构建
- 基于 octo-server commit `49dc9fd` 整理，所有路径行号引用锁定此版本
- 闭卷考试规则：产品问答只读知识库（.md文件），不现场grep源码，保证引用稳定可核验
- 每条结论带来源：`来源: <源码路径>#L<起>-L<止>`
- 增量更新：维护者手动更新知识库文档+锁commit hash，不自动更新

## 静默铁律

项目监理遵循最严格的静默规则：
- 没有需要汇报的事项 → 绝对零输出
- 不发"巡检完成"/"无变化"/"No changes"等过程消息
- 不@就不说话（群聊内产品管家同样遵守）
- 实现方式：cron触发时先执行shell前置脚本巡检GitHub，无变化stdout为空→agent输出NO_REPLY，不唤醒大模型

## 文件结构

```
exam-shared/                    # 两个Agent共享资源（只读事实）
├── octo-server/               # octo-server源码（锁commit 49dc9fd）
├── knowledge-base/            # 产品知识库（9域+INDEX+README）
│   ├── INDEX.md
│   ├── 01-认证与身份.md
│   ├── 02-鉴权模型.md
│   ├── 03-配置.md
│   ├── 04-业务模块清单.md
│   ├── 05-API与错误约定.md
│   ├── 06-IM控制面.md
│   ├── 07-Bot与Agent.md
│   ├── 08-存储与外部依赖.md
│   ├── 09-构建与发布.md
│   └── README.md             # 知识库说明与增量更新机制
└── requirement-repo/
    ├── scripts/
    │   ├── gh-api.sh          # GitHub API封装（带节流）
    │   └── inspector-cron.sh  # cron前置巡检脚本
    ├── config/github-token    # GitHub PAT（600权限，不输出）
    └── snapshots/repo-config.json  # repo地址+主考mention配置

workspace-product-steward/     # 产品管家独立workspace
├── SOUL.md                    # 人设+职责+流程指令
├── AGENTS.md                  # 群聊行为铁律
└── IDENTITY.md

workspace-inspector/           # 项目监理独立workspace
├── SOUL.md                    # 人设+review标准+汇报规则
├── AGENTS.md                  # cron执行流程+静默铁律+汇报格式
├── IDENTITY.md
└── watcher-state/
    └── issue-snapshot.json    # 私有运行状态（issue diff快照）
```

## 考试群

- 群名：octo exam
- 产品管家bot：octo-server产品管家00
- 项目监理bot：octo-server项目管理00
- cron频率：每5分钟
- 所有cron推送目标：octo exam群
