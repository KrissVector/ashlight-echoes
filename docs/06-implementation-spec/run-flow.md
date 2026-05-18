# Run 流程规格

> 状态：核心玩法流程设计源头 / MVP 实现规格
> 适用范围：MVP 新游戏、序章、第 1 章、奖励、商店、事件、存档和章节结算
> 相关文档：`data-schema.md`、`battle-state-machine.md`、`effect-dsl.md`、`mvp-scope.md`

## 目的

本文是《Ashlight Echoes》核心玩法流程的唯一设计源头，同时定义从新 run 开始到第 1 章结束的 MVP 运行流程。

其它文档涉及“节点选择、节点修正、节点结算、奖励、节点结束后随机事件、节点间药水、下一节点生成”的顺序时，必须引用本文，不再重复维护另一套完整流程。

本文负责定死：

- 核心玩法循环的顺序。
- 新游戏如何进入序章和第 1 章。
- 第 1 章随机候选节点如何生成。
- 固定步如何覆盖随机候选节点。
- 节点完成后如何发奖励、触发节点间流程、生成下一步候选。
- 第 1 章普通战、精英战、特殊精英、Boss、商店、事件分别接入哪些内容表和奖励表。
- 存档、失败、通关、第 2 章占位解锁如何落到运行时状态。

本文不定义战斗内部结算；战斗内部流程由 `battle-state-machine.md` 定义。

## 核心玩法循环

正式章节的标准循环如下：

```text
节点候选 2/3 选 1
  -> 每个候选显示模糊敌情 / 节点类型提示 + 可能的 buff/debuff
  -> 选择 1 个候选节点
  -> 应用该节点选项自带的 buff/debuff
  -> 若节点需要战斗，选择上场队伍成员
  -> 进入战斗 / 商店 / 事件
  -> 战斗结束 / 离开商店 / 完成事件
  -> 结算节点结果；若节点结果包含奖励，进入奖励选择
  -> 选取属性成长、装备、药水等奖励
  -> 节点结束后的随机事件（buff/debuff，可能不触发）
  -> 节点间窗口：可使用 useTiming=between_nodes 的药水
  -> 生成下一步候选节点
  -> 回到节点候选选择
```

章节规则可以覆盖候选数量和固定步，例如第 1 章步 1-4 为 2 选 1、步 5-12 为 3 选 1、步 6 为固定剧情分支、步 13 为 Boss。无论候选数量如何变化，核心结算顺序不变。

## 核心状态

```ts
type RunPhase =
  | "profile_menu"
  | "prologue"
  | "between_nodes"
  | "node_selection"
  | "party_selection"
  | "battle"
  | "shop"
  | "event"
  | "reward"
  | "chapter_clear"
  | "run_failed";

interface RunState {
  runId: string;
  seed: string;
  rng: SeededRngState;
  phase: RunPhase;
  currentChapterId: ChapterId;
  currentStep: number;
  completedSteps: CompletedStepRecord[];
  encounterSelectionHistory: EncounterSelectionHistoryEntry[];
  availableNodeOptions: NodeOptionDef[];
  currentNode?: ActiveNodeState;
  roster: RunCharacterState[];
  gold: number;
  potionBag: PotionInstance[];
  activeRunModifiers: ActiveModifierState[];
  pendingRewards: RewardInstance[];
  unlockedUltimatesThisRun: CharacterId[];
  runFlags: Record<string, boolean>;
  autoSaveSlotId?: string;
}
```

`availableNodeOptions` 必须写入存档。继续游戏时不得重新随机生成当前步候选节点。

`completedSteps` 必须记录每一步展示过的候选节点和玩家选择的节点，用于执行连续精英、连续纯战斗等硬规则。

`encounterSelectionHistory` 记录普通战池和精英战池中玩家已经选择过的 encounter，用于优先抽取本 run 尚未选择过的遭遇。

```ts
interface EncounterSelectionHistoryEntry {
  chapterId: ChapterId;
  poolId: string;
  encounterId: EncounterId;
  step: number;
  optionId: string;
}
```

## 新游戏流程

```text
new_game
  -> 创建 ProfileSave
  -> 创建 RunState
  -> prologue step 1
  -> prologue step 2
  -> 自动存档
  -> chapter_1 step 1 node_selection
```

新 `ProfileSave` 初始状态：

- `completedChapters = []`
- `unlockedCharacters = ["reisa"]`
- `unlockedUltimates = []`
- `metaFlags = {}`

新 `RunState` 初始状态：

- `currentChapterId = "prologue"`
- `currentStep = 1`
- 队伍初始只有玲纱。
- 金币初始值为 0。
- 药水背包为空。
- 装备为空。
- 本 run 大招均未解锁，除非局外成长已经解锁。

## 序章流程

序章是线性教程，不生成随机候选节点。

### 序章第 1 步

```text
step 1: 玲纱单人教程战
```

规则：

- 强制上场：玲纱。
- 教学目标：普攻消耗 AS、普攻获得能量、卡牌消耗能量。
- 胜利后不进入路线选择。
- 胜利后进入序章剧情：灼夜加入。

### 序章第 2 步

```text
step 2: 玲纱 + 灼夜双人教程战
```

规则：

- 灼夜加入当前 run 队伍。
- 强制上场：玲纱、灼夜。
- 教学目标：多角色共享牌库和公共能量池。
- 胜利后进入第 1 章。

### 进入第 1 章

序章完成后：

1. `ProfileSave.unlockedCharacters` 加入 `shakuya`。
2. `RunState.currentChapterId = "chapter_1"`。
3. `RunState.currentStep = 1`。
4. 清空 `availableNodeOptions` 和 `currentNode`。
5. 自动存档。
6. 生成第 1 章第 1 步候选节点。
7. 进入 `node_selection`。

## 第 1 章结构

第 1 章固定信息：

- `chapterId = "chapter_1"`
- 标题：银誓余烬
- 主地点：银誓要塞遗址
- 总步数：13

第 1 章步数规则：

| 步数 | 规则 |
|------|------|
| 步 1-4 | 随机生成 2 个候选节点 |
| 步 5-12 | 随机生成 3 个候选节点 |
| 步 6 | 固定为灼夜追杀剧情节点，候选列表固定为 2 个分支：「血战到底！」与「突出重围」 |
| 步 13 | 固定为 Boss，候选列表只有 1 个节点 |

第 1 章步 6 的剧情前提是灼夜和玲纱被血旗佣兵团癫狂残党追上。选择「血战到底！」进入小 Boss 战；选择「突出重围」进入 3 个普通癫狂强盗的追逃战。

第 6 步后，后续第 1 章普通战池允许出现血旗残党追击遭遇 `enc_ch1_normal_bandit_vulture_ambush`。该遭遇不得出现在第 1 章步 1-5。

## 节点类型

```ts
type NodeType =
  | "normal_battle"
  | "elite_battle"
  | "special_elite"
  | "event"
  | "shop"
  | "boss"
  | "story";
```

第 1 章随机候选节点只从以下类型中生成：

- `normal_battle`
- `elite_battle`
- `event`
- `shop`

`special_elite`、`boss` 和 `story` 只通过固定步或明确规则加入，不参与普通权重随机。`story` 节点本身不直接进入战斗，而是生成固定剧情分支选项。

## 候选节点生成

候选节点生成函数：

```ts
function generateNodeOptions(run: RunState): NodeOptionDef[]
```

生成顺序必须是：

1. 检查固定步。
2. 计算本步候选数量。
3. 读取本步权重。
4. 根据历史记录建立限制。
5. 按权重随机生成候选类型。
6. 应用硬规则修正。
7. 为每个候选绑定 encounter / event / shop / modifier / hint。
8. 写入 `run.availableNodeOptions`。

所有随机都必须使用 run RNG，并把结果写入 `availableNodeOptions`、`encounterSelectionHistory`、`completedSteps` 或运行日志，保证读档后不会变。

### 固定步覆盖

固定步优先级最高。

```text
if chapter_1 step 13:
  return [boss option]

if chapter_1 step 6:
  return [
    fixed option "血战到底！" -> special_elite enc_ch1_shakuya_ultimate_unlock,
    fixed option "突出重围" -> normal_battle enc_ch1_shakuya_breakout
  ]
```

固定步不参与普通候选数量、权重和保底修正。

固定步仍然计入历史：

- 第 6 步选择「血战到底！」时，按特殊精英计入连续精英规则，并按纯战斗步计入连续纯战斗规则。
- 第 6 步选择「突出重围」时，按普通战计入普通战历史，并按纯战斗步计入连续纯战斗规则。
- 第 13 步 Boss 是章节终点，不影响下一步生成。

### 候选数量

```text
baseOptionCount =
  step in 1..4  -> 2
  step in 5..12 -> 3
```

若存在 `next_options` 类节点修正，例如“碎片指引”，在 `baseOptionCount` 上增加对应数量。

第 1 章 MVP 没有默认启用“碎片指引”。如果节点修正数据表定义了该效果，生成器必须支持它。

### 权重

第 1 章随机候选节点按步骤区间取权重。

| 步骤位置 | 普通战 | 精英战 | 商店 | 事件 |
|---------|--------|--------|------|------|
| 步 1-4 | 50 | 10 | 10 | 30 |
| 步 5-8 | 40 | 20 | 20 | 20 |
| 步 9-12 | 40 | 30 | 10 | 20 |

权重值按相对权重处理，不要求总和为 100。

### 历史定义

```ts
interface CompletedStepRecord {
  chapterId: ChapterId;
  step: number;
  shownOptions: NodeOptionSummary[];
  selectedOptionId: string;
  selectedNodeType: NodeType;
  completed: boolean;
}
```

硬规则检查的是“展示给玩家的候选节点集合”，不是玩家实际选择的节点。

一个步骤 `hasEliteOption = true`，表示该步展示过至少 1 个 `elite_battle` 或 `special_elite`。

一个步骤 `isPureCombatOptions = true`，表示该步展示给玩家的所有候选节点都是：

- `normal_battle`
- `elite_battle`
- `special_elite`
- `boss`

### 硬规则

硬规则按以下顺序修正候选节点。

#### 步 1 普通战保底

第 1 章第 1 步必须至少有 1 个 `normal_battle`。

若随机生成结果没有普通战，则把第 1 个候选节点替换为普通战。

#### 步 12 商店保底

第 1 章第 12 步必须至少有 1 个 `shop`。

若随机生成结果没有商店，则把最后 1 个候选节点替换为商店。

#### 精英不连续 3 步出现

若前 2 个已完成步骤都 `hasEliteOption = true`，当前步随机生成时不允许出现 `elite_battle`。

处理方式：

1. 当前步抽权重前，从权重表中移除 `elite_battle`。
2. 若硬规则或固定步已经导致当前步是精英，该固定步优先，不做移除。

#### 纯战斗候选不连续超过 3 步

若前 3 个已完成步骤都 `isPureCombatOptions = true`，当前步必须至少出现 1 个非战斗候选。

非战斗候选为：

- `shop`
- `event`

处理方式：

1. 正常随机生成当前步。
2. 若当前步仍是纯战斗候选，则把最后 1 个候选节点替换为 `shop` 或 `event`。
3. 替换类型按当前步非战斗权重随机；若某类型没有可用内容池，则选择另一个可用类型。
4. 若 `shop` 和 `event` 都没有可用内容池，数据校验失败，不允许进入可玩版本。

### 内容池绑定

候选类型确定后，生成器必须绑定具体内容。

| 节点类型 | 必须绑定 |
|----------|----------|
| `normal_battle` | `encounterId`，来自第 1 章普通战遭遇池 |
| `elite_battle` | `encounterId`，来自第 1 章精英战遭遇池 |
| `special_elite` | 第 1 章步 6「血战到底！」固定分支使用 `encounterId = enc_ch1_shakuya_ultimate_unlock` |
| `event` | `eventId`，来自第 1 章事件池 |
| `shop` | `shopId`，来自 [商店系统](../02-systems/shop.md) 和 `shops.json` |
| `boss` | `encounterId = enc_ch1_boss_reinhardt` |

如果候选类型对应内容池为空，数据校验失败。实现层不得临时把空池节点替换成其它类型。

#### 战斗遭遇池抽取规则

普通战池和精英战池都使用“未抽取优先”规则绑定 encounter：

1. 先根据 `chapterId`、`nodeType`、当前步数和剧情条件筛出可用 encounter。
2. 读取本 run 的 `encounterSelectionHistory`。只有玩家选择过绑定该 encounter 的节点，才算该 encounter 已抽取过。
3. 若可用 encounter 中存在未抽取过的条目，则只在未抽取条目中随机抽取。
4. 若可用 encounter 全部已经抽取过，则在全部可用 encounter 中随机抽取。
5. 同一步生成多个普通战或精英战候选时，可以临时排除当前屏幕上已经绑定的同池 encounter，避免同屏重复；该临时排除不写入 `encounterSelectionHistory`，也不算抽取过。若可用池数量不足，则允许重复。

固定剧情战、特殊精英战和 Boss 不消耗普通战池或精英战池的抽取历史，除非它们的 encounter 明确配置为某个随机池成员。

### 节点修正绑定

每个随机候选节点必须绑定 0 个或多个 `node_option` 来源的 `modifierIds`。

规则：

- 修正只能从当前章节允许的 `node-modifiers` 池中抽取。
- 修正必须适配节点类型；只影响战斗的修正不得绑定到纯商店节点，除非该修正明确写着延迟到下一场战斗。
- 修正描述必须在候选节点 UI 中展示。
- 玩家选择节点后，节点修正立即进入 `activeRunModifiers`，并按自身 `duration` 和 `applyTo` 生效。

第 1 章内容完整性要求：必须提供第 1 章可用的节点修正机器可读表。未提供时，节点生成器不得静默生成没有修正的“空白节点”来冒充完整内容。

### 模糊提示

每个 `NodeOptionDef` 必须有 `hintText`。

第 1 章提示文案来自 `docs/03-content/data/node-hints.json`，设计说明见 `docs/03-content/node-hints-chapter-1.md`。

生成规则：

1. 先按本文件规则确定具体节点内容，即绑定 `encounterId` / `eventId` / `shopId`。
2. 再从该内容 id 对应的基础描述列表中使用 run RNG 随机选 1 条，写入 `NodeOptionDef.hintId` 和 `hintText`。
3. 固定剧情分支和章节 Boss 使用专属基础描述，每个固定节点只绑定 1 条描述。
4. 随机候选节点的基础描述必须是共享描述；除固定剧情和 Boss 外，每条基础描述至少绑定 2 个内容节点。
5. 若候选绑定了 `node_option` 节点自带修正，UI 在基础描述后追加该修正的展示文案。
6. `post_node_random` 节点结束后随机事件不参与候选节点描述。

UI 显示：

- `hintText`
- 节点自带修正文案

UI 不直接显示机器类型 `nodeType`，除非是固定剧情节点、Boss 或调试界面。

## 节点选择与进入

玩家选择候选节点后：

```text
select_node(optionId):
  1. 校验 optionId 属于 availableNodeOptions
  2. 若所选节点绑定普通战池或精英战池 encounter，写入 encounterSelectionHistory
  3. 创建 currentNode
  4. 清空 availableNodeOptions
  5. 应用节点选项修正
  6. 如果节点需要战斗且可用角色超过 3，进入 party_selection
  7. 否则按节点类型进入 battle / shop / event
```

MVP 通常只有玲纱和灼夜可用，因此第 1 章可以跳过实际阵容选择 UI，但代码路径必须支持最多 3 人上场。

进入任何战斗前：

- 本场上场角色 HP 回满。
- 战斗使用 `battle-state-machine.md` 初始化。
- 节点修正转成战斗初始 modifier。

## 节点完成流程

节点完成后统一进入节点间流程。

```text
complete_node:
  1. 标记 currentNode completed
  2. 写入 CompletedStepRecord
  3. 发放节点奖励或结算商店 / 事件结果
  4. 处理奖励选择
  5. 处理节点结束后随机事件
  6. 进入 between_nodes
  7. 允许使用节点间药水
  8. currentStep += 1
  9. 如果章节结束，进入 chapter_clear
  10. 否则生成下一步候选节点
```

战斗失败不会进入 `complete_node`，直接进入 `run_failed`。

## 节点间药水窗口

只有 `PotionDef.useTiming = "between_nodes"` 或未填写 `useTiming` 的药水能在 `between_nodes` 使用。`useTiming = "player_action"` 的一次性战斗药水只能在战斗 `player_action` 阶段使用，具体规则见 [战斗状态机](battle-state-machine.md)。

可使用时机：

- 战斗奖励结算后。
- 事件结果结算后。
- 商店离开后。
- 下一步候选节点生成前。

使用药水时：

1. 选择背包中的药水。
2. 校验该药水可在 `between_nodes` 使用。
3. 若药水需要目标，选择目标角色。
4. 执行药水效果，写入 run 状态。
5. 从药水背包移除该药水。

获得药水时若背包已满：

- 玩家选择丢弃已有 1 瓶并获得新药水。
- 或放弃新药水。

Boss 药水奖励背包满时，允许当场使用新药水或丢弃 / 放弃，按药水系统规则执行。

## 奖励流程

### 普通战奖励

普通战胜利后：

1. 属性成长三选一，稀有度：普通 80 / 稀有 20。
2. 金币 +10。
3. 30% 概率掉落 1 瓶药水，品质：普通 70 / 稀有 30。

### 精英战奖励

精英战胜利后：

1. 属性成长三选一，稀有度：稀有 80 / 史诗 20。
2. 金币 +30。
3. 掉落装备，第 1 章普通精英为稀有装备 100%。
4. 30% 概率掉落 1 瓶药水，品质：稀有 70 / 史诗 30。
5. 若 encounter 明确配置 `card_upgrade` 奖励，则触发卡牌升级选择；没有配置时不临时发放卡牌升级。

### 第 1 章步 6 追杀剧情分支奖励

#### 选择「血战到底！」

该分支绑定 `enc_ch1_shakuya_ultimate_unlock`。战斗开始时：

1. 固定应用「血战到底」战场修正：
   - 玩家公共能量回复至能量上限。
   - 敌我全体获得击杀者自身叠暴击 / 暴击伤害、低血提高 ATK 的双方战场规则。
2. 解锁玲纱大招 `immovable` 和灼夜大招 `flame_raven`：
   - 先按正常规则抽取起手手牌；起手抽牌数量不因本次大招解锁减少。
   - 起手抽牌完成后，解锁两张大招，并写入本 run 可用大招。
   - 每张新解锁的大招在本场战斗中额外创建 1 张加入手牌，并创建 1 张加入 `drawPile`；手牌中的大招不占用起手抽牌数量。
   - 本场战斗中两张大招开局即可从手牌使用，之后也可以从牌库抽到。

胜利后：

1. 按精英战发放属性成长、金币和药水掉落。
2. 装备掉落改为史诗装备 100%。
3. 触发 1 次灼夜大招升级选择：
   - 只在 `flame_raven_up1` 和 `flame_raven_up2` 中选择。
   - 选择后立即替换本 run 中的灼夜大招。
4. 设置剧情标记 `shakuya_amulet_origin_revealed = true`：
   - 写入当前 run 的 `runFlags`。
   - 写入局外 `ProfileSave.metaFlags`，用于后续回到灼夜故乡时识别“父母遗物护身符来自地下古代遗迹，是背后势力想夺回研究的完成品样本”的剧情状态。
5. 设置 `ch1_step6_last_stand_cleared = true`。第 1 章 Boss 开场不重复解锁玲纱和灼夜的大招。
6. 自动存档。

#### 选择「突出重围」

该分支绑定 `enc_ch1_shakuya_breakout`。战斗开始时：

1. 固定应用「突出重围」剧情修正 `story_breakout_hp_80`：我方全体当前 HP 变为最大 HP 的 80%。
2. 本分支不立即解锁玲纱和灼夜的大招，也不发放灼夜大招升级机会。

胜利后：

1. 按第 1 章普通战发放奖励。
2. 设置 `ch1_step6_breakout_cleared = true`。
3. 自动存档。

若玩家选择「突出重围」，玲纱大招 `immovable` 和灼夜大招 `flame_raven` 都在第 1 章 Boss 战开场解锁：仍然先按正常规则抽起手手牌，再把新解锁的大招加入手牌和本场战斗牌库。若玩家已在「血战到底！」分支解锁两人大招，Boss 开场不重复解锁。

### Boss 奖励

第 1 章 Boss 胜利后：

1. 属性成长三选一，稀有度：史诗 75 / 传说 25。
2. 金币 +100。
3. 药水必定掉落 1 瓶，品质：史诗 75 / 传说 25。
4. 装备奖励为二选一：
   - 选项 A：史诗装备池随机 1 件。
   - 选项 B：诅咒装备池随机 1 件。
5. 标记第 1 章通关。
6. 解锁第 2 章占位状态。
7. 自动存档。
8. 进入 `chapter_clear`。

### 属性成长选择

属性成长奖励生成 3 个选项。

每个选项独立生成：

```text
rarity -> target -> growth option
```

规则：

- 目标可以是单个角色、全局或即时金币，按成长池定义。
- 每个选项最多刷新 1 次。
- 刷新费用为 5 金币。
- 刷新只替换该选项，不替换其它选项。
- 玩家最终选择 1 个选项，其余丢弃。

## 商店流程

进入商店节点时，根据 `ShopDef` 生成商品并写入 `currentNode.shopInventory`。商品生成后必须随 run 存档保存，读档不得重新刷新。

商店商品、服务、价格、品质分布和禁止项以 [商店系统](../02-systems/shop.md) 为准。本文只定义 run 流程中的接入、存档和离开节点顺序。

离开商店后进入节点完成流程。商店本身不发战斗奖励。

## 事件流程

事件节点进入后：

1. 读取 `EventDef`。
2. 展示事件文本和可选项。
3. 校验每个选项的 `requirements`。
4. 玩家选择 1 个合法选项。
5. 扣除 `costs`。
6. 按 `results` 结算奖励和效果。
7. 进入节点完成流程。

事件结果可以影响：

- 金币。
- 药水。
- 装备。
- 角色属性成长。
- 卡牌升级。
- 节点修正。
- run flag。
- 后续节点候选生成。

第 1 章事件节点是正式节点类型。若第 1 章事件池为空，数据校验失败；不得在运行时用普通战或空白事件替代。

事件节点不会触发普通战或精英战的常规奖励表。选择事件节点的隐藏代价是不会自动获得战斗节点默认的金币、药水判定和战后常规属性成长三选一；事件选项自身给予的金钱、药水、装备、属性成长或卡牌升级属于事件奖励。

## 节点事件与修正

现有地图设计里，“事件”分为三套机制，不能混用。

### 节点结束后随机事件

节点结束后随机事件是节点修正来源之一，不是 `event` 节点。

定义：

- 时机：节点完成后概率触发，发生在节点奖励结算后、节点间药水窗口前。
- 玩家控制：不可控。
- 影响范围：默认仅对我方生效；若修正条目明确写“敌我双方”，则作为战场规则对双方生效。
- 内容来源：`node-modifiers.md` 的“节点结束后随机事件（22 个）”。
- 机器池：第 1 章使用 `pool_ch1_post_node_random_modifiers`，池内条目为 `{ modifierId, weight }`。
- 触发概率：第 1 章节点完成后 100% 触发 1 次。
- 结果：获得 buff 或 debuff，可能影响下一场战斗、下一次节点选择、多个节点或直到商店 / Boss 清除。

当前已定内容：

- 9 个 buff 类。
- 5 个单场 debuff 类。
- 4 个永久型 debuff 类。
- 4 个章节限定类，其中第 1-2 章限定为“瘴气侵体”。

当前已定触发规则与权重：

- 第 1 章每次节点完成后 100% 触发 1 个节点结束后随机事件。
- 第 1 章节点结束后随机事件池内所有事件 `weight = 100`，表示从池内完全随机、等权抽取。
- 固定剧情分支和 Boss 节点完成后的触发规则若无额外覆盖，也按上述规则执行；若后续需要排除某类节点，必须在本文件和机器表中明确写出。

### 节点选项自带修正

节点选项自带修正是玩家选择下一节点前可见的战前信息。

定义：

- 时机：候选节点生成时绑定到候选节点；玩家选择该节点后、进入战斗 / 商店 / 事件前生效。
- 玩家控制：可控，因为玩家能在候选节点中比较并选择。
- 影响范围：按修正自身定义，可以影响我方、敌方、双方、run 或下一次节点选择。
- 内容来源：`node-modifiers.md` 的“节点自带修正（25 个）”。
- UI：候选节点必须显示该修正描述，但不直接显示机器类型。

### 事件节点

`event` 节点是地图节点类型之一，不等同于节点结束后随机事件。

定义：

- 时机：玩家在候选节点中选择事件节点后进入。
- 玩法：非战斗文字交互，玩家在事件内做选择。
- 结果：可以给予奖励、惩罚、中性结果或改变 run 状态。
- 内容要求：具体事件内容贴合章节背景。

第 1 章正式事件内容见 `docs/03-content/events-chapter-1.md`，机器内容表见 `docs/03-content/data/events.json`。事件节点不得用节点结束后随机事件替代。

## 存档

### 自动存档

自动存档触发点：

- 新 run 创建后。
- 每个节点完成、奖励和节点结束后随机事件结算完成后。
- 序章完成并进入第 1 章时。
- 第 1 章 Boss 胜利并写入局外进度后。

自动存档写入 `autoSaveSlotId`，覆盖上一次自动存档。

### 单自动存档槽

MVP 只有 1 个 run 自动存档槽。

规则：

- 继续游戏只读取自动存档槽。
- 新游戏若发现自动存档槽中存在进行中的 run，必须显示覆盖确认。
- 战斗中不写自动存档；关闭或崩溃后继续游戏回到最近一次节点完成后的自动存档。
- run 失败时清空自动存档槽，玩家必须从新 run 开始。
- 第 1 章通关后写入局外进度，然后清空当前 run 自动存档槽。
- 没有自动存档时，继续游戏按钮置灰或隐藏。

### 读档

读档必须恢复：

- `RunState.rng`
- 当前章节和步数。
- 当前候选节点。
- 当前节点商品、事件状态或战斗前状态。
- 角色成长、装备、药水、卡牌实例和大招解锁。
- 已完成步骤历史。

读档不得重新生成当前候选节点、商店商品或奖励选项。

## 失败和通关

### Run 失败

若战斗进入 `defeat`：

1. `RunState.phase = "run_failed"`。
2. 停止当前节点。
3. 展示失败结果。
4. 不发放当前节点奖励。
5. 清空当前 run 自动存档。
6. 是否写入局外成长由后续局外成长规格定义；MVP 可以只保留失败记录。

### 第 1 章通关

第 1 章 Boss 胜利后：

1. `ProfileSave.completedChapters` 加入 `chapter_1`。
2. `ProfileSave.metaFlags.chapter_2_placeholder_unlocked = true`。
3. `RunState.phase = "chapter_clear"`。
4. 自动保存局外进度。
5. 清空当前 run 自动存档。
6. 展示第 1 章通关结果界面。
7. UI 显示第 2 章未开放 / coming next 占位状态。

## MVP 必需内容 ID

第 1 章 MVP 至少需要以下内容表条目。实现可以使用不同显示名，但 id 必须稳定。

### Encounter

| id | 用途 |
|----|------|
| `enc_prologue_reisa_tutorial_1` | 序章第 1 战，玲纱单人教程 |
| `enc_prologue_reisa_shakuya_tutorial_2` | 序章第 2 战，玲纱 + 灼夜教程 |
| `enc_ch1_normal_outer_husks` | 第 1 章普通战，步 1-5，低压活尸群 |
| `enc_ch1_normal_vulture_husks` | 第 1 章普通战，步 1-5，活尸群加入荒原尸鹫 |
| `enc_ch1_normal_void_wolf_whelps` | 第 1 章普通战，步 1-5，虚空狼与幼狼 |
| `enc_ch1_normal_scavenger_vultures` | 第 1 章普通战，步 1-5，食腐尸鬼与荒原尸鹫 |
| `enc_ch1_normal_plague_scavengers` | 第 1 章普通战，步 1-5，食腐尸鬼与染疫尸鬼 |
| `enc_ch1_normal_plague_vulture_pack` | 第 1 章普通战，步 7+，毒素、续航和高速干扰 |
| `enc_ch1_normal_void_wolf_pack` | 第 1 章普通战，步 7+，中后段虚空狼群 |
| `enc_ch1_normal_ashen_crossfire` | 第 1 章普通战，步 7+，灰烬骑士与灰誓弩手 |
| `enc_ch1_normal_hollow_armory` | 第 1 章普通战，步 7+，空甲守卫与活尸士兵 |
| `enc_ch1_normal_bandit_vulture_ambush` | 第 1 章普通战，步 7+，血旗残党追击与荒原尸鹫 |
| `enc_ch1_normal_ashen_knight_squad` | 第 1 章普通战，步 7+，灰烬骑士小队 |
| `enc_ch1_normal_hollow_crossfire` | 第 1 章普通战，步 7+，空甲守卫、灰誓弩手与活尸士兵 |
| `enc_ch1_elite_void_wolf_king` | 第 1 章精英战，虚空狼王 |
| `enc_ch1_elite_ashen_commander` | 第 1 章精英战，骑士统领 |
| `enc_ch1_elite_bound_oath_bulwark` | 第 1 章精英战，缚誓壁垒 |
| `enc_ch1_elite_bell_tower_binder` | 第 1 章精英战，钟楼缚魂师 |
| `enc_ch1_shakuya_ultimate_unlock` | 第 1 章第 6 步「血战到底！」，癫狂的强盗首领追夺灼夜的遗物护身符 |
| `enc_ch1_shakuya_breakout` | 第 1 章第 6 步「突出重围」，灼夜和玲纱从后方突围撤退 |
| `enc_ch1_boss_reinhardt` | 第 1 章 Boss，断誓之盾·莱恩哈特 |

### Reward Table

| id | 用途 |
|----|------|
| `reward_ch1_normal_battle` | 第 1 章普通战奖励 |
| `reward_ch1_elite_battle` | 第 1 章普通精英奖励 |
| `reward_ch1_shakuya_special_elite` | 第 1 章步 6「血战到底！」奖励 |
| `reward_ch1_boss` | 第 1 章 Boss 奖励 |

### Pool

| id | 用途 |
|----|------|
| `pool_ch1_normal_encounters` | 第 1 章普通战遭遇池；按步 1-5 / 步 7+ 分段过滤，并优先抽取本 run 未选择过的遭遇 |
| `pool_ch1_elite_encounters` | 第 1 章精英战遭遇池；优先抽取本 run 未选择过的遭遇 |
| `pool_ch1_events` | 第 1 章事件池；条目为 `{ eventId, weight }`，当前权重均为 100 |
| `pool_ch1_post_node_random_modifiers` | 第 1 章节点结束后随机事件池；条目为 `{ modifierId, weight }`，当前权重均为 100 |
| `pool_ch1_node_modifiers` | 第 1 章节点选项修正池 |
| `pool_equipment_cursed_all` | 诅咒装备池 |
| `pool_potions_legendary_all` | 传说药水池 |

## 数据校验

进入可玩版本前，数据加载器必须校验：

- `chapters.json` 中 `chapter_1.totalSteps = 13`。
- `chapter_1.fixedSteps` 包含步 6 追杀剧情节点和步 13 Boss；步 6 必须有「血战到底！」与「突出重围」两个固定分支。
- 第 1 章普通战、精英战、事件、商店内容池非空。
- 固定步及其固定分支绑定的 encounter id 存在。
- 每个 encounter 都有 reward table。
- 第 1 章节点修正池非空，且修正能绑定到合法节点类型。
- 第 1 章事件池非空。
- `events.json` 中所有事件选项至少有 1 个结果，且结果引用的奖励类型、卡牌、升级、成长池、装备池和药水池都存在。
- `node-hints.json` 中所有随机候选内容都至少有 2 条基础描述；固定剧情分支和 Boss 各有且只有 1 条专属描述。
- 除专属描述外，每条基础描述至少绑定 2 个内容节点。
- 每个第 1 章可用 `node_option` 修正至少有 1 条展示文案。
- 节点结束后随机事件触发概率必须读取章节规则；第 1 章为 100%。抽取权重必须直接读取机器池条目，不得依赖数组顺序。
- 商店数据满足 [商店系统](../02-systems/shop.md) 的校验规则。
- Boss 奖励能表达史诗装备和诅咒装备二选一。
- 所有随机表使用权重，不依赖数组顺序当概率。
- 所有生成结果写入 run 存档，读档不重随机。

## Run 日志

Run 层必须记录以下事件，供调试、复盘和验收：

- 新 run 创建。
- 序章步骤完成。
- 进入章节。
- 每一步候选节点生成结果。
- 硬规则修正原因。
- 玩家选择的节点。
- 节点修正获得和过期。
- 奖励生成、刷新和选择。
- 药水获得、使用、丢弃。
- 装备获得、装备、替换、回收。
- 商店商品生成和购买。
- 事件选项选择和结果。
- 大招解锁。
- 自动存档。
- run 失败。
- 章节通关。

## MVP 游玩验收关注点

形成可玩版本后，实际游玩时重点确认：

1. 新游戏从序章第 1 战开始，玲纱单人上场。
2. 序章第 2 战前灼夜加入。
3. 序章完成后自动进入第 1 章并存档。
4. 第 1 章步 1 生成 2 个候选，且至少 1 个普通战。
5. 第 1 章步 5 生成 3 个候选。
6. 首次第 1 章步 6 固定显示 2 个分支：「血战到底！」和「突出重围」。
7. 第 1 章步 12 至少 1 个商店。
8. 第 1 章步 13 只显示 Boss。
9. 候选节点保存后读档不重新随机。
10. 精英候选不会连续 3 步都出现。
11. 纯战斗候选不会连续超过 3 步。
12. 普通战奖励包含成长三选一、10 金币和 30% 药水判定。
13. 精英战奖励包含成长三选一、30 金币、装备和 30% 药水判定。
14. 选择「血战到底！」时，战斗开始先抽起手手牌，再解锁 `immovable` 和 `flame_raven` 并把两张大招加入手牌和牌库；胜利后触发 1 次灼夜大招升级选择并自动存档。选择「突出重围」时，第 1 章 Boss 战开场用同样流程补解锁两人大招。
15. Boss 胜利后发放 Boss 奖励、保存通关、显示第 2 章未开放占位。
16. 商店行为符合 [商店系统](../02-systems/shop.md) 的规则。
17. 背包满时获得药水，必须选择丢弃旧药水或放弃新药水。
18. 战斗失败进入 run 失败，不发当前节点奖励。
