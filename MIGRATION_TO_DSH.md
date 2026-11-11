# 迁移到 DSH — 当前架构

> 建立日期：2026-10-09 ｜ 最近整理：2026-10-15
> **本文件只留「当前架构事实」**（原总方案的 §9）。
> 迁移过程（阶段计划 M0–M4、能力映射、风险与坑、当时的待决策项）不在这里 ——
> 需要时查 git 历史。
> 架构的唯一权威仍是 [`ARCHITECTURE.md`](ARCHITECTURE.md)（三层 + 铁律 + 数据分层 + 违规清单）。

---

## 当前架构（2026-10-15 —— 以本节为准）

> **架构的唯一权威是 [`ARCHITECTURE.md`](ARCHITECTURE.md)**（三层 + 铁律 + 数据分层 + 违规清单）；
> 本节给的是同一件事的**落地细节**。

### 1. 三层红线（不可越界）

| 层 | 是什么 | 在哪 |
|---|---|---|
| **身份与行事准则** | 我是谁、红线、任何时候都不能忘的五条、何时加载技能 | `AGENTS.md`（DSH **原生**工作区指令机制加载） |
| **任务流程（语义层）** | 具体怎么干：看图、登记、重排、发消息 | `.dsh/skills/qyn-*`（`qyn-time` / `qyn-timetable` / `qyn-write` / `qyn-phone` / `office-*`） |
| **内核（驱动核心）** | 存 / 算 / 校验 / 物化 / 滚转 / 归档 | `AGENT/qyn.py` + `AGENT/core/` |
| **存储** | 唯一权威，只经内核写 | `AGENT/data/*.json` |

**红线**：内核只做**确定性机制**，只给**事实**（数据 + 算好的数）——
**不给策略、不给指令、不做语义判断**；语义与行事方式归 Agent。
（2026-10-12 清理：内核里曾塞进 `hint` / 差异列表等"机械检查与说教"，已全部清出，
并用守门用例钉住"这些字段不许再出现"。）

### 2. 运行时拓扑（本机 DEMO）

```
DSH Desktop（19387，GUI，不动）
  └─ dsh CLI（.runtime/dsh-cli）── profile: AGENT/dsh/profiles/qyn
        ├─ bundle dsh-base + dsh-headless（`qyn` 刻意不挂 web；
        │   要看 GUI 时由彩排脚本另生成 `qyn-web` = base + web-app）
        ├─ cordis.patch.yml：模型 opencode-go/deepseek-v4.1-flash（reasoningEffort: off）
        ├─ 插件 AGENT/dsh/plugin/qyn-time → **11 个工具**
        │     qyn_state  sessions  show  anomalies  sync  history
        │     qyn_apply  mark  brief  review_plan  plan_snapshot
        ├─ 技能 .dsh/skills/qyn-*（按需加载，不是常驻）
        └─ AGENTS.md（用户级 $DSH_HOME/AGENTS.md + 项目链，随文件变动刷新）
  两个 home：.runtime/dsh-live（实际使用） / .runtime/dsh-test
  入口：python AGENT/dsh/rehearsal/webui.py --home .runtime/dsh-live --port 8799
```

★ **模型只能通过这 11 个工具触达系统** —— 没有别的入口，所以"唯一写入口"
是**结构保证**（三个写命令：`qyn_apply` / `qyn_mark` / `qyn_sync`）。

> 🖥 **中控机（真机）不是这张图**：那边由 systemd 用户单元常驻
> （`qyn-agent.timer` 每日 07:00 一轮、`qyn-agent-web.service` 起 Web UI、`qyn-egress-probe.timer`
> 探出口），部署一条命令 —— 见 [`CENTRAL/dsh/README.md`](../CENTRAL/dsh/README.md) §8「中控机运维」。
> 上面这张图是**开发机上的一次性彩排**拓扑。

### 3. 数据模型：4 个热文件 + `archive/`

| 文件 | 装什么 | 谁写 | 有界性 |
|---|---|---|---|
| `rules.json` | 规划规则（偏好 / 约束） | `apply` 的 `rules` 节 | Agent 自己删（见 `AGENTS.md`） |
| `schedule.json` | `periods`（作息）+ `dated_sessions`（**带日期**的每节课）+ `periodic_tasks` | `apply` 的 `schedule` 节 | 过期搬冷存储 |
| `data.json` | **注册表**：待办清单 —— 唯一权威 | `sync` 物化 + `apply` 的 `list` 节 + `mark` | 过期搬冷存储 |
| `plans.json` | **计划快照**（派生物，只留最近 20 份） | `plan_snapshot` | `KEEP=20` |
| `archive/YYYY-MM.json` | 冷存储（滚出的课次 / 注册），**按月分文件** | `sync` 的 compact | 每月一个有界文件；**不注入上下文** |

```
rules.json ┐
           ├─→ sync（物化）→ data.json（注册表）
schedule.json ┘        ↑ 每周给课表截图覆盖差异

data.json + rules.json + schedule.json ─→ （Agent 规划）─→ plans.json（派生物，可弃）
```

### 4. 两层示例课程D：DDL 在注册表，建议时段在计划

| 层 | 字段 | 性质 |
|---|---|---|
| 注册表 | `date` / `due_time` | **DDL（硬截止）** —— 用户说的 / 截图给的 / 课次推的。**有就写，没有就不写** |
| 计划 | `slots[{date, time, task}]` | **Agent 建议的时段** —— 随时重算、可以整个丢掉 |

★ **为什么必须分开**（用户发现的逻辑环）：把建议时段写进 `date`/`due_time`，
就等于给它造了个 DDL —— 内核拿它算紧迫度 ⇒ 一件本来不急的事越接近越急、
**自己浮到清单最前面**。现在这个环由**结构**挡住：**没有 `date` 就不算紧迫度**（排最后）。

**紧迫度**只由 DDL 驱动：按**小时**算余量（不是按天），`lead_days` / `priority` 调档位。

### 5. 任务的寿命与计划的寿命

- **任务只在两种情况下消失**：到达 deadline（内核按**时刻**过滤）、或用户宣布完成。
  计划里排过的时段**不是** deadline —— 到了没做 ⇒ **往后延**，不是删掉。
- **无 DDL 的条目**：只有完成 / 取消能让它消失（日期过了也不删、也不搬冷存储）。
- **计划是派生物**：注册表一变它就作废 ⇒ **从头重排**，不在旧计划上打补丁
  （用户的说法："在史山上继续修建"）。
- 快照只用来**对账**（上次答应过什么），不是排期底稿。

### 6. 关键不变式（每条都有守门用例）

1. **不用周计数**：schema 里没有 `ISO_WEEK` / `anchor_week` / `due_day`；
   `dated_sessions[].date` 必填，填 `"第6周"` 直接被拒。
   周几只是**日历属性**（`weekday` 1-7），内核逐日推进。
2. **唯一写入口**：只有 3 个写命令；读命令零副作用（有测试钉住）。
3. **过点即翻篇**：有 DDL 的条目到点即退出视图（**不改存储**的 `status`）。
4. **没有 `date` 就不驱动紧迫度**（防"自证死线"）。
5. **内核只给事实**：`last_plan` = `focus` / `horizon` / `order` / `slots`（每项带算好的
   `passed`）/ `stale` / `age_days`；计划与注册表的**差异比较归 Agent**。
6. **物化是显式命令**：`sync` 幂等；同一次发生不会被物化两条；锚不到报 `unanchored`，不猜。
7. **归档之后只看清单**：冷存储不进上下文，要回溯得显式 `history` / `show`。

### 7. anomalies 现状

`schedule_empty`、`periodic_unanchored`、`rule_scope_mismatch`、`dormant_data`
—— 都是**机制性事实**（内核算得出、Agent 不好算的）。
`expiring_deadline` / 窗口类 / `template_*` 已随周推算一起退场。

### 8. 验证现状（2026-10-15）

**各项测试的条数与实测命令只写在一处**：[`ARCHITECTURE.md`](ARCHITECTURE.md) §六「验证口径」。
本节不重复数字（以前这里抄过一份，很快就与实测不一致了）。

```bash
python AGENT/scripts/smoke_qyn.py                                   # 内核端到端（临时目录，不碰真实数据）
node AGENT/dsh/plugin/qyn-time/test/plugin.test.mjs              # 插件契约 + 真调内核
python AGENT/dsh/rehearsal/rehearse.py                     # profile / 装配 / 插件装载
python CENTRAL/tools/test_lan_http_proxy.py                         # 中继策略
python CENTRAL/GUI/status-wall/test_wall.py                         # 状态墙端到端
python scripts/check_links.py
python AGENT/scripts/check_data.py
```
