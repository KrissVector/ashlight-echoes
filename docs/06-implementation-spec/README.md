# 实现规格说明

本目录用于承载《Ashlight Echoes》重构开发时直接面向 Codex / Claude Code 的实现规格。

`docs/01-world`、`docs/02-systems`、`docs/03-content`、`docs/04-art` 仍然是设计基线；本目录负责把这些设计翻译成更适合开发执行的数据契约、状态机规则、内容边界和验收标准。

## 当前目标

第一版可玩版本覆盖：

- 序章教程
- 第 1 章：银誓余烬 / 银誓要塞遗址
- 从新游戏开始，到击败第 1 章 Boss 并完成章节结算的完整流程

具体范围见 [mvp-scope.md](mvp-scope.md)。

## 阅读顺序

开发时建议按以下顺序阅读：

1. [mvp-scope.md](mvp-scope.md)：第一版实现边界和验收目标。
2. `data-schema.md`：角色、卡牌、效果、敌人、装备、药水、节点、奖励、存档等运行时数据结构。
3. `battle-state-machine.md`：战斗阶段、回合流程、牌区、目标选择、伤害结算、状态时机、胜负判定。
4. `effect-dsl.md`：卡牌、装备、药水、节点修正等效果的表达格式和执行规则。
5. `run-flow.md`：序章、第 1 章节点生成、奖励、商店、事件、存档和章节结算。
6. `acceptance-cases.md`：MVP 的自动化或手动验收用例。

当前仅先建立 [mvp-scope.md](mvp-scope.md)。其余文件应在正式开发前补齐。

## 资料优先级

如果文档之间出现冲突，按以下顺序判定：

1. `docs/06-implementation-spec/` 中的实现规格
2. `docs/02-systems/` 中的系统规则
3. `docs/03-content/` 中的内容定义
4. `docs/01-world/` 和 `docs/04-art/` 中的世界观与美术方向
5. `docs/05-data-drafts/` 中的迁移数据草案

`docs/05-data-drafts/` 只能作为参考材料。不要把其中的 JSON 直接当成最终 runtime schema；正式结构需要以后由 `data-schema.md` 定义。

## 实现原则

MVP 必须是一个真实可玩的切片，不是菜单 demo。

玩家必须能够从新游戏开始，在序章学会基础操作，进入第 1 章进行路线选择和战斗成长，最终击败或败给第 1 章 Boss，并在无需开发者介入的情况下到达清晰的胜利或失败结果界面。
