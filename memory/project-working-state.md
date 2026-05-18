# Project Working State

Last updated: 2026-05-18

This file is a handoff note for the next LLM session. It is not a formal design source; confirmed gameplay and implementation rules remain in `docs/`.

## Confirmed In This Session

- 第 1 章 5 个事件节点已经设计并落表：
  - 断裂誓碑
  - 残破军械库
  - 星辉残灯
  - 封存医帐
  - 沉默训练场
- 事件节点的隐藏代价已确认：事件节点不获得战斗节点的金币、药水和常规属性成长。
- “随机 X 或 Y 品质”的规则已确认：从所有满足条件的条目中等权随机，不先抽品质。
- “所有角色属性成长”作用于当前 run 队伍全员。
- 第 1 章节点基础描述和节点自带修正文案已设计并落表。
- 节点候选显示规则已确认：
  - 先确定节点内容，再从该内容绑定的基础描述中选择文案。
  - 相似或接近的节点共享相似描述。
  - 除固定剧情战和每章最终 Boss 外，不允许只有一个节点独占某条基础描述。
  - 完整候选节点文案为“节点基础描述 + 节点自带修正文案”。
  - 节点结束后随机事件不参与候选节点可见修正文案。
- 第 1 章节点结束后随机事件规则已确认：
  - 每个节点完成后 100% 触发 1 次。
  - 当前池内所有随机事件等权抽取。
- 存档规则已确认：
  - MVP 只有 1 个 run 自动存档槽。
  - 每个节点完成后自动存档。
  - 继续游戏只读取自动存档。
  - 战斗中关闭或崩溃，继续游戏回到最近一次节点完成后的自动存档。
  - run 失败清空自动存档，强制从头开始。
  - 第 1 章通关写入局外进度后清空当前 run 自动存档。
- 装备没有耐久度；但装备可以失效。这两者不是一回事。
- 旧的 `repair_equipment` / 装备修理口径已替换：
  - 保留装备失效机制。
  - 节点后随机事件改为 `post_equipment_void_corrosion` / 虚空侵蚀。
  - 商店服务改为 `cleanse_equipment` / 装备净化。
  - 语义为净化受到虚空侵蚀的装备，使其恢复生效，不是修理或修复耐久。

## Current Repository State

- 当前仓库仍是文档和内容表项目，尚未创建正式源码工程。
- 第 1 章事件、节点提示、商店、存档、节点后随机事件和装备净化相关文档/JSON 已更新。
- 已新增内容表：
  - `docs/03-content/data/events.json`
  - `docs/03-content/data/node-hints.json`
- 已新增设计文档：
  - `docs/03-content/events-chapter-1.md`
  - `docs/03-content/node-hints-chapter-1.md`

## Next Discussion

- 用户认为现在应该开始考虑美术资产，包括角色和敌人立绘、UI、装备图标、药水图标等。
- 已确认可以先做 MVP 美术资产清单和占位资产规范。
- UI 方案尚未定论。不要把上次助手提出的 UI 信息架构、组件系统或战斗 UI 原型顺序当成已确认方案；用户明确表示“有待商榷”，并决定下次再讨论。
- 下次应优先讨论：
  - MVP 美术资产清单的范围和优先级。
  - UI 应该如何推进：线框、组件、最终视觉、或其他流程。
  - 哪些资产必须先定，哪些可以用占位资源。
