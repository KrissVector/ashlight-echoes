# 战斗状态机规格

> 状态：初版实现规格
> 适用范围：MVP 与后续正式战斗运行时
> 相关文档：`data-schema.md`、`effect-dsl.md`、`mvp-scope.md`

## 目的

本文定义《Ashlight Echoes》的战斗状态机。它面向实现，不面向玩家展示。

本文负责定死：

- 一场战斗从初始化到胜利 / 失败的阶段流转。
- 玩家回合、敌方回合、抢攻、出牌、普攻、敌人行动的执行顺序。
- 牌区、公共能量、AS、状态持续时间、毒 / 流血结算、胜负检查的时机。
- 伤害、治疗、护盾、触发器和卡牌去向如何在同一个动作上下文中结算。

本文不定义 UI 动画、音效、具体敌人 AI 数据格式和奖励发放；这些由 `run-flow.md`、敌人内容表和表现层规格继续定义。

## 核心状态

战斗运行时至少需要以下状态。

```ts
type BattlePhase =
  | "setup"
  | "first_strike"
  | "player_turn_start"
  | "player_action"
  | "player_turn_end"
  | "enemy_turn_start"
  | "enemy_action"
  | "enemy_turn_end"
  | "victory"
  | "defeat";

interface BattleState {
  phase: BattlePhase;
  round: number;
  activeSide?: "player" | "enemy";
  units: BattleUnitState[];
  player: PlayerBattleState;
  enemy: EnemyBattleState;
  pendingTriggers: TriggerState[];
  actionContext?: ActionContext;
  log: BattleEvent[];
  rng: SeededRngState;
}
```

`BattleUnitState` 至少包含：

- 静态引用：`unitId`、`defId`、`side`、`ownerCharacterId?`。
- 当前属性：`hp`、`maxHp`、`atk`、`ap`、`def`、`spd`、`asMax`、`remainingAs`、`crit`、`cdmg`、`evd`、`acc`。
- 战斗资源：`shield`。
- 层数状态：`poisonStacks`、`bleedStacks`、`weaknessStacks`、`vulnerableStacks`。
- 临时状态：buff、debuff、嘲讽、反伤、闪避反击、下次攻击附毒、必定暴击等。
- 生命周期：`isAlive`、`deathOrder?`。

`PlayerBattleState` 至少包含：

- `currentEnergy`
- `maxEnergy`
- `baseMaxEnergy`
- `drawPile`
- `hand`
- `discardPile`
- `exhaustPile`
- `temporaryZone`
- `playedThisBattle`

手牌、牌库、弃牌堆和消耗堆都是共享牌区，不按角色拆分。卡牌实例必须保留 `ownerId`，因为效果来源属性读取卡牌拥有者。

## 总流程

```text
setup
  -> first_strike
  -> player_turn_start
  -> player_action
  -> player_turn_end
  -> enemy_turn_start
  -> enemy_action
  -> enemy_turn_end
  -> player_turn_start
  -> ...
```

任意原子动作后都必须执行胜负检查：

```text
如果全部敌人死亡 -> victory
否则如果全部我方角色死亡 -> defeat
否则继续当前阶段
```

原子动作包括：

- 一段伤害结算。
- 一次治疗。
- 一次状态结算。
- 一个嵌套效果组结算完成。
- 一次敌人行动完成。
- 一个阶段开始或结束触发完成。

胜利优先于失败。若同一个原子动作导致双方同时全部死亡，状态机进入 `victory`。

## 战斗初始化

进入 `setup` 时执行：

1. 根据 encounter 创建敌我双方单位。
2. 我方本场战斗开始时 HP 回满。
3. 所有单位初始护盾为 0，除非 encounter 或节点修正明确提供入场护盾。
4. 清空所有战斗内临时状态。
5. 建立共享牌区：
   - `drawPile` 包含所有上场且已解锁角色的可用卡牌实例。
   - 未解锁的大招卡不进入本场战斗牌库。
   - `hand`、`discardPile`、`exhaustPile`、`temporaryZone` 初始为空。
   - `drawPile` 使用战斗 RNG 洗牌。
6. 设置公共能量：
   - `baseMaxEnergy = 8`
   - `maxEnergy = baseMaxEnergy + modifiers`
   - `currentEnergy = 0`
7. 所有单位 `remainingAs = 0`，等各自阵营回合开始时刷新。
8. `round = 1`。
9. 进入 `first_strike`。

## 抢攻阶段

抢攻只在第一回合正常行动前执行一次。

抢攻单位为全场 `SPD` 最高的存活单位。若多个单位并列最高，使用战斗 RNG 在并列单位中选择，并把选择结果写入战斗日志，确保复盘可重现。

### 我方抢攻

若抢攻单位属于我方：

1. 从该角色本场可用卡牌定义中随机复制 3 张临时牌，放入 `temporaryZone`。
2. 玩家选择其中 1 张免费使用，或选择该角色免费普攻 1 次。
3. 免费使用临时牌时：
   - 不消耗公共能量。
   - 不消耗 AS。
   - 仍然按正常目标、命中、伤害、治疗、触发器和效果顺序结算。
   - 结算结束后临时牌移出战斗，不进入 `discardPile` 或 `exhaustPile`。
4. 免费普攻时：
   - 不消耗 AS。
   - 不获得普攻的 1 点能量。
   - 按正常普攻伤害和触发器结算。
5. 清空未使用的抢攻临时牌。

### 敌方抢攻

若抢攻单位属于敌方：

1. 该敌人执行 1 次 AI 行动。
2. 不消耗 AS。
3. 不使用公共能量。

抢攻结束后固定进入我方第一回合：`player_turn_start`。

## 回合结构

一轮包含一个我方回合和一个敌方回合。

```text
player_turn_start
  -> player_action
  -> player_turn_end
  -> enemy_turn_start
  -> enemy_action
  -> enemy_turn_end
  -> round + 1
```

`round` 在 `enemy_turn_end` 完成后递增。

## 回合开始

我方和敌方共用同一套回合开始流程。

```text
side_turn_start(side):
  1. activeSide = side
  2. 应用该阵营本回合开始生效的延迟资源修正
  3. 结算该阵营存活单位的流血
  4. 胜负检查
  5. 刷新该阵营存活单位 remainingAs
  6. 若 side = player，恢复公共能量并抽牌
  7. 进入该阵营行动阶段
```

### 流血结算

在目标阵营回合开始、任何行动前结算。

对该阵营每个存活单位：

1. 若 `bleedStacks <= 0`，跳过。
2. 造成 `bleedStacks` 点直接 HP 伤害。
3. 该伤害无视 DEF 和护盾。
4. `bleedStacks -= 1`。
5. 若层数降到 0，移除流血状态。

流血可能击杀单位。被流血击杀的单位不能在本回合行动，也不能贡献 ER 或 AS。

### AS 刷新

对该阵营每个存活单位：

```text
remainingAs = asMax
```

`asMax` 必须包含当前仍有效的 AS 上限修正。

### 公共能量恢复

只在我方回合开始恢复公共能量。

```text
currentEnergy = min(maxEnergy, currentEnergy + totalAlivePlayerER)
```

MVP 中角色初始 ER 为 0，因此第一回合通常仍为 0。若后续成长或装备提供 ER，则所有存活上场角色的 ER 相加得到 `totalAlivePlayerER`。

### 抽牌

我方回合开始抽 3 张。

抽牌逐张执行：

1. 如果 `drawPile` 为空，则将 `discardPile` 洗入 `drawPile`，并清空 `discardPile`。
2. 如果 `drawPile` 和 `discardPile` 都为空，本次抽牌停止。
3. 从 `drawPile` 顶部取 1 张。
4. 若 `hand.length < 8`，放入 `hand`。
5. 若手牌已满，新抽到的牌直接放入 `discardPile`。

手牌不会在回合结束时自动弃置。玩家可以跨回合保留手牌。

## 玩家行动阶段

`player_action` 中，玩家可以反复执行以下操作，直到选择结束回合或进入胜负终态：

- 打出 1 张手牌。
- 指挥 1 名存活角色普攻。
- 结束回合。

### 打出手牌

出牌流程：

```text
play_card(cardInstanceId, selectedTargetIds):
  1. 校验阶段为 player_action
  2. 校验卡牌在 hand
  3. 校验卡牌拥有者存活
  4. 校验大招已解锁
  5. 校验 currentEnergy >= card.cost
  6. 校验目标合法
  7. 从 hand 移到 resolving
  8. currentEnergy -= card.cost
  9. 创建 ActionContext
  10. 执行卡牌 effects
  11. 处理卡牌最终去向
  12. 胜负检查
```

出牌不消耗 AS。

若卡牌拥有者死亡，该卡牌不能打出。它可以留在手牌中，但 UI 必须表现为不可用。

### 卡牌最终去向

卡牌效果全部结算后：

1. 若本次结算触发过 `return_to_hand`：
   - 将当前卡牌放回 `hand`。
   - 即使该卡牌 `isExhaust = true`，也不进入 `exhaustPile`。
2. 否则若 `isExhaust = true`：
   - 放入 `exhaustPile`。
3. 否则：
   - 放入 `discardPile`。

如果返回手牌时手牌已满，仍然允许返回，因为该卡牌在使用时已经从手牌移出，正常情况下会释放 1 个手牌位置。若未来效果在同一结算中填满手牌，则 `return_to_hand` 优先，超过上限的处理必须由手牌系统记录为异常并写入日志，不得静默丢牌。

### 普通攻击

普攻流程：

```text
basic_attack(sourceUnitId, selectedEnemyId):
  1. 校验阶段为 player_action
  2. 校验 source 存活
  3. 校验 source.remainingAs > 0
  4. 校验目标为存活敌人
  5. source.remainingAs -= 1
  6. currentEnergy = min(maxEnergy, currentEnergy + 1)
  7. 创建 ActionContext
  8. 执行一次 physical damage，formula = { atk: 1 }
  9. 结算命中触发器和行动触发器
  10. 胜负检查
```

普攻无论命中或未命中，都消耗 AS 并获得 1 点能量。

## 敌方行动阶段

`enemy_action` 中，敌方单位按稳定顺序行动：

1. 按场上敌人站位顺序遍历。
2. 跳过死亡单位。
3. 每个敌人最多行动 `remainingAs` 次。
4. 每次行动前调用敌人 AI 选择行为和目标。
5. 每次行动后 `remainingAs -= 1`。
6. 每次行动后执行胜负检查。

敌人 AI 必须尊重嘲讽：

- 若 AI 行为是单体攻击或单体技能，且存在合法嘲讽目标，则目标必须从嘲讽目标中选择。
- 若多个嘲讽目标存在，按该敌人 AI 的目标选择规则在嘲讽目标中选择。
- 全体攻击不受嘲讽影响。

第 1 章敌人当前可以先用代码模块表达 AI，但必须隔离在清晰的 `EnemyActionResolver` 中，后续迁移到机器可读敌人表时不得重写战斗主循环。

## 回合结束

我方和敌方共用同一套回合结束流程。

```text
side_turn_end(side):
  1. 结算该阵营存活单位的毒
  2. 胜负检查
  3. 结算该阵营单位的回合结束触发器
  4. 递减该阵营单位的临时状态 duration
  5. 递减该阵营单位的 weakness / vulnerable 层数
  6. 移除过期状态
  7. 切换到另一阵营回合
```

### 毒结算

在目标阵营回合结束时结算。

对该阵营每个存活单位：

1. 若 `poisonStacks <= 0`，跳过。
2. 造成 `poisonStacks` 点直接 HP 伤害。
3. 该伤害无视 DEF 和护盾。
4. `poisonStacks -= 1`。
5. 若层数降到 0，移除毒状态。

### 虚弱和脆弱衰减

在目标阵营回合结束时衰减。

```text
weaknessStacks = max(0, weaknessStacks - 1)
vulnerableStacks = max(0, vulnerableStacks - 1)
```

虚弱和脆弱的伤害修正只在伤害窗口生效；衰减本身不造成伤害。

### duration 规则

`duration` 是目标单位自己的阵营回合计数。

- `duration: 1` 的我方自我 buff，在我方当前回合结束时过期。
- `duration: 1` 的敌方 debuff，在敌方下一个回合结束时过期。
- `duration: 2` 表示经过目标阵营 2 次回合结束后过期。
- `duration: "battle"` 持续到本场战斗结束。
- `duration: "run"` 和 `duration: "permanent"` 不应写入战斗临时状态；若由药水或奖励产生，应写入 run 状态。

如果一个效果在目标阵营的回合结束流程中被施加，该效果不参与本次已经开始的 duration 递减，从下一次目标阵营回合结束开始计数。

## 动作上下文

每次出牌、普攻或敌人技能都创建一个 `ActionContext`。

```ts
interface ActionContext {
  actionId: string;
  sourceUnitId: UnitId;
  actionKind: "card" | "basic_attack" | "enemy_skill" | "trigger";
  cardInstanceId?: CardInstanceId;
  cardDefId?: CardId;
  selectedTargets: UnitId[];
  conditionalBonuses: EffectDef[];
  hitRecords: HitRecord[];
  damageRecords: DamageRecord[];
  healRecords: HealRecord[];
  killedUnitIds: UnitId[];
  consumedKillIds: Set<UnitId>;
  returnCurrentCardToHand: boolean;
}
```

`conditional_bonus` 绑定整张卡或整个动作。实现时必须在动作开始前预收集当前动作内所有 `conditional_bonus`，不能因为它写在 `damage` 后面就错过前面的伤害。

`lifesteal`、`heal_as_damage`、`on_kill`、`on_kill_per_target`、`random_debuff_per_hit` 等效果读取当前 `ActionContext` 中已经发生的结果。

## 效果执行顺序

同一个 `effects` 数组按声明顺序执行，但以下效果属于动作级修正或结果读取：

- `conditional_bonus`：动作开始前预收集，结算伤害时读取。
- `lifesteal`：读取当前动作已造成的实际 HP 伤害。
- `heal_as_damage`：读取当前动作已产生的理论治疗量。
- `on_kill` / `on_kill_per_target`：读取当前动作已击杀目标。
- `random_debuff_per_hit`：读取当前动作已命中的 hit 记录。

嵌套 `effects` 创建子动作上下文，但必须保留父动作引用，便于日志和触发来源追踪。嵌套效果不得回写父动作的已读结果，除非该效果明确是父动作触发器的一部分。

## 目标解析

目标解析使用 `effect-dsl.md` 的 `TargetType`。

规则：

- 卡牌先根据 `CardDef.targetType` 校验玩家选择的主目标。
- 效果层 `target` 未填写时，继承当前动作的主目标。
- `selected` 表示当前动作主目标。
- `self` 表示效果来源单位。
- `all_allies` / `all_enemies` / `all_units` 解析为当前合法存活目标列表。
- `random_ally` / `random_enemy` 使用战斗 RNG 从合法存活目标中选择。
- 多目标效果按稳定顺序逐个结算。

`damage.randomTarget = true` 时，每一段 hit 都重新从该效果 `target` 解析出的合法目标池中随机选择 1 个目标。

## 伤害结算管线

所有 `damage` 和普攻使用同一管线。毒、流血、`poison_detonate`、`bleed_detonate`、`percentage_hp_reduction` 和特殊直接 HP 效果有单独规则。

单段伤害流程：

```text
1. 解析目标
2. 命中判定
3. 计算基础伤害
4. DEF 减法
5. 暴击判定和暴击放大
6. 增伤 / 减伤窗口
7. 护盾与 HP 扣减
8. 命中后触发器
9. 击杀记录
```

### 命中判定

```text
hitRate = clamp(source.ACC - target.EVD, 10, 100)
hit = random(1, 100) <= hitRate
```

未命中时：

- 记录 `MISS`。
- 不造成伤害。
- 不触发命中后效果。
- 若防御者有 `counter_on_dodge`，触发闪避反击。

`damageType: "fixed"` 仍然需要命中判定，除非该效果是毒、流血、引爆、反伤、HP 百分比扣减等明确不走命中的特殊伤害。

### 基础伤害

基础伤害由 `formula` 计算并四舍五入。

若计算结果小于 0，按 0 处理。普通 `damage` 在最终伤害窗口后最低为 1；直接 HP 扣减和状态伤害不套用最低 1。

### DEF 减法

若满足以下任一条件，跳过 DEF：

- `damageType: "fixed"`
- `ignoreDefense: true`
- 毒、流血、毒引爆、流血引爆等状态伤害

否则：

```text
afterDefense = max(1, baseDamage - target.DEF)
```

### 暴击

命中后进行暴击判定。

```text
critical = guaranteedCrit || random(1, 100) <= source.CRIT
afterCrit = critical ? round(afterDefense * source.CDMG / 100) : afterDefense
```

`guaranteed_crit` 表示下一次攻击动作必定暴击。一次攻击动作包括一次普攻、一次伤害型卡牌或一次敌人技能；若该动作包含多目标或多段伤害，该动作内所有命中的伤害段都视为暴击。动作完成后消耗 1 次 `guaranteed_crit`。

### 增伤 / 减伤窗口

增伤和减伤在扣除 DEF、结算暴击之后，实际扣护盾或 HP 之前结算。

```text
percentMultiplier = 1 + totalPercentBonus / 100 - totalPercentReduction / 100
modified = round(afterCrit * max(0, percentMultiplier))
modified += target.vulnerableStacks
modified -= source.weaknessStacks
finalDamage = max(1, modified)
```

当前已登记来源：

- `conditional_bonus`：满足条件时提供百分比增伤。
- `vulnerableStacks`：目标受到的伤害增加固定层数。
- `weaknessStacks`：来源造成的伤害减少固定层数。

同一张卡的所有 `damage` 都读取同一组 `conditional_bonus`。对多目标和多段伤害，条件按每个实际受击目标分别判断。

### 护盾和 HP

若 `ignoreShield: true`：

```text
hpDamage = min(target.hp, finalDamage)
target.hp -= finalDamage
```

否则：

```text
shieldDamage = min(target.shield, finalDamage)
target.shield -= shieldDamage
remaining = finalDamage - shieldDamage
hpDamage = min(target.hp, remaining)
target.hp -= remaining
```

`hpDamage` 记录为实际扣到 HP 的伤害，用于 `lifesteal`。若 `target.hp <= 0`，目标死亡。

## 特殊伤害

### 毒和流血

毒和流血是状态结算伤害：

- 不命中判定。
- 不暴击。
- 无视 DEF。
- 无视护盾。
- 直接扣 HP。
- 可以致死。

### 毒引爆

`poison_detonate`：

```text
damage = target.poisonStacks * 3
```

不清空毒层。伤害无视 DEF 与护盾，直接扣 HP。

### 流血引爆

`bleed_detonate`：

```text
damage = target.bleedStacks * target.bleedStacks
target.bleedStacks = 0
```

伤害无视 DEF 与护盾，直接扣 HP。

### HP 百分比扣减

`percentage_hp_reduction` 直接影响当前 HP，绕过护盾。

当前 HP 扣减使用保留生命的计算方式：

```text
newHp = ceil(currentHp * (1 - percent / 100))
reduction = currentHp - newHp
```

当 `0 < percent < 100` 时，该效果不会直接致死。若 `percent >= 100`，`newHp` 可以为 0。

影响最大 HP 的百分比扣减不属于该效果；那类效果会改变 HP 上限，可能导致当前 HP 被压低并致死。

### 反伤

`damage_reflect` 在单位受到攻击时触发。

当前 `source: "shield"` 表示：

1. 在该次攻击伤害应用前读取防御者护盾值。
2. 对攻击者造成等同该护盾值的固定伤害。
3. 反伤不命中、不暴击、不读取攻击者的 `conditional_bonus`。
4. 反伤默认先扣攻击者护盾，再扣 HP。

若一次攻击包含多段命中，每段命中都可以触发一次反伤；每次触发都读取该段伤害应用前的护盾值。

## 治疗结算管线

`heal` 流程：

```text
1. 解析目标
2. 计算基础治疗量
3. 暴击判定和暴击放大
4. 增加目标 HP
5. 记录理论治疗量
```

治疗不命中判定，不受 DEF 影响，可以暴击。

```text
baseHeal = round(evaluateFormula(formula))
critical = random(1, 100) <= source.CRIT
theoreticalHeal = critical ? round(baseHeal * source.CDMG / 100) : baseHeal
target.hp += theoreticalHeal
```

治疗允许超过目标最大 HP。超过最大 HP 的当前 HP 状态直接保留。

`HealRecord.theoreticalAmount` 必须记录暴击后的理论治疗量，供 `heal_as_damage` 读取。满血或超过最大 HP 的目标也贡献完整理论治疗量。

### `heal_as_damage`

`heal_as_damage` 读取同一动作中已经发生的 `HealRecord.theoreticalAmount`。

```text
baseDamage = sum(priorHealRecords.theoreticalAmount)
damage = round(baseDamage * (percent ?? 100) / 100)
```

多目标治疗先求和，再转化为伤害。

该效果造成固定伤害：

- 不命中判定。
- 不暴击。
- 默认先扣护盾，再扣 HP。
- 若未来内容需要绕过护盾，必须显式写 `ignoreShield: true` 或定义新的正式字段后再使用。

## 护盾

`shield`：

```text
shieldGain = round(evaluateFormula(formula))
target.shield = min(target.maxHp * 2, target.shield + shieldGain)
```

护盾可叠加，上限为目标最大 HP 的 200%。

护盾不会在回合结束时自然消失。只有明确有 `duration` 的护盾状态或特殊效果才会过期。

`shield_amplify`：

```text
percent = round(evaluateFormula(formula))
target.shield = min(target.maxHp * 2, round(target.shield * (1 + percent / 100)))
```

## 层数和状态施加

`poison`、`bleed`、`weakness`、`vulnerable` 都使用 `formula` 计算施加层数。

```text
stacks = max(0, round(evaluateFormula(formula)))
target.statusStacks += stacks
```

层数效果可重复叠加。

`buff`、`percent_buff`、`set_stat`、`stat_multiply` 创建临时属性修正。修正必须记录：

- 来源效果 id。
- 目标单位。
- 属性。
- 数值或倍率。
- `duration`。
- 是否 `oncePerBattle`。

属性读取时按以下顺序合成：

```text
base stat
  + flat buffs/debuffs
  -> set_stat overrides if present
  -> percent_buff
  -> stat_multiply
```

若多个 `set_stat` 同时影响同一属性，后施加的效果覆盖先施加的效果。

`conditionMultiplier` 只影响当前效果计算出的数值，不创建额外状态。条件满足时：

```text
finalEffectValue = round(baseEffectValue * multiplier)
```

## 触发器

### `lifesteal`

读取当前动作已经造成的实际 HP 伤害。

```text
heal = round(sum(action.damageRecords.hpDamage) * percent / 100)
source.hp += heal
```

只计算实际扣到 HP 的伤害，不计算被护盾吸收的伤害。

### `on_kill`

当当前动作击杀主要目标时触发一次。

- 主要目标是 `selectedTargets[0]`。
- 若主要目标已经死亡且本动作尚未消费该击杀，执行嵌套效果。
- 执行后标记该击杀已消费。

### `on_kill_per_target`

当前动作每击杀一个目标触发一次。

- 遍历当前动作中尚未消费的 `killedUnitIds`。
- 每个被击杀目标执行一次嵌套效果。
- 嵌套效果的默认目标为触发击杀的目标，除非嵌套效果显式写了 `target`。

### `next_attack_poison`

该状态绑定来源单位。

下一次攻击动作包括：

- 普通攻击。
- 伤害型卡牌。
- 敌人伤害技能。

当该来源的下一次攻击动作命中目标时，对每个命中的目标施加毒层。

- 全体攻击：每个命中的目标都附毒。
- 多段攻击：每个命中的段各自附毒。
- 未命中不附毒。
- 下一次攻击动作完成后，该状态消耗，不论是否命中。

### `random_debuff_per_hit`

读取当前动作中已经命中的 `HitRecord`。

对每条尚未消费的命中记录：

1. 从 `debuffs` 列表中随机选择一个状态。
2. 用 `formula` 计算层数。
3. 对该命中目标施加该状态。

未命中的段不触发。

### `counter_on_dodge`

该状态绑定成功闪避攻击的防御者。

当防御者被攻击且命中判定失败时：

1. 若防御者拥有有效 `counter_on_dodge`，触发一次。
2. 回合内不限制触发次数。
3. 嵌套的攻击性效果默认目标为本次被闪避的攻击者。
4. 嵌套的自身或资源效果默认目标为反击状态来源。
5. 嵌套效果按声明顺序执行。

闪避反击本身可以造成伤害并触发死亡检查，但不应递归触发新的 `counter_on_dodge`，除非未来正式设计明确允许反击反击。

## 资源效果

### `energy_gain`

```text
currentEnergy = min(maxEnergy, currentEnergy + value)
```

### `as_gain`

```text
target.remainingAs += value
```

`as_gain` 只影响当前回合可用 AS，不改变 AS 上限。

### `as_reset`

```text
target.remainingAs = target.asMax
```

### `as_max_increase`

增加目标 `asMax`。

- 若 `duration: "battle"`，持续到本场战斗结束。
- 不自动增加 `remainingAs`，除非同一效果或同一张卡另有 `as_gain`。

### `overload`

`overload` 是延迟资源惩罚，当前用于“下回合能量上限降低”。

当玩家打出包含 `overload` 的卡牌时：

1. 创建一个绑定玩家阵营的延迟资源修正。
2. 在玩家下一个 `player_turn_start` 的资源恢复前生效。
3. `maxEnergy = max(0, baseMaxEnergy + modifiers - energyPenalty)`。
4. 若 `currentEnergy > maxEnergy`，立即压低到 `maxEnergy`。
5. 持续 `duration` 个玩家回合结束后移除。

## 卡牌变形

`card_transform` 在本场战斗内替换卡牌实例引用。

范围包括：

- `drawPile`
- `hand`
- `discardPile`
- `exhaustPile`
- `temporaryZone`

不改写 run 永久卡组。

若被转换卡牌已有升级或临时改造，转换后以 `to` 指向的卡牌定义为准。

## 召唤

`summon` 在战斗中新增敌方单位。

当前第 1 章只需要：

- 按 `enemyId` 创建正式敌人定义。
- 按 `count` 创建指定数量。
- 加入敌方单位列表末尾。
- 新召唤单位初始 HP 为满，护盾为 0，状态为空。
- 若在敌方行动阶段召唤，新单位不在当前敌方行动遍历中行动，从下一次敌方回合开始行动。

第 1 章不使用属性覆盖。莱恩哈特召回亡魂时使用正式 `oathbound_echo` 属性。

## 死亡处理

当单位 `hp <= 0`：

1. 设置 `isAlive = false`。
2. 清空该单位 `remainingAs`。
3. 保留状态记录到战斗日志，但不再触发回合开始 / 结束状态。
4. 若该单位是敌人，记录到当前动作 `killedUnitIds`。
5. 若该单位是我方角色，其手牌中的卡牌不自动移除，但因拥有者死亡不可使用。

死亡单位不能被普通目标选择选中。全体效果只作用于存活单位，除非未来效果明确能复活或影响尸体。

## 战斗结束

### 胜利

进入 `victory` 时：

- 停止所有后续阶段和触发器。
- 保留最终战斗日志。
- 将结果返回 run 流程。
- 不把战斗内临时 HP、护盾、buff、卡牌变形写回 run。
- 按 run 规则处理战后 HP 回满、奖励、剧情解锁和存档。

### 失败

进入 `defeat` 时：

- 停止所有后续阶段和触发器。
- 保留最终战斗日志。
- 将失败结果返回 run 流程。
- run 失败结算由 `run-flow.md` 定义。

## 战斗日志

每个可见或可测试结果都必须写入 `BattleEvent`。

至少记录：

- 阶段切换。
- 抽牌、洗牌、弃牌、消耗。
- 能量变化。
- AS 变化。
- 目标选择。
- 命中 / 未命中。
- 暴击。
- 伤害分解：基础值、DEF 后、暴击后、增伤后、护盾伤害、HP 伤害。
- 治疗分解：基础值、暴击后理论治疗量、目标 HP 变化。
- 护盾变化。
- 状态施加、层数变化、duration 过期。
- 击杀。
- 触发器触发。
- 胜利 / 失败。

战斗日志是自动化测试和 UI 动画的共同数据源。表现层不得通过重新计算战斗逻辑来推断动画结果。

## 校验要求

实现战斗状态机时必须覆盖以下校验：

- 不能在非 `player_action` 阶段打出手牌或普攻。
- 出牌时公共能量不足必须失败，不得进入效果结算。
- 普攻时 AS 不足必须失败。
- 死亡单位不能行动、不能被普通目标选择选中。
- `damage` 必须有 `damageType` 和可计算 `formula`。
- `heal`、`shield`、`poison`、`bleed`、`weakness`、`vulnerable` 必须有 `formula`。
- 所有随机选择必须使用战斗 RNG，并写入日志。
- 每个原子动作后必须胜负检查。
- 触发器不得无限递归；同一触发链必须有深度上限或明确的递归阻断规则。

## MVP 验收用例

至少需要自动化或半自动化覆盖：

1. 第一回合能量为 0，玩家必须普攻或使用免费抢攻牌才能获得出牌资源。
2. 普攻消耗 1 AS、获得 1 能量，命中失败也获得能量。
3. 手牌满时抽牌，新抽牌进入弃牌堆。
4. Deck 为空时，Discard 洗回 Deck。
5. 消耗牌进入 Exhaust，不再洗回 Deck。
6. `return_to_hand` 可以覆盖 `isExhaust`。
7. 毒和流血无视 DEF 与护盾，直接扣 HP。
8. 流血在目标阵营回合开始结算，毒在目标阵营回合结束结算。
9. `conditional_bonus` 即使写在 `damage` 后面，也能绑定同一张卡全部伤害。
10. `嗜血之刃·猎杀` 同时满足两个条件时，总增伤为 +200%。
11. `lifesteal` 只按实际 HP 伤害回血，不计算护盾吸收部分。
12. `heal_as_damage` 使用暴击后的理论治疗量，满血目标也贡献治疗量，多目标治疗先求和。
13. `percentage_hp_reduction` 绕过护盾，且 0-99% 当前 HP 扣减不直接致死。
14. `damage_reflect` 读取攻击伤害应用前的护盾值。
15. `next_attack_poison` 对下一次攻击中每个命中的目标附毒。
16. `random_debuff_per_hit` 只在命中段触发。
17. `counter_on_dodge` 回合内可多次触发，但不会触发反击反击。
18. 莱恩哈特二阶段 `summon` 召回 `oathbound_echo`，新召回单位从下一次敌方回合开始行动。
