# 效果 DSL 规格

> 状态：初版规格
> 适用范围：MVP 内容与后续正式实现的效果表达和执行约定
> 相关文档：`data-schema.md`、`battle-state-machine.md`、`mvp-scope.md`

## 目的

本文定义《Ashlight Echoes》运行时使用的效果 DSL。这里的 DSL 不是给玩家展示的文本，而是卡牌、装备、药水、敌人技能、节点修正、事件结果和奖励共同使用的数据表达格式。

目标是让通用内容通过 JSON 数据驱动，避免为普通伤害、治疗、护盾、状态、抽牌、能量、卡牌操作等重复规则单独硬编码逻辑。

传说装备、诅咒装备、节点修正和少数药水存在规则级特殊机制。它们不强行扩展 DSL，而是使用 `custom` 效果和 `customId` handler registry（处理函数注册表）。所有 `customId` 的语义和触发窗口见 [custom-effect-handlers.md](custom-effect-handlers.md)。

本文只整理并固化现有设计中已经出现的规则。涉及新增机制或现有设计未定死的地方，必须先讨论确认，再写入正式 DSL。

## 资料来源

本文基于以下现有设计：

- `docs/02-systems/battle.md`：伤害、治疗、护盾、毒、流血、虚弱、脆弱、AS、能量等战斗规则。
- `docs/02-systems/cards.md`：卡牌来源、牌区、升级、消耗和变形规则。
- `docs/02-systems/equipment.md`：装备属性加成、附带卡牌、改变卡牌、诅咒牌。
- `docs/02-systems/potions.md`：药水永久成长、卡牌复制、删除、改造。
- `docs/02-systems/node-modifiers.md`：节点修正和环境效果。
- `docs/03-content/data/cards.json`：当前正式卡牌内容表。
- `docs/06-implementation-spec/data-schema.md`：运行时数据结构基线。

## 核心结构

正式效果结构沿用 `data-schema.md` 中的 `EffectDef`：

```ts
interface EffectDef {
  type: EffectType;
  target?: TargetType;
  damageType?: DamageType;
  source?: string;
  stat?: StatId | GlobalStatId;
  formula?: ValueFormula;
  value?: number;
  percent?: number;
  multiplier?: number;
  hits?: number;
  randomTarget?: boolean;
  primaryApScaling?: number;
  secondaryApScaling?: number;
  energyPenalty?: number;
  oncePerBattle?: boolean;
  ignoreDefense?: boolean;
  ignoreShield?: boolean;
  enemyId?: EnemyId;
  count?: number;
  chance?: number;
  duration?: Duration;
  condition?: ConditionDef;
  conditionMultiplier?: ConditionMultiplierDef;
  debuffs?: string[];
  effects?: EffectDef[];
  transforms?: CardTransformDef[];
  cardOperation?: CardOperationDef;
  customId?: string;
  tags?: string[];
}

interface CardTransformDef {
  from: CardId[];
  to: CardId;
}

interface ConditionMultiplierDef {
  condition: ConditionDef;
  multiplier: number;
}
```

通用规则：

- `type` 必须存在，并且必须能在效果注册表中找到执行器。
- `target` 缺省时继承外层动作目标；卡牌通常继承 `CardDef.targetType`。
- `effects` 用于嵌套触发，如击杀触发、条件触发或阶段进入触发。
- `customId` 只用于当前 schema 无法表达但设计已经存在的特殊效果；使用前必须在 [custom-effect-handlers.md](custom-effect-handlers.md) 中记录语义和触发窗口。
- 执行器不得根据 `description` 文本推断逻辑；描述只用于 UI。

## 公式规范

正式公式使用 `formula` 字段：

```ts
interface ValueFormula {
  base?: number;
  atk?: number;
  ap?: number;
  hp?: number;
  maxHp?: number;
  def?: number;
  shield?: number;
  poisonStacks?: number;
  bleedStacks?: number;
  vulnerableStacks?: number;
  weaknessStacks?: number;
  debuffTypeCount?: number;
  chapter?: number;
  consumedCards?: number;
  currentEnergy?: number;
  products?: FormulaProductTerm[];
}

interface FormulaProductTerm {
  factors: FormulaFactor[];
  coefficient: number;
}

type FormulaFactor =
  | "atk"
  | "ap"
  | "hp"
  | "maxHp"
  | "def"
  | "shield"
  | "poisonStacks"
  | "bleedStacks"
  | "vulnerableStacks"
  | "weaknessStacks"
  | "debuffTypeCount"
  | "chapter"
  | "consumedCards"
  | "currentEnergy";
}
```

公式计算：

```text
value =
  base
  + source.ATK * atk
  + source.AP * ap
  + source.currentHp * hp
  + source.maxHp * maxHp
  + source.DEF * def
  + source.shield * shield
  + target.poisonStacks * poisonStacks
  + target.bleedStacks * bleedStacks
  + target.vulnerableStacks * vulnerableStacks
  + target.weaknessStacks * weaknessStacks
  + target.debuffTypeCount * debuffTypeCount
  + chapterIndex * chapter
  + battle.consumedCards * consumedCards
  + battle.currentEnergy * currentEnergy
  + sum(product(factors) * coefficient for each products item)
```

乘积项说明：

- `products` 用于表达多个变量相乘的公式项。
- 每个乘积项先读取 `factors` 中的变量值并相乘，再乘以 `coefficient`。
- 例如“每种异常状态增加 25% AP 伤害”写作：`products: [{ factors: ["ap", "debuffTypeCount"], coefficient: 0.25 }]`。
- 这里的 `debuffTypeCount` 表示目标身上的异常状态种类数，不是异常状态层数总和。

取整规则：

- 所有最终伤害、治疗、护盾、层数、属性修正结果均四舍五入取整。
- 百分数字段用百分数存储，`percent: 30` 表示 30%，不是 0.3。
- 层数效果也使用公式计算层数。

卡牌、装备、药水、敌人技能和节点修正都必须使用 `formula` 表达可计算数值。不得在新内容表里使用散落在效果根节点上的临时公式字段。

## 目标解析

`target` 使用 `data-schema.md` 中的 `TargetType`：

```ts
type TargetType =
  | "self"
  | "selected"
  | "single_ally"
  | "all_allies"
  | "single_enemy"
  | "all_enemies"
  | "random_ally"
  | "random_enemy"
  | "all_units";
```

规则：

- `self`：效果来源单位。
- `selected`：玩家或 AI 为当前动作选择的主要目标。
- `single_ally` / `single_enemy`：需要显式选择一个合法目标。
- `all_allies` / `all_enemies` / `all_units`：解析为多个目标，按稳定顺序逐个执行效果。
- `random_ally` / `random_enemy`：从合法存活目标中随机选取。
- 卡牌层的 `targetType` 决定可选目标；效果层的 `target` 可以覆盖卡牌目标。
- 从敌人视角执行时，`ally` 指敌方阵营，`enemy` 指我方阵营。

## 执行上下文

效果至少运行在以下上下文中：

| 上下文 | 来源 | 可用状态 |
|--------|------|----------|
| `battle` | 卡牌、普攻、敌人技能、战斗内状态 | `BattleState`、来源单位、目标单位、牌区 |
| `between_nodes` | 药水、节点间事件、商店服务 | `RunSave`、角色 run 状态、药水背包、装备槽 |
| `reward` | 战后奖励、Boss 奖励 | `RunSave`、奖励池、待领奖励 |
| `run_setup` | 新 run、序章、章节进入 | `ProfileSave`、`RunSave` |

效果执行器必须校验上下文。例如：

- `damage` 只能在战斗上下文执行。
- `potion` 的永久属性和卡牌操作只能在节点之间执行。
- `card_operation` 可以由药水、装备、事件或奖励触发，但商店不能直接提供删牌或买牌服务。

## 效果执行顺序

同一 `effects` 数组按声明顺序执行。

一次卡牌或技能的高层流程为：

```text
校验费用 / 使用条件
  -> 解析主要目标
  -> 逐条执行 effects
  -> 处理触发的嵌套 effects
  -> 根据卡牌 isExhaust 将卡牌放入 Discard 或 Exhaust
  -> 检查胜负
```

具体战斗阶段、状态触发窗口和胜负检查位置由 `battle-state-machine.md` 定义。

## 战斗结算效果

### `damage`

造成伤害。

必填：

- `damageType`: `physical`、`magical` 或 `fixed`
- `formula`

规则来自 `battle.md`：

1. 命中判定：`HitRate = clamp(source.ACC - target.EVD, 10, 100)`。
2. 基础伤害由 `formula` 计算。
3. 非固定伤害走 DEF 减法模型：`max(1, damage - target.DEF)`。
4. 命中后进行暴击判定；普通攻击、卡牌伤害和治疗均可暴击。
5. 扣除防御并结算暴击后，进入增伤 / 减伤窗口；`conditional_bonus`、脆弱、虚弱等在此结算。
6. 普通伤害先扣护盾，再扣 HP。
7. 最终伤害最低为 1。
8. 毒、流血及其引爆伤害无视 DEF 与护盾，直接扣 HP。
9. 如果同一攻击动作后续附加流血，流血是否生效取决于伤害结算后的目标护盾状态，见 `bleed`。

字段：

- `hits`：多段伤害次数。未填写时按 1 段处理。
- `randomTarget: true`：每段从 `target` 解析出的合法目标中重新随机选择。
- `ignoreDefense: true`：跳过 DEF 减法。
- `ignoreShield: true`：跳过护盾，直接扣 HP。该字段只用于明确绕过护盾的效果，例如读取毒层造成的直接 HP 伤害。

### `heal`

治疗目标。

必填：

- `formula`

规则：

- 治疗量由 `formula` 计算，并按战斗规则取整。
- 治疗可以暴击。
- 治疗不受 DEF 影响。
- 治疗不能恢复护盾。
- 治疗允许超过目标最大 HP，直接保留为当前 HP 超过最大 HP 的状态。

### `shield`

为目标施加护盾。

必填：

- `formula`

规则：

- 护盾可叠加。
- 护盾上限为目标最大 HP 的 200%。
- 护盾不会在回合结束时自然消失，除非效果自身有 `duration` 且状态机定义了过期处理。
- 受到普通伤害时先扣护盾，再扣 HP。
- 毒、流血及其引爆伤害无视护盾，直接扣 HP。

### `shield_amplify`

按百分比增加目标当前护盾。

必填：

- `formula`，计算出的值视为百分数。

示例：

```json
{
  "type": "shield_amplify",
  "target": "self",
  "formula": { "base": 10, "ap": 2.0 }
}
```

表示当前护盾增加 `10 + AP * 2.0`%。

## 状态与层数效果

### `poison`

施加毒层。

规则：

- 毒以层数叠加。
- 目标回合结束后结算。
- 结算时造成等于当前毒层数的伤害，然后层数 -1。
- 毒伤害无视 DEF 与护盾，直接扣 HP。

### `bleed`

施加流血层。

规则：

- 流血以层数叠加。
- 攻击附加流血时，先结算该攻击的伤害和护盾扣减；只有目标在伤害结算后 `shield <= 0` 时才施加流血。
- 若攻击未命中，或伤害结算后目标仍有护盾，则该次攻击附加的流血跳过，并写入战斗日志。
- 若当前 `bleed` 效果没有绑定到攻击伤害结果，例如事件、节点修正、药水或纯状态牌直接施加流血，则不检查护盾。
- 目标回合开始前结算。
- 结算时造成等于当前流血层数的伤害，然后层数 -1。
- 流血伤害无视 DEF 与护盾，直接扣 HP。

### `weakness`

施加虚弱层。

规则：

- 虚弱以层数叠加。
- 目标造成的伤害在扣除防御并结算暴击后，减少虚弱层数，最低为 1。
- 每回合结束时层数 -1。

### `vulnerable`

施加脆弱层。

规则：

- 脆弱以层数叠加。
- 目标受到的伤害在扣除防御并结算暴击后，增加脆弱层数。
- 每回合结束时层数 -1。

### `poison_detonate`

读取毒层造成伤害，不清空毒层。

规则：

- 按 `battle.md`，每层毒造成 3 点伤害。
- 总伤害 = `当前毒层数 * 3`。
- 不清空毒层。
- 无视 DEF 与护盾，直接扣 HP。

若内容需要“造成等同毒层数的伤害，不消耗毒层”，使用普通 `damage` 表达：

```json
{
  "type": "damage",
  "target": "all_enemies",
  "damageType": "fixed",
  "formula": { "poisonStacks": 1 },
  "ignoreDefense": true,
  "ignoreShield": true
}
```

这类效果是毒层读取伤害，不是 `poison_detonate`，不会自动套用每层 3 点的引爆倍率。

### `bleed_detonate`

消耗目标全部流血层并造成伤害。

规则：

- 总伤害 = `流血层数 * 流血层数`。
- 结算后流血层归零。
- 无视 DEF 与护盾，直接扣 HP。

### `taunt`

施加嘲讽。

字段：

- `target`：获得嘲讽状态的单位。
- `duration`：持续回合数。

规则：

- 拥有嘲讽的我方单位会强制敌方单体攻击优先选择自己。
- 多个嘲讽目标同时存在时，敌方 AI 在合法嘲讽目标中按存在感权重选择，除非技能另有明确目标规则。
- 嘲讽只约束敌方主动选择目标，不改变全体攻击、随机攻击或已指定不可改目标的效果。

### `presence_gain`

增加目标存在感权重。

字段：

- `target`
- `value`
- `duration`

规则：

- 存在感只用于敌方单体目标选择，不影响伤害、治疗、护盾或其它属性公式。
- `duration` 为数字时，按回合数持续；未写持续时间的数据无效。
- 多个存在感增益可以叠加。
- 当前用于会临时改变受击倾向的卡牌；具体数值以 `docs/03-content/data/cards.json` 为准。

### `mark`

给目标施加标记状态。

字段：

- `target`
- `duration`

规则：

- 标记本身不造成伤害。
- 标记会记录来源单位 `sourceUnitId`，方便敌人技能优先选择由自己施加或自己关注的标记目标。
- 多个来源的标记可以并存；同一来源再次施加标记时刷新该来源标记的持续时间。
- 标记持续时间结束后移除。
- 当前用于癫狂的强盗首领「猎杀标记」。

## 行动、牌区和资源效果

### `draw`

抽牌。

字段：

- `value`: 抽牌数量。

规则：

- 从 `drawPile` 抽到 `hand`。
- 手牌上限为 8。
- `drawPile` 为空时，将 `discardPile` 洗回 `drawPile`。
- 超过手牌上限的新抽牌按战斗系统规则自动弃置。

### `energy_gain`

获得公共能量。

字段：

- `value`: 获得能量数量。

规则：

- 增加当前战斗公共能量。
- 不超过当前 `maxEnergy`。

### `as_gain`

获得当前回合可用 AS。

字段：

- `value`: 增加数量。

规则：

- 立即增加目标当前回合 `remainingAs`。
- 不改变目标 AS 上限，除非效果同时包含 `as_max_increase`。

### `as_reset`

将目标当前 AS 恢复至本回合上限。

规则：

- 如果未指定 `target`，默认作用于来源单位。
- 只影响当前战斗状态，不改变 run 内角色基础 AS。

### `as_max_increase`

提升 AS 上限。

字段：

- `value`
- `duration`

规则：

- 若 `duration: "battle"`，持续到本场战斗结束。
- 若由药水等永久效果提供，应写入 run 状态而不是战斗临时状态。
- `oncePerBattle: true` 可用于限制同一来源在同一场战斗中只应用一次该增益。

### `return_to_hand`

让当前使用的卡牌返回手牌。

规则：

- 只对当前正在结算的卡牌有效。
- 允许覆盖当前卡牌的 `isExhaust`。如果一张消耗型卡牌在结算中触发 `return_to_hand`，则本次结算结束后返回手牌，不进入消耗堆。

### `overload`

超载惩罚。下回合或指定持续时间内降低能量上限。

示例：

```json
{
  "type": "overload",
  "energyPenalty": 4,
  "duration": 1
}
```

字段：

- `energyPenalty`：能量上限降低值。
- `duration`：持续回合数。

规则：

- `energyPenalty: 4` 表示能量上限 -4。
- `星陨·超载` 使用该效果表达下回合能量上限 -4。

## 属性修正效果

### `buff`

增加固定属性值。

字段：

- `target`
- `stat`
- `formula`
- `duration`

规则：

- `duration` 为数字时，按回合数持续。
- `duration: "battle"` 时持续到本场战斗结束。
- 若是药水或成长奖励导致的永久 run 成长，不应使用战斗临时 buff，而应写入 `RunCharacterState.stats`。
- `conditionMultiplier` 表示条件倍率。条件满足时，本条效果的数值乘以指定倍率。
- 当前用于 `自然祝福·共鸣`：目标是施法者自己时，两条 buff 各自乘以 2。

### `percent_buff`

按百分比增加属性。

字段：

- `target`
- `stat`
- `formula`
- `duration`

规则：

- `formula` 结果视为百分数。
- 示例：`formula: { "base": 40 }` 表示目标指定属性 +40%。
- `formula` 可以为负数，用于百分比属性 debuff。例如命中率 -50% 可写作对 `acc` 施加 `formula: { "base": -50 }`、持续 1 回合；闪避率 -50% 同理对 `evd` 生效。

### `permanent_buff`

本场战斗内永久增益。

当前卡牌中用于灼夜大招：

```json
{
  "type": "permanent_buff",
  "target": "self",
  "stat": "crit",
  "value": 50,
  "duration": "battle",
  "oncePerBattle": true
}
```

规则：

- `duration: "battle"` 表示只持续本场战斗，不写回 run。
- `oncePerBattle: true` 表示同一来源在同一场战斗中只生效一次。

### `stat_multiply`

属性乘法修正。

字段：

- `target`
- `stat`
- `multiplier`
- `duration`

规则：

- 例如 `multiplier: 2` 表示属性翻倍。
- 乘法修正与固定值修正、百分比修正的叠加顺序由 `battle-state-machine.md` 定义。

### `set_stat`

临时设置属性为指定值。

字段：

- `target`
- `stat`
- `value`
- `duration`

当前用于“闪避变为 0”。

## 触发与条件效果

### `guaranteed_crit`

让后续攻击必定暴击。

字段：

- `count`: 生效次数。

规则：

- 当前设计语义是“下次攻击必定暴击”。
- 命中判定仍然正常执行；只有命中后暴击判定被强制成功。

### `lifesteal`

按伤害恢复 HP。

字段：

- `percent`

规则：

- 读取同一动作中造成的最终 HP 伤害。
- 恢复量 = `最终 HP 伤害 * percent / 100`。
- 只按实际扣到 HP 的伤害计算，不包含被护盾吸收的伤害。

### `conditional_bonus`

条件加伤。它绑定当前正在结算的整张卡，而不是只绑定最近一个 `damage` 效果。

字段：

- `condition`：条件对象，用来判断当前受击目标或来源是否满足条件。
- `percent`：增伤百分比，`percent: 100` 表示伤害 +100%。

示例：

```json
{
  "type": "conditional_bonus",
  "condition": {
    "type": "hp_below",
    "target": "selected",
    "threshold": 0.5
  },
  "percent": 100
}
```

已登记条件：

- `hp_below`：目标当前 HP 比例低于 `threshold`。
- `target_has_status`：目标拥有指定状态，状态 id 写在 `statusId`。
- `stat_equals`：目标指定属性等于 `value`。

规则：

- `conditional_bonus` 在扣除防御并结算暴击之后、实际扣除护盾或 HP 之前结算。
- 同一张卡内的全部 `damage` 效果都读取该卡上的 `conditional_bonus`。
- 多个 `conditional_bonus` 可以叠加，叠加方式为加算百分比。
- 对多目标或多段伤害，条件按每个实际受击目标分别判断。
- 例：`嗜血之刃·猎杀` 同时满足目标 HP < 50% 和目标有流血时，两个 `percent: 100` 同时生效，总增伤为 +200%。
- 例：`虚空裂隙·穿透` 使用 `stat_equals` 判断目标 DEF 是否为 0；若为 0，则伤害 +15%。

### `on_kill`

当前动作击杀主要目标时触发嵌套效果。

字段：

- `effects`

规则：

- 用于单目标击杀触发。
- 嵌套效果按声明顺序执行。

### `on_kill_per_target`

当前动作每击杀一个目标触发一次嵌套效果。

字段：

- `effects`

规则：

- 用于全体伤害或多目标伤害。
- 每个被击杀目标触发一次。

### `damage_reflect`

受到攻击时反弹伤害。

字段：

- `source`：反弹数值来源；`shield` 表示读取被攻击前的护盾值。
- `duration`：反弹状态持续回合数。

示例：

```json
{
  "type": "damage_reflect",
  "source": "shield",
  "duration": 1
}
```

规则：

- 该效果表达“本回合受攻击时反弹护盾值的伤害”。
- 反弹值读取被攻击前的护盾值。
- 具体触发窗口由 `battle-state-machine.md` 定义。

### `next_attack_poison`

让来源单位的下一次攻击在命中时附加毒层。

当前用于影月的 `暗影疾步` 系列。

字段：

- `formula`：命中时附加的毒层数公式。

规则：

- “下一次攻击”包括普通攻击和伤害型卡牌。
- 只有命中目标时才附加毒层。
- 如果下一次攻击是全体攻击、多目标攻击或多段攻击，则每个命中的目标都附加毒层。
- 附加毒层数由 `formula` 计算。
- 该效果在下一次攻击完成后消耗。

### `random_debuff_per_hit`

每段命中后附加随机异常状态。

当前用于瑠璃的 `奥术飞弹·混沌`。

字段：

- `debuffs`：可随机抽取的状态 id 列表。
- `formula`：每次命中附加的状态层数公式。

规则：

- 只在命中时触发；未命中不附加 debuff。
- 多段伤害每段独立触发。
- 每次触发时，从 `debuffs` 列表中随机选择一个状态。
- 附加层数由 `formula` 计算。

### `counter_on_dodge`

闪避攻击时自动反击。

当前用于影月的 `幻影步·残影`。

字段：

- `duration`：反击状态持续回合数。
- `effects`：每次成功闪避后执行的嵌套效果。

示例：

```json
{
  "type": "counter_on_dodge",
  "duration": 1,
  "effects": [
    {
      "type": "damage",
      "damageType": "physical",
      "formula": { "atk": 1 }
    },
    {
      "type": "poison",
      "formula": { "ap": 0.5 }
    },
    {
      "type": "energy_gain",
      "value": 1
    }
  ]
}
```

规则：

- 持续时间内，只要来源单位成功闪避敌方攻击，就触发一次反击。
- 回合内不限制触发次数。
- 嵌套的攻击性效果默认作用于被闪避的攻击者；例如 `damage` 和 `poison` 默认打回本次攻击来源。
- 嵌套的自身或资源效果默认作用于反击状态来源；例如 `energy_gain` 给玩家公共能量。
- 嵌套效果可以显式写 `target` 覆盖默认目标。

## 特殊伤害和组合效果

### `heal_as_damage`

将治疗量转化为伤害。

当前用于“治愈之光·裁决”和“生命绽放·荆棘”：治疗目标后，对敌人造成等同理论治疗量的伤害。

字段：

- `target`：伤害目标。
- `percent`：转化比例；未填写时按 100% 处理。

规则：

- 读取同一张卡内前置 `heal` 的理论治疗量。
- 理论治疗量包含暴击后的治疗放大。
- 理论治疗量不受目标当前缺失 HP 限制；满血目标也贡献治疗量。
- 多目标治疗时，将所有治疗目标的理论治疗量求和，再转化为伤害。
- 造成的伤害目标由 `target` 指定；`治愈之光·裁决` 使用 `random_enemy`。

### `split_damage`

分裂伤害。对主目标和其它敌人造成不同倍率伤害。

示例：

```json
{
  "type": "split_damage",
  "damageType": "magical",
  "primaryApScaling": 5.0,
  "secondaryApScaling": 2.5
}
```

语义：

- 主目标受到 `primaryApScaling * AP` 伤害。
- 其它敌人受到 `secondaryApScaling * AP` 伤害。
- `primaryApScaling` 表示主目标 AP 倍率。
- `secondaryApScaling` 表示副目标 AP 倍率。
- 伤害类型由 `damageType` 指定。
- 该效果使用正常命中、防御、暴击、增伤 / 减伤和护盾结算。

### `percentage_hp_reduction`

减少目标当前 HP 百分比。

字段：

- `target`
- `percent`

规则：

- 当前用于“生命凋零”：敌人当前剩余生命值减少 10%。
- 当前机器可读内容表中，`life_wither`（生命凋零）以及部分节点修正会使用该效果。
- 该效果绕过护盾，直接影响 HP。
- 当前 HP 百分比扣减按目标当前剩余 HP 计算；从算术上不会直接致死，除非后续内容明确使用 100% 或其它特殊规则。
- 影响最大 HP / HP 上限的百分比扣减属于另一类效果，可能导致当前 HP 高于新上限并被压低，因此可能直接致死。

## 卡牌操作

### `card_transform`

批量转换本场战斗内的卡牌。

当前用于初叶的 `神恩降临·裁决`。

字段：

- `transforms`：转换组列表。
- `from`：要被转换的原卡牌 id 列表。
- `to`：转换后的目标卡牌 id。

规则：

- 一个 `card_transform` 可以包含多组 `{ from, to }` 转换。
- 每组转换会把所有 `from` 中匹配到的卡牌实例替换成 `to`。
- 转换范围是本场战斗内该角色当前所有牌区，包括抽牌堆、手牌、弃牌堆、消耗堆和临时牌区。
- 默认持续到本场战斗结束。
- 不改写 run 里的永久卡组。
- 若被转换的卡牌已有升级或临时改造，转换后以 `to` 指向的卡牌定义为准。

示例：

```json
{
  "type": "card_transform",
  "transforms": [
    {
      "from": ["healing_light", "healing_light_up1", "healing_light_up2"],
      "to": "judgment_light"
    },
    {
      "from": ["bloom", "bloom_up1", "bloom_up2"],
      "to": "life_wither"
    }
  ]
}
```

正式卡牌操作使用 `card_operation`：

```ts
interface CardOperationDef {
  operation:
    | "add_card"
    | "copy_card"
    | "remove_card"
    | "modify_card"
    | "transform_card"
    | "add_curse_card";
  targetOwner?: "selected_character" | "equipment_owner" | "self" | "all_characters";
  cardDefId?: CardId;
  count?: number;
  allowIntrinsic?: boolean;
  filter?: CardFilterDef;
  modification?: CardModification;
}
```

规则：

- 操作静态卡牌定义时必须生成或修改 `CardInstance`，不能改写 `CardDef`。
- 装备附带卡牌必须记录 `sourceInstanceId`；装备替换或回收时同步移除。
- 药水造成的新增、复制、删除、改造为永久 run 改动。
- 商店不能直接卖卡，也不能直接删卡；只能通过药水间接触发。

## 敌人和召唤效果

### `summon`

召唤或召回敌人单位。

第 1 章 Boss 莱恩哈特二阶段使用该效果召回 `ashen_knight`（灰烬骑士）。

字段：

- 指定 `enemyId`。
- 指定数量。

示例：

```json
{
  "type": "summon",
  "enemyId": "ashen_knight",
  "count": 1
}
```

规则：

- `enemyId` 表示要召回的敌人定义 id。
- `count` 表示召回数量。
- 第 1 章 MVP 不使用属性覆盖；召回的灰烬骑士使用正式 `ashen_knight` 属性。

## 校验规则

数据加载器必须校验：

- `type` 属于正式 `EffectType`。
- `damage` 必须有 `damageType` 和可计算公式。
- `heal`、`shield`、`poison`、`bleed`、`weakness`、`vulnerable` 必须有可计算公式。
- `buff`、`percent_buff`、`set_stat`、`stat_multiply` 必须指定 `stat`。
- 百分比使用百分数，不使用 0.3 表示 30%。
- `target` 必须是合法 `TargetType`。
- `duration` 必须是数字、`battle`、`run` 或 `permanent`。
- `card_operation` 不得在商店服务中直接作为删牌或买牌功能暴露。
- 内容表必须使用本文件定义的正式 DSL；没有登记语义的字段不得进入机器可读内容表。
