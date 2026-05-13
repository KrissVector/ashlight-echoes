# Ashlight Echoes 文档索引

> 目的：让新人和 LLM 快速理解项目文档结构、每个文档的职责、有效性和阅读顺序。

## 项目文档原则

本项目文档按“设计源头 -> 内容定义 -> 实现规格 -> 机器内容表”组织。

实现者不能只读某一个文件就开始写代码。正确做法是先确认对应系统的设计源头，再阅读实现规格和机器内容表。发现冲突时，应修正文档或内容表，不允许在实现层临时猜测。

`docs/02-systems/map-encounters.md` 是当前有效的地图模块设计，不是旧模块。`docs/06-implementation-spec/run-flow.md` 的职责是把它翻译成可实现流程，不能替换它的设计。

## 推荐阅读顺序

### 了解项目

1. [00-overview.md](00-overview.md)：项目定位、核心循环、章节结构、全局设计支柱。
2. [01-world/world-setting.md](01-world/world-setting.md)：世界观、章节叙事、角色和敌人背景。
3. [03-content/characters.md](03-content/characters.md)：可玩角色定位、属性和初始卡牌。

### 开发 MVP

1. [06-implementation-spec/mvp-scope.md](06-implementation-spec/mvp-scope.md)：MVP 范围、必须实现内容和验收目标。
2. [06-implementation-spec/data-schema.md](06-implementation-spec/data-schema.md)：运行时数据结构。
3. [06-implementation-spec/battle-state-machine.md](06-implementation-spec/battle-state-machine.md)：战斗阶段、回合、牌区、伤害、治疗、状态和触发器。
4. [06-implementation-spec/effect-dsl.md](06-implementation-spec/effect-dsl.md)：卡牌、装备、药水、节点修正共用的效果 DSL（效果领域专用表达格式）。
5. [02-systems/map-encounters.md](02-systems/map-encounters.md)：当前地图与节点选择设计源头。
6. [02-systems/node-modifiers.md](02-systems/node-modifiers.md)：战后随机事件与节点选项自带修正的设计源头。
7. [02-systems/roguelike-run.md](02-systems/roguelike-run.md)：run（单局流程）、奖励、成长池、金币、商店和章节推进设计源头。
8. [06-implementation-spec/run-flow.md](06-implementation-spec/run-flow.md)：新 run（单局流程）、序章、第 1 章节点生成、奖励、商店、事件、存档和通关流程。
9. [03-content/data/README.md](03-content/data/README.md)：正式机器内容表入口。

## 文档模块

### 根目录

| 文档 | 有效性 | 作用 |
|------|--------|------|
| [00-overview.md](00-overview.md) | 当前有效 | 项目级总览。定义游戏类型、核心循环、章节结构和项目迁移决策。 |
| [README.md](README.md) | 当前有效 | 文档索引。说明文档体系、阅读顺序、冲突处理和维护规则。 |
| [../memory/README.md](../memory/README.md) | 当前有效 | LLM 协作记忆入口。说明换电脑、pull 后和提交前的记忆同步规则。 |

### `01-world/` 世界观与叙事

| 文档 | 有效性 | 作用 |
|------|--------|------|
| [world-setting.md](01-world/world-setting.md) | 当前有效 | 世界观、虚空灾变、章节主地点、角色叙事背景和第 1 章叙事基线。 |

世界观文档负责“为什么”和“讲什么”，不负责运行时规则。若战斗、奖励、节点生成与世界观文案冲突，先保留规则正确性，再回头调整叙事表达。

### `02-systems/` 系统设计源头

这些文档是玩法设计源头。实现规格必须尊重这里的设计，不应擅自替换。

| 文档 | 有效性 | 作用 |
|------|--------|------|
| [battle.md](02-systems/battle.md) | 当前有效 | 战斗规则设计源头：能量、AS、手牌、命中、暴击、DEF、护盾、毒、流血、虚弱、脆弱等。 |
| [cards.md](02-systems/cards.md) | 当前有效 | 卡牌系统设计源头：固有牌、大招、共享牌库、牌区、费用、升级、消耗、变形。 |
| [map-encounters.md](02-systems/map-encounters.md) | 当前有效 | 地图模块设计源头：每步二/三选一、随机候选节点、固定步、战后随机事件、节点选项自带修正、事件节点、商店、Boss。 |
| [node-modifiers.md](02-systems/node-modifiers.md) | 当前有效 | 节点修正设计源头：22 个战后随机事件、25 个节点自带修正，以及两者的触发时机和控制方式。 |
| [roguelike-run.md](02-systems/roguelike-run.md) | 当前有效 | 肉鸽 run（单局流程）、奖励、成长池、金币经济、商店和整体章节推进设计源头。 |
| [equipment.md](02-systems/equipment.md) | 当前有效 | 装备系统、稀有度、装备列表、掉落、商店回收、诅咒装备。 |
| [potions.md](02-systems/potions.md) | 当前有效 | 药水系统、背包、使用时机、药水列表、商店和战斗掉落入口。 |
| [team-characters.md](02-systems/team-characters.md) | 当前有效 | 队伍与角色术语说明，上场人数和角色成长来源。 |

### `03-content/` 内容设计

| 文档 | 有效性 | 作用 |
|------|--------|------|
| [characters.md](03-content/characters.md) | 当前有效 | 角色定位、属性、初始牌和解锁章节说明。数值以 `data/characters.json` 为准。 |
| [enemies-chapter-1.md](03-content/enemies-chapter-1.md) | 当前有效 | 序章与第 1 章普通敌人、精英敌人、步 6 追杀剧情节点和 Boss 的数值、AI 与叙事方向。 |
| [build-archetypes.md](03-content/build-archetypes.md) | 当前有效 | 构筑原型和系统联动方向，用于指导内容扩展和平衡。 |
| [data/](03-content/data/README.md) | 当前有效 | 正式机器内容表。当前包含角色、卡牌、章节、装备、药水、节点修正和奖励表。 |

`03-content/data/` 中的 JSON 是机器可读内容表，不是草案。`id`（唯一标识）、名称、初始牌、章节主地点、卡牌效果等实现时应读取这里的数据。

### `04-art/` 美术与视觉

| 文档 / 资源 | 有效性 | 作用 |
|-------------|--------|------|
| [style-guide.md](04-art/style-guide.md) | 当前有效 | 美术风格、UI 基调、角色和场景视觉方向。 |
| [concept-index.md](04-art/concept-index.md) | 当前有效 | 概念图索引。 |
| `concept/` 与图片资源 | 参考资源 | 视觉参考，不定义玩法规则。 |

### `../memory/` LLM 协作记忆

| 文档 | 有效性 | 作用 |
|------|--------|------|
| [README.md](../memory/README.md) | 当前有效 | 说明同步到项目内的 LLM 记忆文件用途和同步规则。 |
| [user-communication-preferences.md](../memory/user-communication-preferences.md) | 当前有效 | 用户沟通偏好、字段解释要求、文档质量要求、记忆同步规则和项目交付标准。 |

这些文件用于换电脑、换 LLM 工具或 pull 之后恢复协作上下文，不是玩法设计源头。每次提交前必须把当前本机 LLM 记忆同步到 `memory/`；每次 pull 之后必须先把 `memory/` 同步回本机 LLM 记忆目录。项目规则仍以 `02-systems/`、`03-content/`、`06-implementation-spec/` 中的正式文档为准。

### `06-implementation-spec/` 实现规格

这些文档面向 Codex / Claude Code 直接开发。它们把系统设计翻译成可实现的数据契约、状态机和流程。

| 文档 | 有效性 | 作用 |
|------|--------|------|
| [README.md](06-implementation-spec/README.md) | 当前有效 | 实现规格目录说明和 MVP 开发阅读顺序。 |
| [mvp-scope.md](06-implementation-spec/mvp-scope.md) | 当前有效 | 第 1 版可玩范围、必须实现内容和验收标准。 |
| [data-schema.md](06-implementation-spec/data-schema.md) | 当前有效 | 运行时数据结构：角色、卡牌、效果、敌人、节点、奖励、商店、事件、存档。 |
| [effect-dsl.md](06-implementation-spec/effect-dsl.md) | 当前有效 | 效果 DSL（效果领域专用表达格式）：公式、目标、伤害、治疗、状态、触发器、卡牌操作、召唤。 |
| [battle-state-machine.md](06-implementation-spec/battle-state-machine.md) | 当前有效 | 战斗状态机：初始化、抢攻、回合、出牌、普攻、敌方行动、伤害和治疗结算。 |
| [run-flow.md](06-implementation-spec/run-flow.md) | 初版规格，需要继续校准 | Run（单局流程）流程：新游戏、序章、第 1 章随机候选节点、固定步、奖励、商店、事件、存档。必须以 `map-encounters.md` 为设计源头。 |

## 三套“事件”不要混用

地图模块里“事件”有三种含义：

| 名称 | 来源文档 | 时机 | 玩家控制 | 说明 |
|------|----------|------|----------|------|
| 战后随机事件 | `node-modifiers.md` | 节点完成后概率触发 | 不可控 | 22 个增益/减益（buff/debuff），属于节点修正来源之一；默认只对我方生效，明确写敌我双方的条目作为战场规则对双方生效。 |
| 节点选项自带修正 | `node-modifiers.md` | 候选节点生成时展示，进入节点前生效 | 可控 | 25 个节点修正，玩家选择节点时能看到并主动承担代价/收益。 |
| 事件节点 | `map-encounters.md` | 玩家选择 `event`（事件节点类型）节点后进入 | 可控 | 非战斗文字交互节点，做选择后获得奖励、惩罚或中性结果。 |

后续写 `run-flow.md`、事件表或节点修正表时，必须保持这三套机制分离。

## 冲突处理

文档冲突不能靠实现层猜。

处理规则：

- 系统设计意图以 `02-systems/` 为源头。
- 机器可读内容以 `03-content/data/` 为源头。
- 实现流程、字段和结算顺序以 `06-implementation-spec/` 为源头。
- 如果实现规格和系统设计冲突，先视为文档 bug，修正其中一个文档并说明原因。
- 如果 JSON 内容表和规则文档冲突，数值 / `id`（唯一标识） / 名称优先检查 JSON，规则 / 时机 / 流程优先检查系统设计和实现规格。
- 不允许新增“兼容旧写法”段落来掩盖冲突；应直接迁移、删除或替换旧内容。

## 修改文档时先改哪里

| 变更类型 | 先改 | 再改 |
|----------|------|------|
| 地图、节点、随机候选、固定步 | `02-systems/map-encounters.md` | `06-implementation-spec/run-flow.md`、相关数据表 |
| 战后随机事件或节点修正 | `02-systems/node-modifiers.md` | `run-flow.md`、机器可读节点修正表 |
| 战斗结算规则 | `02-systems/battle.md` | `battle-state-machine.md`、`effect-dsl.md` |
| 卡牌效果表达 | `effect-dsl.md` | `data-schema.md`、`03-content/data/cards.json` |
| 角色 / 卡牌 / 章节内容 | `03-content/*.md` 或 `03-content/data/*.json` | `data-schema.md` 如字段变化 |
| 装备 / 药水内容 | `02-systems/equipment.md` / `potions.md` | `03-content/data/equipment.json`、`03-content/data/potions.json` |
| MVP 范围 | `mvp-scope.md` | README、MVP 验收标准 |

## 当前主要缺口

这些不是无效文档，而是尚未完全机器化或尚未定死的内容：

- 第 1 章事件节点正式列表、文案库和事件结果表。
- 战后随机事件触发概率和抽取权重。
- 第 1 章普通战、精英战、特殊精英、Boss 的机器 encounter 表。
- 商店、事件的机器内容表。

补这些内容时，必须先回到对应设计源头确认规则，再写实现规格和数据表。

MVP 暂不单独维护 `acceptance-cases.md`。形成可玩版本后，以项目负责人实际游玩体验为主要验收方式；验收标准见 `mvp-scope.md`。

## 给 LLM 的工作规则

- 不要把 `map-encounters.md` 当作旧设计；它是当前地图模块设计源头。
- 不要只根据字段名猜语义；先读对应系统文档和实现规格。
- 不要新增一次性字段来绕过通用 DSL；优先使用 `EffectDef`（单个效果定义）、`ValueFormula`（数值公式）、`condition`（生效条件）、`conditionMultiplier`（满足条件时的倍率修正）等通用结构。
- 不要留下半迁移内容、兼容旧字段表或互相矛盾的版本。
- 发现规则未定时，直接把缺口说清楚并向项目负责人确认，不要自行补设定。
