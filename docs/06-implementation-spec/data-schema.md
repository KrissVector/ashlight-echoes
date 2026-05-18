# 数据结构规格

> 状态：初版规格
> 适用范围：MVP 与后续正式实现的运行时数据契约
> 相关文档：`mvp-scope.md`、`battle-state-machine.md`、`effect-dsl.md`、`run-flow.md`

## 目的

本文定义《Ashlight Echoes》正式实现使用的数据结构。这里的结构面向 Codex / Claude Code 直接开发。正式内容表位于 `docs/03-content/data/`，实现时应以这些内容表作为角色、卡牌、章节、装备、药水、节点修正、商店、事件和奖励表 `id`（唯一标识）的来源。

本文使用 TypeScript 风格描述字段类型，但不要求最终项目必须使用 TypeScript。实现时可以转成 JSON Schema、Zod、Pydantic、C# class、Godot Resource 或其他等价结构。

## 现有文档盘点

### 正式内容表

| 来源 | 当前内容 | schema 处理 |
|------|---------|-------------|
| `docs/03-content/data/characters.json` | 5 名角色 id、名称、定位、基础属性、初始牌、大招牌 | 作为 `CharacterDef` 的正式内容来源 |
| `docs/03-content/data/cards.json` | 22 张卡牌，其中 20 张固有卡 + 2 张变形牌，包含费用、目标、效果、升级 | 作为 `CardDef` 的正式内容来源；效果字段遵循 `effect-dsl.md` 的正式 DSL |
| `docs/03-content/data/chapters.json` | 序章与 1-5 章的章节标题、主地点、固定步摘要 | 作为 `ChapterDef` 的正式内容来源；遭遇表和节点生成细节在 `run-flow.md` 补齐 |
| `docs/03-content/data/equipment.json` | 装备、装备附带卡、诅咒牌和装备池 | 作为 `EquipmentDef`、装备关联 `CardDef` 和装备池的正式内容来源 |
| `docs/03-content/data/potions.json` | 药水、药水新增卡和药水池 | 作为 `PotionDef`、药水关联 `CardDef` 和药水池的正式内容来源 |
| `docs/03-content/data/node-modifiers.json` | 节点结束后随机事件、节点选项自带修正和第 1 章可用池 | 作为 `NodeModifierDef` 的正式内容来源 |
| `docs/03-content/data/node-hints.json` | 第 1 章节点基础描述、固定节点专属描述和节点自带修正文案 | 作为 `NodeHintDef` 与候选节点展示文案的正式内容来源 |
| `docs/03-content/data/reward-tables.json` | 属性成长池、第 1 章奖励表、金币规则和装备掉落品质分布 | 作为 `RewardTableDef` 与成长池的正式内容来源 |
| `docs/03-content/data/shops.json` | 第 1 章商店定义、商品规则和常驻服务 | 作为 `ShopDef` 的正式内容来源 |
| `docs/03-content/data/events.json` | 第 1 章事件节点、选项、结果和事件池 | 作为 `EventDef` 的正式内容来源 |

对应的权威规则来源如下：

| 来源 | 当前已有内容 | schema 处理 |
|------|-------------|-------------|
| `docs/03-content/characters.md` | 5 名角色基础属性、定位、解锁章节 | 角色设计说明来源；数值以 `docs/03-content/data/characters.json` 为准 |
| `docs/02-systems/cards.md` | 固有卡牌数量、上场限制、牌区规则、升级规则 | 卡牌与牌区规则权威来源 |
| `docs/02-systems/battle.md` | 属性列表、能量、AS、伤害、状态和效果类型 | 战斗属性与效果类型权威来源 |
| `docs/02-systems/roguelike-run.md` | 奖励、金币、成长池、掉落分布 | 奖励和经济权威来源 |
| `docs/02-systems/equipment.md` | 装备稀有度、槽位、掉落、装备列表 | 装备设计来源；机器内容见 `docs/03-content/data/equipment.json` |
| `docs/02-systems/potions.md` | 药水稀有度、背包、使用时机、药水列表 | 药水设计来源；机器内容见 `docs/03-content/data/potions.json` |
| `docs/02-systems/shop.md` | 商店商品、服务、价格、品质分布和 `shops.json` 规则 | 商店设计来源；机器内容见 `docs/03-content/data/shops.json` |
| `docs/02-systems/map-encounters.md` | 章节、步数、节点类型、选项生成、固定步 | 地图节点候选设计来源 |
| `docs/02-systems/node-modifiers.md` | 47 个节点修正与效果描述 | 节点修正设计来源；机器内容见 `docs/03-content/data/node-modifiers.json` |
| `docs/03-content/enemies-chapter-1.md` | 序章与第 1 章敌人数值、AI 行为和遭遇组合 | 序章与第 1 章敌人和遭遇内容来源 |
| `docs/03-content/events-chapter-1.md` | 第 1 章事件节点、选项收益和随机规则 | 第 1 章事件内容来源；机器内容见 `docs/03-content/data/events.json` |
| `docs/03-content/node-hints-chapter-1.md` | 第 1 章节点基础描述、节点自带修正文案和展示规则 | 第 1 章节点提示来源；机器内容见 `docs/03-content/data/node-hints.json` |
| `docs/06-implementation-spec/run-flow.md` | 核心玩法循环、节点结算顺序、奖励和下一节点生成 | 完整 run 流程唯一设计源头 |

## 统一约定

### 命名

- JSON 字段使用 `camelCase`。
- 静态定义 id 使用小写蛇形：`reisa`、`shield_strike`、`chapter_1`。
- 运行时实例 id 使用前缀加唯一后缀：`card_inst_001`、`equip_inst_001`。
- 枚举值使用小写蛇形。
- 百分比属性以百分数存储：`crit: 5` 表示 5%，`cdmg: 150` 表示 150%。
- 缩放系数以倍率存储：`atk: 0.8` 表示 80% ATK。

### 静态定义与运行时实例

必须区分两类数据：

- `*Def`：静态定义，来自内容表，不在 run 中被修改。
- `*Instance` / `*State`：运行时实例和存档状态，会随着 run 改变。

示例：

```ts
CardDef       // "盾击"这张卡的模板
CardInstance  // 本局中某个角色牌池里的一张"盾击"实例
```

装备、药水、卡牌复制、卡牌删除、卡牌改造都只能修改运行时实例或 run 状态，不能改写静态定义。

### 通用枚举

```ts
type ChapterId =
  | "prologue"
  | "chapter_1"
  | "chapter_2"
  | "chapter_3"
  | "chapter_4"
  | "chapter_5";

type StatId =
  | "hp"
  | "atk"
  | "ap"
  | "def"
  | "spd"
  | "as"
  | "crit"
  | "cdmg"
  | "evd"
  | "acc"
  | "presence";

type GlobalStatId =
  | "er"
  | "maxEnergy";

type CharacterRole =
  | "tank"
  | "physical_dps"
  | "magical_dps"
  | "support"
  | "assassin";

type CardRarity =
  | "normal"
  | "rare"
  | "epic"
  | "special"
  | "curse";

type EquipmentRarity =
  | "normal"
  | "rare"
  | "epic"
  | "legendary"
  | "curse";

type PotionRarity =
  | "normal"
  | "rare"
  | "epic"
  | "legendary";

type GrowthRarity =
  | "normal"
  | "rare"
  | "epic"
  | "legendary";
```

`special` 卡牌用于装备牌、药水牌、变形牌、剧情牌等非普通奖励池卡牌。诅咒牌使用 `curse`。

## 属性结构

```ts
interface CharacterStats {
  hp: number;
  atk: number;
  ap: number;
  def: number;
  spd: number;
  as: number;
  crit: number;
  cdmg: number;
  evd: number;
  acc: number;
  presence: number;
}

interface EnemyStats {
  hp: number;
  atk: number;
  ap: number;
  def: number;
  spd: number;
  crit: number;
  cdmg: number;
  evd: number;
  acc: number;
}

interface RunGlobalStats {
  er: number;
  maxEnergy: number;
}
```

MVP 默认全局属性：

```json
{
  "er": 0,
  "maxEnergy": 8
}
```

## 角色定义

```ts
type CharacterId = string;
type CardId = string;

interface CharacterDef {
  id: CharacterId;
  name: string;
  title: string;
  role: CharacterRole;
  unlockChapter: ChapterId;
  baseStats: CharacterStats;
  initialCards: CardId[];
  ultimateCardId: CardId;
  tags?: string[];
}
```

规则：

- `initialCards` 包含 4 张固有卡牌，其中 1 张必须等于 `ultimateCardId`。
- 固有卡牌基础无副本。
- `baseStats.presence` 是敌方默认单体攻击选目标时使用的基础存在感权重。具体数值只以 `docs/03-content/data/characters.json` 为准，其它文档不得重复维护数值表。
- 角色自身不会自然新增固有卡牌；新增、复制、删除、改造来自装备、药水、升级或剧情。
- MVP 可先加载全部角色定义，但 UI 只允许使用已解锁角色。

示例：

```json
{
  "id": "reisa",
  "name": "玲纱",
  "title": "不碎之盾",
  "role": "tank",
  "unlockChapter": "prologue",
  "baseStats": {
    "hp": 180,
    "atk": 14,
    "ap": 0,
    "def": 0,
    "spd": 8,
    "as": 1,
    "crit": 5,
    "cdmg": 150,
    "evd": 5,
    "acc": 100,
    "presence": 150
  },
  "initialCards": [
    "shield_strike",
    "iron_bastion",
    "counter_guard",
    "immovable"
  ],
  "ultimateCardId": "immovable"
}
```

## 卡牌定义

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

type DamageType =
  | "physical"
  | "magical"
  | "fixed";

type CardSourceType =
  | "innate"
  | "ultimate"
  | "equipment"
  | "potion"
  | "curse"
  | "event"
  | "temporary";

interface CardDef {
  id: CardId;
  name: string;
  ownerId?: CharacterId | null;
  sourceType: CardSourceType;
  rarity: CardRarity;
  cost: number;
  targetType: TargetType;
  isExhaust: boolean;
  isUltimate?: boolean;
  description: string;
  effects: EffectDef[];
  upgrades?: CardUpgradeDef[];
  tags?: string[];
}

interface CardUpgradeDef {
  id: string;
  name: string;
  cost: number;
  targetType: TargetType;
  isExhaust: boolean;
  description: string;
  effects: EffectDef[];
}
```

规则：

- 固有卡牌 `sourceType` 为 `innate`，大招为 `ultimate`。
- 每张可升级卡牌最多 2 个升级方向，每张卡在单次 run 中最多升级 1 次。
- 装备或药水新增的卡牌也使用 `CardDef`，但运行时必须生成 `CardInstance` 并绑定实际角色。
- 卡牌是否被战斗内变形不属于 `CardDef` 静态定义，应记录在 `CardInstance.modifications` 或直接变更实例引用。

### 卡牌实例

```ts
type CardInstanceId = string;
type EquipmentInstanceId = string;
type PotionInstanceId = string;

interface CardInstance {
  instanceId: CardInstanceId;
  cardDefId: CardId;
  ownerId: CharacterId;
  sourceType: CardSourceType;
  sourceInstanceId?: EquipmentInstanceId | PotionInstanceId | string;
  upgradeId?: string;
  modifications: CardModification[];
  canRemove: boolean;
  locked: boolean;
}

interface CardModification {
  type:
    | "cost_delta"
    | "set_cost"
    | "set_exhaust"
    | "add_effect"
    | "replace_effects"
    | "transform_to";
  value?: number | boolean | string;
  effects?: EffectDef[];
  sourceInstanceId?: string;
}
```

`canRemove` 是运行时规则开关。具体药水或事件能否移除固有卡牌，由该效果的 `CardOperationDef.allowIntrinsic` 决定；商店边界见 [商店系统](../02-systems/shop.md)。

## 效果定义

`EffectDef` 是所有卡牌、装备、药水、节点修正、事件和敌人技能共享的效果结构。本文定义数据外形；精确执行顺序和每个效果类型的完整语义由 `effect-dsl.md` 与 `battle-state-machine.md` 定义。

```ts
type EffectType =
  | "damage"
  | "heal"
  | "shield"
  | "shield_amplify"
  | "poison"
  | "bleed"
  | "weakness"
  | "vulnerable"
  | "poison_detonate"
  | "bleed_detonate"
  | "taunt"
  | "mark"
  | "lifesteal"
  | "as_gain"
  | "as_reset"
  | "as_max_increase"
  | "buff"
  | "percent_buff"
  | "permanent_buff"
  | "stat_multiply"
  | "debuff"
  | "presence_gain"
  | "draw"
  | "energy_gain"
  | "guaranteed_crit"
  | "conditional_bonus"
  | "on_kill"
  | "on_kill_per_target"
  | "heal_as_damage"
  | "damage_reflect"
  | "next_attack_poison"
  | "random_debuff_per_hit"
  | "counter_on_dodge"
  | "card_transform"
  | "card_operation"
  | "percentage_hp_reduction"
  | "split_damage"
  | "set_stat"
  | "return_to_hand"
  | "overload"
  | "summon"
  | "custom";

type Duration = number | "battle" | "run" | "permanent";

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

interface ConditionDef {
  type:
    | "always"
    | "hp_below"
    | "hp_above"
    | "target_has_status"
    | "self_has_status"
    | "target_is_self"
    | "stat_equals"
    | "turn_mod"
    | "phase_enter"
    | "on_kill"
    | "custom";
  target?: TargetType;
  stat?: StatId | GlobalStatId;
  threshold?: number;
  statusId?: string;
  value?: number | string | boolean;
}

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

### 公式规范

所有可计算数值都写入 `formula`，例如：

```json
{
  "type": "damage",
  "damageType": "physical",
  "formula": {
    "base": 0,
    "atk": 0.8,
    "ap": 1.0
  }
}
```

规则：

- `formula` 中所有字段相加后得到基础数值，再按战斗系统规则取整。
- `percent` 表示百分数，如 `30` 表示 30%。
- 层数效果如毒、流血、虚弱、脆弱也使用 `formula` 计算施加层数。
- 攻击附加流血时，必须先结算伤害；只有目标在伤害结算后没有护盾，流血才会施加。非攻击来源直接施加流血不受该限制。
- `poison_detonate` 读取毒层但不清空毒层。
- `bleed_detonate` 消耗全部流血层。

### 卡牌操作

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

interface CardFilterDef {
  ownerId?: CharacterId;
  sourceType?: CardSourceType[];
  rarity?: CardRarity[];
  tags?: string[];
  excludeUltimate?: boolean;
}
```

商店与卡牌操作的边界见 [商店系统](../02-systems/shop.md)。

## 装备定义

```ts
type EquipmentId = string;

type StatModifierOp =
  | "add_flat"
  | "add_percent"
  | "set"
  | "multiply";

interface StatModifierDef {
  stat: StatId | GlobalStatId;
  op: StatModifierOp;
  value: number;
}

interface EquipmentDef {
  id: EquipmentId;
  name: string;
  rarity: EquipmentRarity;
  description: string;
  statModifiers: StatModifierDef[];
  passiveEffects: EffectDef[];
  cardOperations: CardOperationDef[];
  price?: PriceDef;
  tags?: string[];
}

interface PriceDef {
  min: number;
  max: number;
  sellRatio?: number;
}
```

规则：

- 每名角色最多装备 3 件。
- 装备后锁定，只能被新装备替换；商店处理规则见 [商店系统](../02-systems/shop.md)。
- 史诗、传说、诅咒装备必须至少有一项 `cardOperations`。
- 装备添加的卡牌必须记录 `sourceInstanceId`，替换或回收装备时同步移除。
- 诅咒装备的商店处理规则见 [商店系统](../02-systems/shop.md)。

### 装备实例

```ts
interface EquipmentInstance {
  instanceId: EquipmentInstanceId;
  equipmentDefId: EquipmentId;
  ownerId: CharacterId;
  slotIndex: number;
  locked: boolean;
  disabled: boolean;
  addedCardInstanceIds: CardInstanceId[];
}
```

## 药水定义

```ts
type PotionId = string;
type PotionUseTiming = "between_nodes" | "player_action";

interface PotionDef {
  id: PotionId;
  name: string;
  rarity: PotionRarity;
  description: string;
  targetType: "single_character" | "all_characters" | "none";
  useTiming?: PotionUseTiming;
  consumeOnUse?: boolean;
  effects: EffectDef[];
  price?: PriceDef;
  tags?: string[];
}

interface PotionInstance {
  instanceId: PotionInstanceId;
  potionDefId: PotionId;
  acquiredAtChapter: ChapterId;
  acquiredAtStep: number;
}
```

规则：

- 药水背包全队共享，上限 3。
- `useTiming` 缺省时视为 `between_nodes`，用于永久成长药水。
- `useTiming: "between_nodes"` 的药水只能在节点之间使用，效果写入 `RunCharacterState`、`RunGlobalStats` 或对应卡牌实例。
- `useTiming: "player_action"` 的药水只能在战斗 `player_action` 阶段使用，不消耗 AS，不消耗或产生能量，效果在战斗上下文中结算。
- `consumeOnUse` 缺省时视为 `true`。药水使用后从背包移除。
- 一次性战斗药水的治疗为即时效果；属性增益应使用 `duration: "battle"`，战斗结束后移除。
- 背包满时获得新药水，必须选择丢弃旧药水或放弃新药水。

## 敌人定义

```ts
type EnemyId = string;

type EnemyRole =
  | "tutorial"
  | "normal"
  | "elite"
  | "boss"
  | "summon";

interface EnemyDef {
  id: EnemyId;
  name: string;
  role: EnemyRole;
  chapterId: ChapterId;
  typeTags: string[];
  stats: EnemyStats;
  ai: EnemyAIDef;
  passiveEffects?: EnemyPassiveDef[];
  visualKeywords?: string[];
}

interface EnemyPassiveDef {
  id: string;
  trigger:
    | "battle_start"
    | "aura"
    | "before_targeted"
    | "on_hit_by_basic_attack"
    | "on_hit_by_card"
    | "on_damaged"
    | "on_shield_break"
    | "on_ally_death"
    | "turn_start"
    | "turn_end";
  condition?: ConditionDef;
  targetSelector: TargetSelectorDef;
  effects: EffectDef[];
  cooldown?: number;
  oncePerTurn?: boolean;
}

interface EnemyAIDef {
  type: "weighted_actions" | "phase_weighted_actions" | "custom";
  actions?: EnemyActionDef[];
  phases?: EnemyPhaseDef[];
  customId?: string;
}

interface EnemyPhaseDef {
  id: string;
  condition: ConditionDef;
  enterOnceEffects?: EffectDef[];
  actions: EnemyActionDef[];
}

interface EnemyActionDef {
  id: string;
  name: string;
  weight: number;
  condition: ConditionDef;
  targetSelector: TargetSelectorDef;
  effects: EffectDef[];
  cooldown?: number;
  oncePerBattle?: boolean;
  consumesAction?: boolean;
}

interface TargetSelectorDef {
  type:
    | "random_ally"
    | "lowest_hp_ally"
    | "highest_atk_or_ap_ally"
    | "taunt_target"
    | "presence_weighted_enemy"
    | "marked_enemy"
    | "self"
    | "all_allies"
    | "all_enemies"
    | "custom";
  customId?: string;
}
```

注意：

- 从敌人视角看，`ally` 是敌方阵营，`enemy` 是我方阵营。实现时可以改名为 `opponent` 避免歧义，但数据含义必须固定。
- 敌人使用 `EnemyStats`，不使用角色的 `as`。敌方单位每回合最多执行 1 次 AI 行动；多段攻击属于一次行动内部的多段效果。
- 敌人行动不使用固定优先级。每次敌人需要行动时，先筛选 `condition`、`cooldown`、`oncePerBattle` 均允许的行动，再按 `weight` 做加权随机。
- `weight` 是相对权重，不要求总和为 100。多个行动权重都为 100 时，表示这些行动在条件同时满足时等概率。
- `condition: always` 的行动也参与同一轮权重随机；它不是“保底优先级”，除非其它行动条件都不满足。
- 数组顺序不得影响敌人行动选择。
- 若敌人拥有周期强招，该强招也应作为普通权重行动表达，例如使用“每 N 回合”条件进入候选池；不要绕过权重池做固定触发。
- `presence_weighted_enemy` 表示从敌人视角按玩家角色存在感权重选择单体目标。嘲讽目标存在时，目标池先限制为嘲讽目标，再按存在感权重抽取。
- `marked_enemy` 表示从敌人视角优先选择带有该敌人关注标记的玩家角色；若多个合法标记目标存在，按存在感权重在这些目标中抽取；若没有标记目标，回退到 `presence_weighted_enemy`。
- `customId` 用于需要清晰命名的自定义目标选择，例如最低 HP 比例玩家、当前被瞄准目标、攻击者或固定剧情目标。
- `passiveEffects` 用于被动触发、开场效果和光环，例如染疫尸鬼反施毒、灰壳开场护盾、精英敌人援护。复杂 Boss 或精英机制仍可使用 `ai.customId` 或清晰隔离的 `EnemyActionResolver`。

## 章节、节点和遭遇

```ts
type EncounterId = string;
type NodeModifierId = string;

type NodeType =
  | "normal_battle"
  | "elite_battle"
  | "special_elite"
  | "boss"
  | "shop"
  | "event"
  | "story";

interface ChapterDef {
  id: ChapterId;
  index: number;
  title: string;
  mainLocation: string;
  totalSteps: number;
  fixedSteps: FixedStepDef[];
  optionRules: NodeOptionRuleDef[];
  unlocksAfterClear?: UnlockDef[];
}

interface FixedStepDef {
  step: number;
  firstClearOnly: boolean;
  nodeType: NodeType;
  encounterId?: EncounterId;
  optionCount: number;
  fixedOptions?: FixedStepOptionDef[];
}

interface FixedStepOptionDef {
  optionId: string;
  label: string;
  nodeType: NodeType;
  encounterId?: EncounterId;
  eventId?: string;
  shopId?: string;
  modifierIds: NodeModifierId[];
  summary?: string;
}

interface NodeOptionRuleDef {
  stepRange: [number, number];
  optionCount: number;
  weights: Record<NodeType, number>;
  requiredNodeTypes?: NodeType[];
  forbiddenNodeTypes?: NodeType[];
}

interface NodeOptionDef {
  optionId: string;
  nodeType: NodeType;
  hintId?: NodeHintId;
  hintText: string;
  encounterId?: EncounterId;
  eventId?: string;
  shopId?: string;
  modifierIds: NodeModifierId[];
}

type NodeHintId = string;

interface NodeHintDef {
  id: NodeHintId;
  chapterId: ChapterId;
  text: string;
  exclusive: boolean;
  tags?: string[];
}

interface NodeHintRulesDef {
  composition: "base_hint_then_node_option_modifier_hint";
  selectionOrder: "content_first_then_hint";
  sharedHintMinContentBindings: number;
  exclusiveHintAllowedForTags: string[];
  postNodeRandomModifiersAreExcluded: boolean;
}

interface NodeHintsDataFile {
  schemaVersion: number;
  rules: NodeHintRulesDef;
  baseHints: NodeHintDef[];
  contentHints: {
    encounters: Record<EncounterId, NodeHintId[]>;
    events: Record<EventId, NodeHintId[]>;
    shops: Record<ShopId, NodeHintId[]>;
  };
  nodeOptionModifierHints: Record<NodeModifierId, string[]>;
}

interface EncounterDef {
  id: EncounterId;
  chapterId: ChapterId;
  nodeType: NodeType;
  name: string;
  enemyGroups: EnemyGroupDef[];
  forcedCharacters?: CharacterId[];
  battleStartModifierIds?: NodeModifierId[];
  battleStartUltimateUnlocks?: BattleStartUltimateUnlockDef[];
  rewardTableId: string;
  beforeDialogueId?: string;
  afterDialogueId?: string;
  unlocksOnClear?: UnlockDef[];
  tags?: string[];
}

interface EncounterPoolEntryDef {
  encounterId: EncounterId;
  stepRange?: [number, number];
  requiredFlags?: string[];
  forbiddenFlags?: string[];
  tags?: string[];
}

interface EncounterDataFile {
  schemaVersion: number;
  encounters: EncounterDef[];
  pools: Record<string, EncounterPoolEntryDef[]>;
}

interface BattleStartUltimateUnlockDef {
  characterId: CharacterId;
  ultimateCardId: CardId;
  addToOpeningHand: boolean;
  addToDrawPile: boolean;
  condition?: BattleStartUltimateUnlockConditionDef;
}

interface BattleStartUltimateUnlockConditionDef {
  type: "ultimate_not_unlocked";
  characterId: CharacterId;
  ultimateCardId: CardId;
}

interface EnemyGroupDef {
  enemyId: EnemyId;
  count: number;
  overrides?: Partial<EnemyStats>;
}
```

MVP 必须至少定义：

- 序章教程战 1。
- 序章教程战 2。
- 第 1 章普通战遭遇。
- 第 1 章精英战遭遇。
- 第 1 章第 6 步灼夜追杀剧情节点：固定 2 个分支，分别绑定 `enc_ch1_shakuya_ultimate_unlock` 与 `enc_ch1_shakuya_breakout`。
- 第 1 章 Boss 战。

`battleStartUltimateUnlocks` 用于“战斗开始时解锁大招”的剧情战。执行顺序必须是：先按正常规则抽起手手牌，再解锁这些大招；若 `addToOpeningHand = true`，每张新解锁的大招额外加入手牌 1 张，不占用起手抽牌数量；若 `addToDrawPile = true`，每张新解锁的大招同时加入本场 `drawPile` 1 张，并用战斗 RNG 洗入剩余牌库。

若 `condition.type = "ultimate_not_unlocked"`，只有目标大招在当前 run 中尚未解锁时才执行该项。第 1 章 Boss 使用该条件处理「步 6 选择突出重围后，玲纱和灼夜大招在 Boss 战开场解锁；若步 6 已选择血战到底，则 Boss 不重复解锁」。

## 节点修正定义

```ts
type NodeModifierSource =
  | "post_node_random"
  | "node_option"
  | "event"
  | "shop_service";

type ModifierPolarity =
  | "buff"
  | "debuff"
  | "mixed";

type ModifierApplyTo =
  | "player"
  | "enemy"
  | "both"
  | "run"
  | "next_options";

interface NodeModifierDef {
  id: NodeModifierId;
  name: string;
  source: NodeModifierSource[];
  polarity: ModifierPolarity;
  applyTo: ModifierApplyTo;
  duration: Duration;
  chapterLimit?: ChapterId[];
  effects: EffectDef[];
  cleanseRule?: CleanseRuleDef;
  description: string;
}

interface NodeModifierPoolEntryDef {
  modifierId: NodeModifierId;
  weight: number;
}

interface NodeModifierDataFile {
  schemaVersion: number;
  nodeModifiers: NodeModifierDef[];
  pools: Record<string, NodeModifierId[] | NodeModifierPoolEntryDef[]>;
}

interface CleanseRuleDef {
  type:
    | "after_boss"
    | "shop_cleanse"
    | "shop_cleanse_equipment"
    | "after_battles"
    | "after_steps"
    | "none";
  value?: number;
  price?: number;
}
```

节点修正必须从 `node-modifiers.md` 的 47 项规则转成机器可读数据。第 1 章只加载 `chapterLimit` 允许出现的修正。

规则：

- `source: "post_node_random"` 表示节点结束后随机事件，不是战斗后专属事件。
- 节点结束后随机事件池必须使用 `NodeModifierPoolEntryDef[]`，实现层直接读取 `weight` 做加权随机。
- 当前第 1 章节点完成后 100% 触发 1 个节点结束后随机事件。
- 当前第 1 章节点结束后随机事件池 `pool_ch1_post_node_random_modifiers` 中所有条目 `weight = 100`，表示从池内完全随机、等权抽取。
- 节点选项自带修正池可以继续使用 `NodeModifierId[]`；是否绑定到候选节点由节点生成规则决定。

## 奖励定义

```ts
type RewardType =
  | "growth_choice"
  | "growth_apply"
  | "gold"
  | "potion_drop"
  | "equipment_drop"
  | "card_upgrade"
  | "unlock"
  | "save"
  | "custom";

interface RewardTableDef {
  id: string;
  rewards: RewardDef[];
}

interface RewardDef {
  type: RewardType;
  chance?: number;
  count?: number;
  rarityWeights?: Partial<Record<GrowthRarity | PotionRarity | EquipmentRarity, number>>;
  poolId?: string;
  config?: Record<string, unknown>;
}

type GrowthRewardTargetScope =
  | "selected_character"
  | "run_roster"
  | "global";

type GrowthRewardSelectionMode =
  | "same_for_all_targets"
  | "random_per_target";

interface GrowthApplyRewardConfig {
  targetScope: GrowthRewardTargetScope;
  selectionMode?: GrowthRewardSelectionMode;
  growthOptionId?: string;
  growths?: InlineGrowthDef[];
}

interface InlineGrowthDef {
  op: "add_flat" | "add_percent";
  value: Record<string, number>;
}

interface GrowthOptionDef {
  id: string;
  rarity: GrowthRarity;
  targetType: "single_character" | "global" | "instant";
  stat?: StatId | GlobalStatId;
  op: "add_flat" | "add_percent";
  value: number | Record<string, number>;
}

interface UnlockDef {
  type:
    | "character"
    | "ultimate"
    | "chapter"
    | "run_flag"
    | "meta_flag";
  targetId: string;
}
```

规则：

- 战后属性成长为三选一，三个选项独立生成。
- 每个成长选项最多单独刷新 1 次。
- 金币、药水、装备、卡牌升级按 `roguelike-run.md`、`equipment.md`、`potions.md` 的当前规则生成。
- `growth_apply` 直接应用属性成长，不进入三选一界面。它用于事件奖励、剧情奖励等明确给定成长的来源。
- `growth_apply.config.targetScope = "run_roster"` 表示作用于当前 `RunState.roster` 全员，不限当前上场角色。
- `growth_apply.config.selectionMode = "random_per_target"` 表示每个目标独立从 `poolId` 指向的成长池中随机 1 项；未写时默认同一个成长应用到全部目标。
- 奖励若只写 `poolId` 而不写 `rarityWeights`，则从该池所有条目中等权随机抽取，不按品质先抽。
- `card_upgrade` 可通过 `config.targetCardId`、`config.upgradeIds` 和 `config.choiceCount` 指定只升级某一张牌，例如第 1 章步 6「血战到底！」胜利后只在 `flame_raven_up1` 与 `flame_raven_up2` 中二选一。
- Boss 奖励表不得只写一个“随机装备”；必须能表达第 1 章史诗装备与诅咒装备二选一。

## 商店定义

```ts
type ShopId = string;

interface ShopDef {
  id: ShopId;
  chapterId: ChapterId;
  itemRules: ShopItemRuleDef[];
  services: ShopServiceDef[];
}

type ShopItemType =
  | "potion"
  | "equipment";

interface ShopItemRuleDef {
  itemType: ShopItemType;
  minCount: number;
  maxCount: number;
  rarityWeights: Partial<Record<PotionRarity | EquipmentRarity, number>>;
  poolId?: string;
}

interface ShopServiceDef {
  type:
    | "cleanse_modifier"
    | "cleanse_equipment"
    | "remove_cursed_equipment"
    | "recycle_equipment"
    | "recycle_potion";
  price: ShopServicePrice;
}

type ShopServicePrice =
  | number
  | ShopServicePriceRule;

interface ShopServicePriceRule {
  type: "sell_ratio";
  sellRatio: number;
}
```

商店商品、服务、价格、品质分布和禁止项见 [商店系统](../02-systems/shop.md)。本文只定义 `ShopDef` 的运行时数据结构。

## 事件定义

```ts
type EventId = string;

interface EventDef {
  id: EventId;
  chapterId: ChapterId;
  title: string;
  prompt: string;
  options: EventOptionDef[];
  tags?: string[];
}

interface EventOptionDef {
  id: string;
  text: string;
  requirements?: RequirementDef[];
  costs?: CostDef[];
  results: EventResultDef[];
}

interface RequirementDef {
  type: "gold_at_least" | "has_potion" | "has_modifier" | "custom";
  value?: number | string | boolean;
}

interface CostDef {
  type: "gold" | "potion" | "modifier" | "custom";
  value?: number | string;
}

interface EventResultDef {
  chance?: number;
  effects: EffectDef[];
  rewards?: RewardDef[];
  text?: string;
}

interface EventPoolEntryDef {
  eventId: EventId;
  weight: number;
}

interface EventDataFile {
  schemaVersion: number;
  events: EventDef[];
  pools: Record<string, EventPoolEntryDef[]>;
}
```

第 1 章事件机器内容见 `docs/03-content/data/events.json`。事件节点不会触发普通战或精英战的常规奖励表；事件选项自身给予的金钱、药水、装备、属性成长或卡牌升级是事件奖励。

## 运行时存档

### 局外存档

```ts
interface ProfileSave {
  schemaVersion: number;
  profileId: string;
  completedChapters: ChapterId[];
  unlockedCharacters: CharacterId[];
  unlockedUltimates: CharacterId[];
  metaFlags: Record<string, boolean>;
  createdAt: string;
  updatedAt: string;
}
```

MVP 只要求保存第 1 章已通关、第 2 章占位解锁，以及后续可继续扩展的角色和大招解锁状态。

### Run 存档

```ts
type RunStatus =
  | "active"
  | "won"
  | "failed"
  | "abandoned";

type RunPhase =
  | "prologue"
  | "between_nodes"
  | "choosing_node"
  | "in_battle"
  | "in_shop"
  | "in_event"
  | "reward"
  | "chapter_clear";

interface RunSave {
  schemaVersion: number;
  runId: string;
  seed: string;
  status: RunStatus;
  phase: RunPhase;
  currentChapterId: ChapterId;
  currentStep: number;
  availableNodeOptions: NodeOptionDef[];
  encounterSelectionHistory: EncounterSelectionHistoryEntry[];
  gold: number;
  globalStats: RunGlobalStats;
  roster: RunCharacterState[];
  potionBag: PotionInstance[];
  activeModifiers: ActiveModifierState[];
  pendingRewards: PendingRewardState[];
  routeHistory: RouteHistoryEntry[];
  flags: Record<string, boolean>;
  autoSaveUpdatedAt?: string;
}
```

### 角色运行时状态

```ts
interface RunCharacterState {
  characterId: CharacterId;
  unlocked: boolean;
  available: boolean;
  ultimateUnlocked: boolean;
  stats: CharacterStats;
  cardPool: CardInstance[];
  equipmentSlots: Array<EquipmentInstance | null>;
  blockedUntilStep?: number;
  deadForRun?: boolean;
}
```

规则：

- `stats` 是本局当前永久属性，等于基础属性加上 run 内成长、药水、装备等可持久影响。
- 每场战斗开始时根据 `stats.hp` 回满 HP；战斗内当前 HP 不写回 `RunCharacterState`。
- `equipmentSlots` 固定长度为 3。
- `blockedUntilStep` 用于“追兵将至”等临时不可上场效果。
- `deadForRun` 用于“死亡宣告”等本局永久不可上场效果。

### 待领奖励

```ts
interface PendingRewardState {
  rewardId: string;
  sourceEncounterId?: EncounterId;
  type: RewardType;
  options?: unknown[];
  selected?: boolean;
}

interface RouteHistoryEntry {
  chapterId: ChapterId;
  step: number;
  nodeType: NodeType;
  encounterId?: EncounterId;
  eventId?: EventId;
  shopId?: ShopId;
  modifierIds: NodeModifierId[];
  result: "cleared" | "failed" | "skipped";
}

interface EncounterSelectionHistoryEntry {
  chapterId: ChapterId;
  poolId: string;
  encounterId: EncounterId;
  step: number;
  optionId: string;
}

interface ActiveModifierState {
  modifierId: NodeModifierId;
  source: NodeModifierSource;
  remainingBattles?: number;
  remainingSteps?: number;
  stacks?: number;
  params?: Record<string, number | string | boolean>;
}
```

`availableNodeOptions` 必须随 run 存档写入，读档后不得重新生成当前候选。`encounterSelectionHistory` 用于普通战池和精英战池的“未抽取优先”规则：只有玩家选择了绑定该 encounter 的节点，才记为已抽取；只展示给玩家但未被选择，不写入该历史。

## 战斗状态快照

`battle-state-machine.md` 会定义完整状态机。本文只定义保存或调试时需要的最低结构。

```ts
type BattlePhase =
  | "first_strike"
  | "turn_start"
  | "draw"
  | "player_action"
  | "enemy_action"
  | "turn_end"
  | "victory"
  | "defeat";

interface BattleState {
  battleId: string;
  encounterId: EncounterId;
  turnNumber: number;
  phase: BattlePhase;
  energy: number;
  maxEnergy: number;
  allies: CombatantState[];
  enemies: CombatantState[];
  drawPile: CardInstanceId[];
  hand: CardInstanceId[];
  discardPile: CardInstanceId[];
  exhaustPile: CardInstanceId[];
  temporaryCards: CardInstance[];
  activeModifiers: ActiveModifierState[];
}

interface CombatantState {
  combatantId: string;
  defId: CharacterId | EnemyId;
  side: "player" | "enemy";
  stats: CharacterStats | EnemyStats;
  currentHp: number;
  shield: number;
  remainingAs?: number;
  statuses: StatusInstance[];
  alive: boolean;
}

interface StatusInstance {
  statusId: string;
  stacks?: number;
  duration?: Duration;
  sourceCombatantId?: string;
  effects?: EffectDef[];
}
```

MVP 不支持战斗中存档；但为了调试、自动测试和崩溃恢复，战斗状态结构应保持可序列化。

`remainingAs` 只用于玩家角色。敌方单位没有 AS，不写入 `remainingAs`；敌方行动阶段中每个存活敌人每回合只解析 1 次 AI 行动。

## 推荐数据文件拆分

正式实现建议至少拆为：

```text
docs/03-content/data/chapters.json
docs/03-content/data/characters.json
docs/03-content/data/cards.json
docs/03-content/data/equipment.json
docs/03-content/data/potions.json
docs/03-content/data/enemies.json
docs/03-content/data/encounters.json
docs/03-content/data/node-hints.json
docs/03-content/data/node-modifiers.json
docs/03-content/data/reward-tables.json
docs/03-content/data/shops.json
docs/03-content/data/events.json
```

如果项目采用数据库或资源系统，以上文件可以对应为表、资源目录或构建产物，但字段契约不应改变。

## 校验规则

实现数据加载器时必须校验：

- 所有 `id` 全局唯一，且引用目标存在。
- `CharacterDef.initialCards` 中所有卡牌存在，且 `ownerId` 匹配该角色。
- 每个角色恰好有 1 张 `isUltimate = true` 的大招卡。
- 上场角色数量为 1-3。
- `equipmentSlots` 长度固定为 3。
- 药水背包数量不超过 3。
- 所有百分比属性使用百分数，不使用 0.05 这种小数表达。
- 所有 `rarity` 属于对应枚举。
- 所有 `EffectDef.type` 必须在效果注册表中存在。
- 所有第 1 章 MVP 必需 encounter、enemy、reward table 均存在。
- 商店数据必须满足 [商店系统](../02-systems/shop.md) 的校验规则。
- 商店服务不得包含 `manual_save`；MVP 只使用单自动存档槽。
- 第 1 章 Boss 奖励必须能表达史诗装备与诅咒装备二选一。
- `node-hints.json` 中所有 `contentHints` 引用的 encounter、event、shop 和 hint id 必须存在。
- 除 `exclusive = true` 的固定剧情和 Boss 描述外，每条第 1 章基础描述必须至少被 2 个内容节点引用。
- 每个第 1 章随机候选内容必须至少绑定 2 条基础描述；固定剧情分支和 Boss 必须各绑定 1 条专属描述。
- 每个第 1 章可用 `node_option` 修正必须至少有 1 条节点自带修正文案。

## 内容表缺口

本文只定义结构，不补全部内容。当前没有阻塞第 1 章 MVP 开发的内容表缺口。
