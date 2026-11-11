# 信息中心 · 时间管理值守员

一台塞在角落里、不关机的小主机，替你把**时间**记着，并在该动手的时候提醒你——
可以理解为「作业、课表、DDL 这些东西，谁替我记着并每天发到微信里」的那层服务。

它连点都不用点：早上 07:00、下午 14:00 各跑一轮，先发今天的课（一行一节：时间 · 课名 · 教室），
再给规划；你随手回一句「论文改到 15 号」「周五停课」「某门课的报告交了」，
它自己登记、重排、存快照。**主打一个不用你动手——微信里说一句就行。**

## 几条踩过坑才立下的规矩

- 它**不心算日期**。日期、余量、课次身份、归档全由内核算好给它——硬算日期时，校园调休会教它做人。
- 它**不允许手改 json**。写操作只走 4 个工具（`apply` / `mark` / `sync` / `plan-snapshot`）。
- **语义归模型，机制归代码**。算不算"同一件事"、该不该提前准备、排不下怎么办，由模型判断；
  代码只管存取、算术、校验、归档——一行语义判断都不许进内核。
- **第一条是"排不上就说排不上"**：排到截止之后等于没排，它会直接说卡在哪、
  最晚什么时候必须动手，不假装排好了。

## 结构

三层：控制层（DSH 运行时，只做装配、零语义代码）／数据工具层（`qyn.py` + 12 个工具，只存不算）／
智能体层（模型 + 准则 + 技能，语义全在这）。课表按**绝对日期**存，一次给一学期也行；
物化窗口 14 天、按月归档，文件不会越用越大。

## 仓库

| 仓库 | 作用 |
|---|---|
| [qyn-kernel](https://github.com/Prqingyu/qyn-kernel) | 内核：唯一写入口、零语义 |
| [qyn-dsh-plugin](https://github.com/Prqingyu/qyn-dsh-plugin) | 把内核封成 12 个 DSH 工具 |
| [qyn-skills](https://github.com/Prqingyu/qyn-skills) | 技能与准则（语义层，`AGENTS.md` 在这） |
| [qyn-phone](https://github.com/Prqingyu/qyn-phone) | 手机通道（微信收发 + 附件） |
| [qyn-central](https://github.com/Prqingyu/qyn-central) | 中控机装配（部署脚本 / systemd / 看板） |
| [qyn-docs](https://github.com/Prqingyu/qyn-docs) | 文档（架构 / 迁移 / 路线图） |
| [qyn-opencode-go](https://github.com/Prqingyu/qyn-opencode-go) | 顺带一个查模型端点额度的 |

隔壁项目：[bmc-tool](https://github.com/Prqingyu/bmc-tool)（带外管理）。

想先看它怎么想事儿的，从 [`ARCHITECTURE.md`](ARCHITECTURE.md) 进去——
十分钟就知道为什么「不许用代码做语义判断」值得单独开一份文档。

## 许可

MIT（见 `LICENSE`）。**是真 MIT，没有"限制商用"那条。**

唯一的要求写在人情上：别把课表接口挂校园墙卖同学，也别把中控机的内网入口截图发班群——
那台机器上还挂着看板和别的东西。
