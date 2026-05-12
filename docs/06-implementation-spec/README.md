# 实现规格说明

本目录用于承载《Ashlight Echoes》重构开发时直接面向 Codex / Claude Code 的实现规格。

完整文档体系见 [../README.md](../README.md)。本目录不是完整设计入口；它负责把 `docs/02-systems/` 和 `docs/03-content/` 中已经确认的设计翻译成更适合开发执行的数据契约、状态机规则、内容边界和验收标准。

## 当前目标

第一版可玩版本覆盖：

- 序章教程
- 第 1 章：银誓余烬 / 银誓要塞遗址
- 从新游戏开始，到击败第 1 章 Boss 并完成章节结算的完整流程

具体范围见 [mvp-scope.md](mvp-scope.md)。

## 阅读顺序

开发时建议按以下顺序阅读：

1. [mvp-scope.md](mvp-scope.md)：第一版实现边界和验收目标。
2. [data-schema.md](data-schema.md)：角色、卡牌、效果、敌人、装备、药水、节点、奖励、存档等运行时数据结构。
3. [battle-state-machine.md](battle-state-machine.md)：战斗阶段、回合流程、牌区、目标选择、伤害结算、状态时机、胜负判定。
4. [effect-dsl.md](effect-dsl.md)：卡牌、装备、药水、节点修正等效果的表达格式和执行规则。
5. [custom-effect-handlers.md](custom-effect-handlers.md)：`customId` 特殊效果的 handler 注册、触发窗口和全量语义清单。
6. [run-flow.md](run-flow.md)：序章、第 1 章节点生成、奖励、商店、事件、存档和章节结算。
7. `acceptance-cases.md`：MVP 的自动化或手动验收用例，当前尚未建立。

当前已建立 [mvp-scope.md](mvp-scope.md)、[data-schema.md](data-schema.md)、[battle-state-machine.md](battle-state-machine.md)、[effect-dsl.md](effect-dsl.md)、[custom-effect-handlers.md](custom-effect-handlers.md) 和 [run-flow.md](run-flow.md)。其中 [run-flow.md](run-flow.md) 是初版规格，仍需继续按 [../02-systems/map-encounters.md](../02-systems/map-encounters.md) 与 [../02-systems/node-modifiers.md](../02-systems/node-modifiers.md) 校准。

## 与设计源头的关系

`06-implementation-spec/` 的定位是实现契约，不是玩法设计替代品。

- 系统设计意图以 `docs/02-systems/` 为源头。
- 角色、卡牌、敌人等内容定义以 `docs/03-content/` 和 `docs/03-content/data/` 为源头。
- 字段结构、运行时状态、结算顺序和可执行流程以本目录为实现规格。
- 如果实现规格与系统设计冲突，先视为文档 bug，必须修正文档，不允许在实现层自行猜测。
- 如果机器内容表与规则文档冲突，数值 / `id`（唯一标识） / 名称先检查内容表，规则 / 时机 / 流程先检查系统设计和实现规格。
- 不允许为了兼容旧写法新增过渡字段、兼容表或互相矛盾的说明；应直接迁移、删除或替换旧内容。

## 实现原则

MVP 必须是一个真实可玩的切片，不是菜单演示版。

玩家必须能够从新游戏开始，在序章学会基础操作，进入第 1 章进行路线选择和战斗成长，最终击败或败给第 1 章 Boss，并在无需开发者介入的情况下到达清晰的胜利或失败结果界面。
