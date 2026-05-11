# Run 流程规格

> 状态：初版实现规格
> 适用范围：MVP 新游戏、序章、第 1 章、奖励、商店、事件、存档和章节结算
> 相关文档：`data-schema.md`、`battle-state-machine.md`、`effect-dsl.md`、`mvp-scope.md`

## 目的

本文定义《Ashlight Echoes》从新 run 开始到第 1 章结束的运行流程。

本文负责定死：

- 新游戏如何进入序章和第 1 章。
- 第 1 章随机候选节点如何生成。
- 固定步如何覆盖随机候选节点。
- 节点完成后如何发奖励、触发节点间流程、生成下一步候选。
- 第 1 章普通战、精英战、特殊精英、Boss、商店、事件分别接入哪些内容表和奖励表。
- 存档、失败、通关、第 2 章占位解锁如何落到运行时状态。

本文不定义战斗内部结算；战斗内部流程由 `battle-state-machine.md` 定义。

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
  manualSaveSlotId?: string;
}
```

`availableNodeOptions` 必须写入存档。继续游戏时不得重新随机生成当前步候选节点。

`completedSteps` 必须记录每一步展示过的候选节点和玩家选择的节点，用于执行连续精英、连续纯战斗等硬规则。

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
| 首次步 6 | 固定为灼夜大招解锁战，候选列表只有 1 个节点 |
| 步 13 | 固定为 Boss，候选列表只有 1 个节点 |

“首次步 6”指本 run 中灼夜大招尚未解锁，且局外成长没有让灼夜大招开局解锁。

## 节点类型

```ts
type NodeType =
  | "normal_battle"
  | "elite_battle"
  | "special_elite"
  | "event"
  | "shop"
  | "boss";
```

第 1 章随机候选节点只从以下类型中生成：

- `normal_battle`
- `elite_battle`
- `event`
- `shop`

`special_elite` 和 `boss` 只通过固定步或明确规则加入，不参与普通权重随机。

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

所有随机都必须使用 run RNG，并把结果写入 `completedSteps` 或运行日志，保证读档后不会变。

### 固定步覆盖

固定步优先级最高。

```text
if chapter_1 step 13:
  return [boss option]

if chapter_1 step 6 and shakuya ultimate is not unlocked:
  return [special_elite option]
```

固定步不参与普通候选数量、权重和保底修正。

固定步仍然计入历史：

- 第 6 步特殊精英按精英候选计入连续精英规则。
- 第 6 步特殊精英按纯战斗步计入连续纯战斗规则。
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
| `special_elite` | `encounterId = enc_ch1_shakuya_ultimate_unlock` |
| `event` | `eventId`，来自第 1 章事件池 |
| `shop` | `shopId`，来自第 1 章商店规则 |
| `boss` | `encounterId = enc_ch1_boss_reinhardt` |

如果候选类型对应内容池为空，数据校验失败。实现层不得临时把空池节点替换成其它类型。

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

UI 显示：

- `hintText`
- 节点修正描述
- 可选的风险 / 收益提示

UI 不直接显示机器类型 `nodeType`，除非是固定剧情节点、Boss 或调试界面。

## 节点选择与进入

玩家选择候选节点后：

```text
select_node(optionId):
  1. 校验 optionId 属于 availableNodeOptions
  2. 创建 currentNode
  3. 清空 availableNodeOptions
  4. 应用节点选项修正
  5. 如果节点需要战斗且可用角色超过 3，进入 party_selection
  6. 否则按节点类型进入 battle / shop / event
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
  5. 处理战后随机事件
  6. 进入 between_nodes
  7. 允许使用药水
  8. currentStep += 1
  9. 如果章节结束，进入 chapter_clear
  10. 否则生成下一步候选节点
```

战斗失败不会进入 `complete_node`，直接进入 `run_failed`。

## 节点间药水窗口

药水只能在 `between_nodes` 使用。

可使用时机：

- 战斗奖励结算后。
- 事件结果结算后。
- 商店离开后。
- 下一步候选节点生成前。

使用药水时：

1. 选择背包中的药水。
2. 若药水需要目标，选择目标角色。
3. 执行药水效果，写入 run 状态。
4. 从药水背包移除该药水。

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

### 灼夜大招解锁战奖励

第 1 章第 6 步特殊精英胜利后：

1. 按精英战发放属性成长、金币和药水掉落。
2. 装备掉落改为史诗装备 100%。
3. 解锁灼夜大招 `flame_raven`：
   - 加入本 run 可用大招。
   - 后续战斗中灼夜大招进入牌库。
4. 自动存档。

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

第 1 章商店默认规则：

- 装备商品使用普通装备池，品质普通 100%。
- 药水商品随机 1-3 瓶，品质普通 100%。
- 支持手动存档，价格 10 金币。
- 支持装备 / 药水回收，回收价为售价 40%。
- 支持诅咒装备处理，花费 25 金币。
- 若没有正式情报内容表，不显示情报商品。

商店明确禁止：

- 直接出售卡牌。
- 直接提供删牌服务。

卡牌复制、删除和改造只能通过药水等正式效果间接发生。

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
- 节点修正。
- run flag。
- 后续节点候选生成。

第 1 章事件节点是正式节点类型。若第 1 章事件池为空，数据校验失败；不得在运行时用普通战或空白事件替代。

## 战前 / 战后事件与修正

现有地图设计里，“事件”分为三套机制，不能混用。

### 战后随机事件

战后随机事件是节点修正来源之一，不是 `event` 节点。

定义：

- 时机：节点完成后概率触发，发生在节点奖励结算后、节点间药水窗口前。
- 玩家控制：不可控。
- 影响范围：仅对我方生效。
- 内容来源：`node-modifiers.md` 的“战后随机事件（21 个）”。
- 结果：获得 buff 或 debuff，可能影响下一场战斗、下一次节点选择、多个节点或直到商店 / Boss 清除。

当前已定内容：

- 8 个 buff 类。
- 5 个单场 debuff 类。
- 4 个永久型 debuff 类。
- 4 个章节限定类，其中第 1-2 章限定为“瘴气侵体”。

当前未定内容：

- 战后随机事件的具体触发概率。
- 触发后不同事件的抽取权重。
- 第 1 章是否禁用部分过重的永久型 debuff。

实现层不得自行填写这些概率或权重；需要在后续事件 / 节点修正规格中确认后再写入机器可读表。

### 节点选项自带修正

节点选项自带修正是玩家选择下一节点前可见的战前信息。

定义：

- 时机：候选节点生成时绑定到候选节点；玩家选择该节点后、进入战斗 / 商店 / 事件前生效。
- 玩家控制：可控，因为玩家能在候选节点中比较并选择。
- 影响范围：按修正自身定义，可以影响我方、敌方、双方、run 或下一次节点选择。
- 内容来源：`node-modifiers.md` 的“节点自带修正（25 个）”。
- UI：候选节点必须显示该修正描述，但不直接显示机器类型。

### 事件节点

`event` 节点是地图节点类型之一，不等同于战后随机事件。

定义：

- 时机：玩家在候选节点中选择事件节点后进入。
- 玩法：非战斗文字交互，玩家在事件内做选择。
- 结果：可以给予奖励、惩罚、中性结果或改变 run 状态。
- 内容要求：具体事件内容贴合章节背景。

现有地图模块只给了事件示例，没有定死第 1 章事件列表、文案库和结果表。后续补事件内容时，必须先按该定义设计，不能把战后随机事件当作事件节点替代。

## 存档

### 自动存档

自动存档触发点：

- 序章完成并进入第 1 章时。
- 灼夜大招解锁战胜利后。
- 第 1 章 Boss 胜利后。

自动存档写入 `autoSaveSlotId`，覆盖上一次自动存档。

### 手动存档

手动存档只在商店节点提供。

规则：

- 价格 10 金币。
- 写入 `manualSaveSlotId`。
- 不覆盖自动存档。

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
5. 是否写入局外成长由后续局外成长规格定义；MVP 可以只保留失败记录。

### 第 1 章通关

第 1 章 Boss 胜利后：

1. `ProfileSave.completedChapters` 加入 `chapter_1`。
2. `ProfileSave.metaFlags.chapter_2_placeholder_unlocked = true`。
3. `RunState.phase = "chapter_clear"`。
4. 展示第 1 章通关结果界面。
5. UI 显示第 2 章未开放 / coming next 占位状态。

## MVP 必需内容 ID

第 1 章 MVP 至少需要以下内容表条目。实现可以使用不同显示名，但 id 必须稳定。

### Encounter

| id | 用途 |
|----|------|
| `enc_prologue_reisa_tutorial_1` | 序章第 1 战，玲纱单人教程 |
| `enc_prologue_reisa_shakuya_tutorial_2` | 序章第 2 战，玲纱 + 灼夜教程 |
| `enc_ch1_normal_ashen_wraith_pack` | 第 1 章普通战，灰烬游魂群 |
| `enc_ch1_elite_fallen_knight` | 第 1 章精英战，堕落骑士 |
| `enc_ch1_shakuya_ultimate_unlock` | 第 1 章第 6 步灼夜大招解锁战 |
| `enc_ch1_boss_reinhardt` | 第 1 章 Boss，断誓之盾·莱恩哈特 |

### Reward Table

| id | 用途 |
|----|------|
| `reward_ch1_normal_battle` | 第 1 章普通战奖励 |
| `reward_ch1_elite_battle` | 第 1 章普通精英奖励 |
| `reward_ch1_shakuya_special_elite` | 灼夜大招解锁战奖励 |
| `reward_ch1_boss` | 第 1 章 Boss 奖励 |

### Pool

| id | 用途 |
|----|------|
| `pool_ch1_normal_encounters` | 第 1 章普通战遭遇池 |
| `pool_ch1_elite_encounters` | 第 1 章精英战遭遇池 |
| `pool_ch1_events` | 第 1 章事件池 |
| `pool_ch1_node_modifiers` | 第 1 章节点选项修正池 |
| `pool_equipment_common_all` | 普通装备池 |
| `pool_equipment_rare_all` | 稀有装备池 |
| `pool_equipment_epic_all` | 史诗装备池 |
| `pool_equipment_cursed_all` | 诅咒装备池 |
| `pool_potions_common_all` | 普通药水池 |
| `pool_potions_rare_all` | 稀有药水池 |
| `pool_potions_epic_all` | 史诗药水池 |
| `pool_potions_legendary_all` | 传说药水池 |

## 数据校验

进入可玩版本前，数据加载器必须校验：

- `chapters.json` 中 `chapter_1.totalSteps = 13`。
- `chapter_1.fixedSteps` 包含步 6 特殊精英和步 13 Boss。
- 第 1 章普通战、精英战、事件、商店内容池非空。
- 固定步 encounter id 存在。
- 每个 encounter 都有 reward table。
- 第 1 章节点修正池非空，且修正能绑定到合法节点类型。
- 第 1 章事件池非空。
- 若启用战后随机事件，必须有已确认的触发概率和抽取权重；实现层不得自行补默认概率。
- 商店商品池非空，且第 1 章商店不会直接出售卡牌或删牌服务。
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
- 商店商品生成、购买、存档。
- 事件选项选择和结果。
- 大招解锁。
- 自动 / 手动存档。
- run 失败。
- 章节通关。

## MVP 验收用例

至少需要覆盖：

1. 新游戏从序章第 1 战开始，玲纱单人上场。
2. 序章第 2 战前灼夜加入。
3. 序章完成后自动进入第 1 章并存档。
4. 第 1 章步 1 生成 2 个候选，且至少 1 个普通战。
5. 第 1 章步 5 生成 3 个候选。
6. 首次第 1 章步 6 只显示灼夜大招解锁战。
7. 第 1 章步 12 至少 1 个商店。
8. 第 1 章步 13 只显示 Boss。
9. 候选节点保存后读档不重新随机。
10. 精英候选不会连续 3 步都出现。
11. 纯战斗候选不会连续超过 3 步。
12. 普通战奖励包含成长三选一、10 金币和 30% 药水判定。
13. 精英战奖励包含成长三选一、30 金币、装备和 30% 药水判定。
14. 灼夜大招解锁战胜利后解锁 `flame_raven` 并自动存档。
15. Boss 胜利后发放 Boss 奖励、保存通关、显示第 2 章未开放占位。
16. 商店不会直接出售卡牌或直接删牌。
17. 背包满时获得药水，必须选择丢弃旧药水或放弃新药水。
18. 战斗失败进入 run 失败，不发当前节点奖励。
