# Custom Effect Handlers

> 状态：正式实现约定
> 目的：在不扩展效果 DSL 的前提下，登记所有 `customId` 的语义和触发窗口。

## 定位

`custom` 效果用于当前通用 DSL 不适合表达的特殊机制。它不是自由文本，也不是让实现层根据 `description`（描述文本）猜逻辑的入口。

实现时必须建立集中注册表：

```ts
type CustomEffectWindow =
  | "install"
  | "uninstall"
  | "battle_start"
  | "turn_start"
  | "turn_end"
  | "before_action"
  | "after_action"
  | "before_card_play"
  | "after_card_play"
  | "before_basic_attack"
  | "after_basic_attack"
  | "before_damage"
  | "after_damage"
  | "on_hit"
  | "on_miss"
  | "on_crit"
  | "on_kill"
  | "on_damaged"
  | "on_heal"
  | "on_shield_gain"
  | "on_card_exhausted"
  | "on_draw"
  | "on_hand_state"
  | "before_reward"
  | "after_reward"
  | "node_option_generation"
  | "node_enter"
  | "node_complete"
  | "shop_service"
  | "effect_resolution";
```

```ts
interface CustomEffectHandler {
  customId: string;
  windows: CustomEffectWindow[];
  execute(ctx: CustomEffectContext): void;
}
```

规则：

- 所有 `customId` 必须在本文登记后才能进入机器内容表。
- 所有 handler 必须使用战斗或 run RNG，不得调用不可复现随机。
- 所有可见结果必须写入 Battle log 或 Run log。
- 来自装备的 handler 必须能按 `sourceInstanceId` 卸载；装备替换或回收后不再生效。
- 来自药水的永久 handler 写入 run 状态，不可撤销。
- 来自节点修正的 handler 按 `ActiveModifierState` 的剩余战斗、剩余步数或清除规则过期。
- 不允许 handler 根据中文 `description` 解析数值；数值以 `customId` 本文登记语义或效果字段为准。

## 触发窗口说明

| 窗口 | 中文说明 | 常见用途 |
|------|----------|----------|
| `install` | 效果安装时 | 装备、药水、节点修正写入运行时状态 |
| `uninstall` | 效果卸载时 | 装备替换、节点修正过期、商店净化 |
| `battle_start` | 战斗初始化后、抢攻前 | 开场扣血、开场抽牌修正、命运赌盘 |
| `turn_start` | 阵营回合开始 | 每回合扣血、回血、毒雾、能量修正 |
| `turn_end` | 阵营回合结束 | 每回合回血、未受伤检测 |
| `before_action` / `after_action` | 任意行动前后 | 献祭之心、行动后惩罚 |
| `before_card_play` / `after_card_play` | 出牌前后 | 费用限制、出牌扣血、出牌后消耗手牌 |
| `before_basic_attack` / `after_basic_attack` | 普攻前后 | 普攻多段、普攻抽牌、普攻附加效果 |
| `before_damage` / `after_damage` | 单段伤害前后 | 增伤、延迟伤害、溅射 |
| `on_hit` / `on_miss` | 命中或未命中后 | 附毒、赌徒之剑未命中惩罚 |
| `on_crit` | 暴击后 | 暴击回能、暴击吸血、暴击抽牌 |
| `on_kill` | 击杀记录后 | 噬魂、连锁反应、诅咒装备击杀收益 |
| `on_damaged` | 受到伤害后 | 反噬之盾、受伤记录 |
| `on_heal` | 治疗后 | 治疗转化或治疗加成扩展 |
| `on_shield_gain` | 获得护盾后 | 水晶壁垒、护盾牌扩展 |
| `on_card_exhausted` | 卡牌进入消耗堆后 | 灰烬之书、焚世之冠 |
| `on_draw` / `on_hand_state` | 抽牌或手牌状态变化 | 不可打出诅咒牌、额外费用 |
| `before_reward` / `after_reward` | 奖励生成前后 | 金币加成、双倍掉落 |
| `node_option_generation` | 生成下一步候选节点时 | 碎片指引、不祥预感、盲选 |
| `node_enter` | 进入节点时 | 怪物数量倍率、战斗环境绑定 |
| `node_complete` | 节点完成后 | 死亡宣告步数、节点结束后随机事件 |
| `shop_service` | 商店服务执行时 | 净化、修理、诅咒装备处理 |
| `effect_resolution` | 当前 custom 效果被主动结算时 | 装备卡、药水牌、诅咒牌的主动效果 |

## 敌人机制

| `customId` | 窗口 | 语义 |
|------------|------|------|
| `enemy_void_corrosion_apply_2_vulnerable_2_weakness_if_target_unshielded_on_hit` | `after_damage` | 用于癫狂的强盗、癫狂的强盗首领和断誓之盾·莱恩哈特。该敌人的攻击段命中并完成本段伤害后，读取同段 `DamageRecord`；若 `shieldDamage = 0` 且 `targetShieldAfterDamage <= 0`，视为目标命中前没有护盾，对该目标施加 2 层脆弱和 2 层虚弱；若目标命中前已有护盾，即使本次伤害打破护盾也不施加，并记录 `VOID_CORROSION_BLOCKED_BY_SHIELD`。 |
| `enemy_void_corrosion_self_1_vulnerable_1_weakness_on_attack_hit` | `on_damaged` | 用于癫狂的强盗、癫狂的强盗首领和断誓之盾·莱恩哈特。该敌人每次被攻击段命中后，自身获得 1 层脆弱和 1 层虚弱。多段攻击每段独立触发；毒、流血等非攻击状态伤害不触发。 |
| `enemy_plague_ghoul_retaliate_poison_1_to_unshielded_basic_attacker` | `on_hit_by_basic_attack` | 染疫尸鬼被我方普攻命中后，若攻击者当前没有护盾，则对攻击者施加 1 层毒。 |
| `enemy_vulture_predation_atk_plus_30_if_player_hp_below_30` | `aura`, `before_damage` | 荒原尸鹫存活时，若任一存活我方角色当前 HP 比例低于 30%，荒原尸鹫当前伤害计算中的 ATK +30%。 |
| `enemy_arbalist_clear_aimed_target` | `after_damage` | 灰誓弩手使用「贯心弩矢」后，清除由该弩手施加的瞄准标记。 |
| `enemy_hollow_sentinel_clear_self_shield_after_damage` | `after_damage` | 空甲守卫「盾击」结算后，清除自身全部护盾。 |
| `enemy_void_wolf_bite_bleed_or_double_if_unshielded` | `after_damage` | 虚空狼系「撕咬」命中后，若目标在本段伤害结算后没有护盾：目标无流血时施加 2 层流血；目标已有流血时将当前流血层数翻倍。 |
| `enemy_void_wolf_claw_bleed_and_extra_hit_if_target_bleeding` | `after_damage` | 虚空狼系「裂爪」每个命中段后，若目标没有护盾则施加 1 层流血；该行动两段伤害结算后，若目标存在流血，追加 1 次 100% ATK 物理伤害，追加命中后若目标没有护盾再施加 1 层流血。 |
| `enemy_void_wolf_king_atk_plus_10_percent_per_bleeding_player` | `aura`, `before_damage` | 虚空狼王存活时，每有 1 名存活我方角色带有流血，虚空狼王当前伤害计算中的 ATK +10%。 |
| `enemy_ashen_commander_aura_other_allies_atk_plus_20_def_plus_2` | `aura` | 骑士统领存活时，其它存活敌方单位 ATK +20%、DEF +2；统领死亡后该光环失效。 |
| `enemy_ashen_commander_intercept_50_percent` | `before_targeted` | 骑士统领存活且其它敌方单位将被单体攻击指定为目标时，50% 概率将目标改为骑士统领。 |
| `enemy_bulwark_def_plus_10_while_shielded` | `aura` | 缚誓壁垒当前护盾大于 0 时自身 DEF +10；护盾为 0 时该加成失效。 |
| `enemy_apply_cannot_act_1_on_self_shield_break` | `on_shield_break` | 触发者护盾从大于 0 变为 0 时，触发者获得 1 回合无法行动状态。 |
| `enemy_binder_soul_echo_ap_plus_30_on_ally_death` | `on_ally_death` | 钟楼缚魂师存活时，每有 1 个其它敌方单位死亡，自身 AP +30%，持续本场战斗。 |
| `enemy_binder_redirect_target_to_random_ally_50_percent` | `before_targeted` | 钟楼缚魂师被单体攻击指定为目标时，若存在其它存活敌方单位，50% 概率将目标改为随机其它敌方单位。 |
| `enemy_binder_siphon_kill_random_ally_heal_self_50_max_hp` | `effect_resolution` | 钟楼缚魂师「汲魂」杀死 1 名随机其它敌方单位，并治疗自身 50% 最大 HP；若没有其它存活敌方单位，该效果不执行。 |
| `enemy_leader_battle_start_mark_shakuya_3_vulnerable_3` | `battle_start` | 癫狂的强盗首领战斗开始时，若灼夜在我方上场阵容中，对灼夜施加 3 回合猎杀标记和 3 层脆弱。 |
| `enemy_leader_debuff_detonation_damage_and_consume_stacks` | `effect_resolution` | 癫狂的强盗首领「终极引爆」对全体我方分别读取脆弱、虚弱、流血和中毒层数总和，造成 `层数总和 × 100% ATK` 物理伤害；结算后消耗参与计算的这些层数。 |

### 敌人自定义条件与目标选择

| id | 类型 | 语义 |
|----|------|------|
| `attacker` | targetSelector.customId | 当前触发上下文中的攻击者。 |
| `lowest_hp_ratio_enemy` | targetSelector.customId | 从敌人视角选择当前 HP 比例最低的存活我方角色；并列时按存在感权重抽取。 |
| `aimed_target_by_self` | targetSelector.customId | 选择由当前来源敌人施加了瞄准标记的存活我方角色；没有合法目标时该行动不可用。 |
| `random_enemy_ally` | targetSelector.customId | 从当前来源敌人的其它存活友方单位中随机选择 1 个。 |
| `character_shakuya` | targetSelector.customId | 选择上场阵容中的灼夜；若灼夜未上场，该目标选择为空。 |
| `aimed_target_by_self_exists` | condition.custom value | 当前来源敌人存在自己施加的瞄准目标。 |
| `self_has_shield` | condition.custom value | 当前来源单位护盾大于 0。 |
| `self_shield_zero` | condition.custom value | 当前来源单位护盾等于 0。 |
| `target_is_not_self` | condition.custom value | 当前效果解析目标不是来源单位自身。 |
| `turn_number_eq_1` | condition.custom value | 当前战斗回合数等于 1。 |
| `turn_number_gte_2_and_no_marked_player` | condition.custom value | 当前战斗回合数大于等于 2，且场上没有由当前来源关注的存活标记目标。 |
| `marked_enemy_alive` | condition.custom value | 当前来源关注的标记目标仍然存活。 |

## 装备被动

| `customId` | 窗口 | 语义 |
|------------|------|------|
| `attacks_cannot_be_evaded` | `before_damage` | 装备者的攻击跳过命中/闪避失败结果，视为命中；仍可暴击。 |
| `gain_1_energy_on_critical_hit` | `on_crit` | 装备者造成暴击后，玩家公共能量 +1，不超过能量上限。 |
| `basic_attack_lifesteal_50` | `after_basic_attack` | 装备者普攻按实际 HP 伤害的 50% 治疗自己。 |
| `battle_start_current_hp_minus_30_percent` | `battle_start` | 装备者战斗开始时当前 HP 减少最大 HP 的 30%，最低保留 1 HP。 |
| `all_attacks_apply_1_poison` | `on_hit` | 装备者普攻和伤害卡牌每个命中段对命中目标施加 1 层毒。 |
| `hit_damage_plus_30_miss_stun_self` | `before_damage`, `on_miss` | 装备者命中时该段伤害 +30%；未命中时装备者获得 1 回合无法行动状态。 |
| `draw_one_less_card_each_player_turn` | `turn_start` | 玩家回合抽牌数量 -1，最低为 0。 |
| `atk_bonus_from_max_hp_15_percent` | `install`, `uninstall` | 装备者 ATK 增加最大 HP 的 15%，随最大 HP 变化重新计算。 |
| `atk_and_ap_use_higher_value` | `install`, `uninstall` | 装备者读取 ATK 或 AP 时，两者均按当前较高值参与公式计算。 |
| `mixed_damage_bonus_50_percent_other_attack_stat` | `before_damage` | 装备者物理伤害额外附带 50% AP 伤害；法术伤害额外附带 50% ATK 伤害。 |
| `critical_hit_lifesteal_15` | `on_crit` | 装备者暴击造成实际 HP 伤害后，按该 HP 伤害的 15% 治疗自己。 |
| `shield_gain_plus_50_percent_and_critical_damage_taken_double` | `on_shield_gain`, `before_damage` | 装备者获得护盾量 +50%；装备者受到暴击伤害时最终伤害 ×2。 |
| `owner_kill_grants_run_atk_plus_3` | `on_kill` | 装备者击杀敌人后，本 run 装备者 ATK 永久 +3。 |
| `turn_end_all_allies_heal_20_plus_ap_20_percent` | `turn_end` | 玩家回合结束时，全队治疗 `20 + 装备者 AP × 20%`。 |
| `heal_effect_plus_30_percent` | `on_heal` | 装备者治疗卡牌产生的理论治疗量 +30%。 |
| `if_unhurt_this_turn_next_turn_atk_ap_plus_30_percent` | `turn_end`, `turn_start`, `on_damaged` | 若装备者本方回合内未受伤，则下一个本方回合 ATK/AP +30%，持续 1 回合。 |
| `basic_attack_combo_damage_plus_20_percent_stackable` | `before_basic_attack`, `on_hit` | 装备者普攻命中后，下一次普攻伤害 +20%，可叠加；本场战斗结束清空。 |
| `third_and_later_basic_attack_applies_1_bleed` | `on_hit` | 装备者本回合第 3 次及之后普攻命中时，若目标在本次普攻伤害结算后没有护盾，则施加 1 层流血。 |
| `damage_plus_200_percent_against_enemy_hp_below_20` | `before_damage` | 装备者对当前 HP 比例低于 20% 的敌人造成的伤害 +200%。 |
| `overflow_energy_each_point_grants_ap_plus_10_percent_this_turn` | `turn_start`, `before_energy_gain` | 玩家能量将超过上限时，每溢出 1 点，本回合装备者 AP +10%。 |
| `atk_plus_8_percent_per_10_percent_missing_hp` | `before_damage` | 装备者每损失 10% 最大 HP，当前伤害计算中的 ATK +8%。 |
| `first_damage_to_full_hp_enemy_plus_100_percent` | `before_damage` | 装备者对满血敌人的第一次伤害 +100%；每个敌方单位每场最多触发一次。 |
| `when_damaged_apply_vulnerable_to_attacker_2x_chapter` | `on_damaged` | 装备者受到攻击伤害后，对攻击者施加 `2 × 当前章节数` 层脆弱。 |
| `gold_reward_plus_50_percent` | `before_reward` | 装备者在队伍中时，战斗金币奖励 +50%，向下取整。 |
| `each_exhausted_card_grants_battle_atk_ap_plus_5` | `on_card_exhausted` | 装备者每有一张牌被消耗，本场战斗 ATK/AP +5。 |
| `basic_attack_draw_1_card_and_play_card_triggers_free_basic_attack` | `after_basic_attack`, `after_card_play` | 装备者普攻后抽 1 牌；装备者出牌后对同一主目标免费普攻 1 次，该免费普攻不抽牌但可回能量。 |
| `basic_attack_splash_50_percent_ignore_defense_and_evasion` | `after_basic_attack` | 装备者普攻命中后，对其它敌人造成本次最终伤害 50% 的固定溅射，忽略 DEF 和闪避。 |
| `basic_attack_ignore_evasion` | `before_basic_attack` | 装备者普攻跳过闪避失败，视为命中；仍可暴击。 |
| `draw_2_extra_each_turn_choose_keep` | `turn_start` | 玩家回合抽牌时额外查看 2 张牌，玩家选择保留哪些，未保留牌进入弃牌堆。 |
| `bind_ally_shared_hp_pool_atk_ap_take_higher` | `install`, `battle_start`, `uninstall` | 装备时选择一名队友绑定；战斗中双方共享 HP 池，ATK/AP 均取双方较高值。 |
| `damage_delayed_one_turn_plus_50_ignore_all_defense` | `before_damage`, `turn_start` | 装备者造成的伤害不立即扣除，记录为延迟伤害；目标下个回合开始时结算 `记录伤害 ×150%`，无视防御与护盾。 |
| `once_per_battle_undying_one_turn_on_hp_zero` | `before_death`, `turn_end` | 装备者每场战斗首次 HP 归零时进入不灭 1 回合：HP 保持 1，免伤，ATK+100%，受到攻击时反击普攻。 |
| `damage_uses_higher_atk_or_ap_result` | `before_damage` | 装备者伤害公式分别按 ATK、AP 主要缩放计算，取较高结果。 |
| `damage_type_all_stat_use_higher_scaling` | `before_damage` | 装备者卡牌伤害类型视为全属性，按 ATK/AP 缩放较高结果结算。 |
| `each_exhausted_card_gain_1_energy_draw_1_and_fifth_exhausted_deals_300_atk_aoe` | `on_card_exhausted` | 装备者每消耗 1 张牌，获得 1 能量并抽 1 牌；本场第 5 张消耗牌时，对全体敌人造成 300% ATK 伤害。 |
| `owner_kill_absorbs_50_percent_target_stats_this_battle` | `on_kill` | 装备者击杀敌人后，吸收目标 ATK/AP/DEF/CRIT/CDMG/最大 HP 的 50%，持续本场战斗。 |

## 诅咒装备被动

| `customId` | 窗口 | 语义 |
|------------|------|------|
| `blood_sacrifice_other_allies_current_hp_50_percent_to_owner_atk_ap_and_heal` | `turn_start` | 装备者回合开始时，其它队员各损失 50% 当前 HP；装备者获得等同总血祭 HP 50% 的 ATK/AP 加成，并治疗总血祭 HP 10%。每次血祭后获得 1 张 `curse_blood_price`。 |
| `damage_splash_30_percent_to_all_others_add_out_of_control_and_exhaust_one_draw_pile_card` | `after_damage` | 装备者每次造成伤害时，对除主目标外的所有单位造成 30% 溅射；加入 1 张 `curse_out_of_control` 到牌库，并消耗牌库中 1 张牌。 |
| `play_card_lose_20_percent_current_hp_every_3_cards_add_void_pulse_to_hand` | `after_card_play` | 装备者每次出牌损失 20% 当前 HP；每打出 3 张牌，将 1 张 `curse_void_pulse` 加入手牌。 |
| `turn_start_lose_30_percent_current_hp_kill_add_bloodlust_impulse` | `turn_start`, `on_kill` | 装备者每回合开始损失 30% 当前 HP；击杀敌人后获得 1 张临时 `curse_bloodlust_impulse`，本回合未打出则消失。 |
| `attacks_50_percent_chance_target_random_ally_when_not_solo` | `before_damage` | 装备者普攻和伤害牌有 50% 概率改为攻击随机队友；若队伍只有装备者存活则不触发。 |
| `turn_start_reduce_max_hp_20_percent_kill_gain_enemy_hp_30_percent_and_blood_lead` | `turn_start`, `on_kill` | 装备者每回合最大 HP 上限 -20%；击杀敌人后获得目标 30% 最大 HP 上限并治疗等量 HP，同时获得 1 张 `curse_blood_lead`。 |
| `other_allies_def_minus_100` | `install`, `uninstall` | 装备者以外的队友 DEF -100；装备卸下后移除。 |
| `turn_start_add_hunger_kill_heal_enemy_max_hp_and_gain_enemy_atk_ap_20_percent` | `turn_start`, `on_kill` | 装备者每回合向手牌添加 1 张 `curse_hunger`；击杀后回复目标 100% 最大 HP，溢出转护盾，并获得目标 20% ATK/AP，本场可叠加，消除手中 2 张 `curse_hunger`。 |
| `damage_ignore_defense_def_set_0_cannot_gain_shield_exhaust_card_add_void_echo` | `install`, `before_damage`, `on_shield_gain`, `on_card_exhausted` | 装备者伤害无视 DEF；自身 DEF 固定为 0，无法获得护盾；自身所有卡牌变为消耗型；消耗除 `curse_void_echo` 外的卡牌时获得 1 张 `curse_void_echo`。 |
| `each_action_damages_random_ally_max_hp_20_gain_sacrifice_on_kill_or_ally_death` | `after_action`, `on_kill`, `on_death` | 装备者每次行动扣除随机队友 20% 最大 HP；每次击杀或有队友因装备者原因死亡时获得 1 层祭品（ATK/AP+30%）。每 10 层获得 1 张 `curse_evil_god_gaze`。 |
| `battle_start_roll_one_of_ten_fate_roulette_effects` | `battle_start` | 每场战斗开始随机 10 种命运效果之一：ATK翻倍、AP翻倍、AS+3、HP减半、无法出牌、伤害+100%、ATK=0、AP=0、AS=0、普攻秒杀且 AS=1。 |

## 装备牌和诅咒牌主动效果

| `customId` | 窗口 | 语义 |
|------------|------|------|
| `set_self_current_hp_to_1` | `effect_resolution` | 当前来源单位 HP 设为 1，不触发死亡。 |
| `gamble_strike_200_atk_50_double_50_self_hit` | `effect_resolution` | 对选中目标造成 200% ATK 物理伤害；50% 概率该伤害翻倍，50% 概率改为打自己。两次判定使用战斗 RNG。 |
| `next_player_turn_cannot_play_cards` | `effect_resolution`, `turn_start` | 给玩家阵营添加延迟状态：下一个玩家回合不能出牌。 |
| `all_attacks_guaranteed_crit_this_turn` | `effect_resolution` | 来源单位本回合所有攻击动作命中后必定暴击。 |
| `self_cannot_be_targeted_by_enemy_single_target_attacks_this_turn` | `effect_resolution` | 来源单位本回合不能被敌方单体攻击或单体技能选择为目标；全体和随机目标不受影响。 |
| `spend_all_current_energy` | `effect_resolution` | 当前玩家公共能量清零。 |
| `on_battle_victory_gain_gold_15x_chapter` | `effect_resolution`, `after_reward` | 若本场战斗胜利，额外获得 `15 × 当前章节数` 金币。 |
| `exhaust_another_hand_card` | `effect_resolution` | 玩家从手牌中选择当前卡以外的 1 张牌，将其移入消耗堆。 |
| `self_cannot_basic_attack_this_turn` | `effect_resolution` | 来源单位本回合不能再普攻。 |
| `look_top_5_deck_choose_2_to_hand_rest_bottom` | `effect_resolution` | 查看牌库顶 5 张，选择 2 张加入手牌，其余按玩家选择顺序放到牌库底。 |
| `swap_hp_ratio_with_bound_ally_one_turn_cooldown` | `effect_resolution` | 与绑定队友互换当前 HP 百分比；该卡进入 1 回合冷却。 |
| `resolve_all_delayed_damage_double` | `effect_resolution` | 立即结算所有由装备者造成并仍未结算的延迟伤害，结算值 ×2。 |
| `exhaust_all_other_hand_cards_gain_1_energy_and_draw_1_per_card` | `effect_resolution` | 消耗手中所有其它牌；每消耗 1 张，获得 1 能量并抽 1 牌。 |
| `absorb_killed_target_100_percent_stats_this_battle` | `on_kill` | 来源动作击杀目标时，来源单位吸收目标 ATK/AP/DEF/CRIT/CDMG/最大 HP 的 100%，持续本场战斗。 |
| `unplayable_in_hand_owner_loses_10_percent_max_hp_each_turn` | `on_hand_state`, `turn_start` | 该牌在手中不可打出；其拥有者每个己方回合开始损失 10% 最大 HP。多张按各自独立扣除。 |
| `unplayable_in_hand_max_energy_minus_1_stackable` | `on_hand_state` | 该牌在手中不可打出；玩家公共能量上限 -1，可叠加；离开手牌时恢复。 |
| `void_pulse_damage_atk_ap_10_percent_times_cards_played_this_battle` | `effect_resolution` | 对全体敌人造成 `(10% AP + 10% ATK) × 本场已出牌数` 的法术伤害。 |
| `self_all_damage_plus_30_percent_this_turn` | `effect_resolution` | 来源单位本回合所有伤害 +30%。 |
| `unplayable_in_hand_all_card_cost_plus_1` | `on_hand_state` | 该牌在手中不可打出；所有手牌费用 +1，可叠加；离开手牌时恢复。 |
| `random_card_from_global_card_pool_to_hand` | `effect_resolution` | 从全游戏卡牌池随机生成 1 张临时卡加入手牌，包含角色牌、装备牌和药水牌。 |
| `fate_coin_50_draw_3_50_discard_all_hand` | `effect_resolution` | 50% 概率抽 3 张牌；50% 概率弃掉全部手牌。使用战斗 RNG。 |

## 药水特殊效果

| `customId` | 窗口 | 语义 |
|------------|------|------|
| `battle_start_heal_owner_5_percent_max_hp` | `battle_start` | 药水目标角色每场战斗开始时治疗自身 5% 最大 HP。 |
| `selected_character_basic_attack_lifesteal_20` | `after_basic_attack` | 药水目标角色普攻按实际 HP 伤害的 20% 治疗自己。 |
| `selected_character_draw_1_on_critical_hit` | `on_crit` | 药水目标角色造成暴击后抽 1 张牌。 |
| `selected_character_basic_attack_hits_2_each_60_percent_damage_one_energy_total` | `before_basic_attack` | 药水目标角色普攻变为 2 段，每段 60% ATK 伤害；仍只消耗 1 AS 并只回复 1 能量。 |
| `shield_card_also_grants_all_allies_20_percent_same_shield` | `on_shield_gain` | 药水目标角色的护盾卡对主目标施加护盾后，额外给全队施加本次护盾量 20% 的护盾。 |
| `selected_character_kill_deals_30_percent_killed_target_max_hp_to_random_enemy` | `on_kill` | 药水目标角色击杀敌人时，对随机存活敌人造成被击杀目标 30% 最大 HP 的固定伤害。 |
| `selected_character_non_exhaust_card_effects_double` | `before_card_play` | 药水目标角色原本非消耗型卡牌被改为消耗型后，其效果数值翻倍。只作用于该药水改造过的卡牌。 |
| `selected_character_poison_stacks_applied_double` | `before_status_apply` | 药水目标角色施加毒层时，最终施加层数 ×2。 |
| `next_card_damage_plus_50_percent` | `effect_resolution`, `before_damage` | 来源单位下一张伤害型卡牌的所有伤害 +50%，触发后移除。 |
| `return_random_card_from_exhaust_to_hand` | `effect_resolution` | 从消耗堆随机选择 1 张牌返回手牌；若手牌满仍返回并记录日志。 |

## 节点修正和 run 特殊效果

| `customId` | 窗口 | 语义 |
|------------|------|------|
| `next_battle_victory_double_gold_growth_choices_and_equipment_drops` | `before_reward` | 下场战斗胜利奖励中，金币 ×2、属性成长奖励发放 2 次、装备掉落发放 2 次。触发后移除。 |
| `gain_gold_20_plus_10x_chapter` | `effect_resolution` | 立即获得 `20 + 10 × 当前章节数` 金币。 |
| `next_battle_initial_draw_plus_2` | `battle_start` | 下场战斗起手抽牌数量 +2。触发后移除。 |
| `last_stand_all_units_start_full_energy_killer_crit_plus_10_cdmg_plus_20_missing_hp_atk_plus_0_5_percent` | `battle_start`, `on_kill`, `on_damaged`, `on_heal`, `before_damage` | 下场战斗开始时，玩家公共能量回复至能量上限。敌我全体获得“血战到底”。任意单位击杀 1 个敌对单位后，只有击杀者自身获得暴击率 +10%、暴击伤害 +20%，本场可叠加；敌方击杀我方召唤物或角色时，也只强化完成击杀的敌方单位。每个单位按自身当前失血比例获得 ATK 百分比加成：`ATK加成% = (1 - 当前HP / 最大HP) × 100 × 0.5`，随 HP 变化动态重算。战斗结束后移除。 |
| `next_node_selection_option_count_plus_1` | `node_option_generation` | 下一次节点候选数量 +1。触发后移除。 |
| `dispatch_characters_to_block_pursuers_next_node_unavailable` | `node_complete` | 立即要求派遣角色阻拦追兵；当前可用角色 ≤3 时派 1 人，≥4 时派 2 人。被派遣角色下一节点不可上场。 |
| `next_node_selection_only_combat_options` | `node_option_generation` | 下一次候选节点只从普通战、精英战、特殊精英中生成；固定步仍优先。触发后移除。 |
| `player_damage_taken_plus_30_percent_next_battle` | `before_damage` | 下场战斗中我方受到所有伤害 +30%。战斗结束后移除。 |
| `after_player_card_play_30_percent_exhaust_random_hand_card` | `after_card_play` | 下场战斗中玩家每次出牌后，30% 概率随机消耗 1 张其它手牌。战斗结束后移除。 |
| `player_damage_taken_plus_20_percent_each_battle` | `before_damage` | 持续期间我方受到所有伤害 +20%；击败 Boss 或商店净化后移除。 |
| `disable_random_equipment_on_random_character` | `effect_resolution`, `shop_service` | 随机选择 1 名角色的 1 件装备设为失效；商店修理后恢复。 |
| `mark_random_character_death_after_10_nodes` | `node_complete`, `shop_service` | 随机标记 1 名角色。之后每完成 1 个节点计数 +1，达到 10 后该角色本 run 死亡不可上场；商店可净化或复活。 |
| `random_character_atk_ap_minus_20_percent_until_5_battles_or_shop` | `battle_start`, `battle_end`, `shop_service` | 随机角色 ATK/AP -20%；经过 5 场战斗或商店治疗后移除。 |
| `next_node_selection_blind_all_information` | `node_option_generation` | 下一次候选节点隐藏所有类型提示、修正描述和风险收益信息。触发后移除。 |
| `player_allies_lose_5_percent_max_hp_each_turn_next_battle` | `turn_start` | 下场战斗中，我方全员每个己方回合开始损失 5% 最大 HP。战斗结束后移除。 |
| `enemy_count_multiplier_3` | `node_enter` | 当前战斗 encounter 中敌人数量 ×3。Boss 不受影响，除非 encounter 明确允许。 |
| `enemy_count_multiplier_5` | `node_enter` | 当前战斗 encounter 中敌人数量 ×5。Boss 和特殊剧情战不受影响，除非 encounter 明确允许。 |
| `equipment_and_gold_drops_double` | `before_reward` | 当前精英战装备掉落和金币奖励翻倍。 |
| `enemy_all_units_heal_3_percent_max_hp_each_turn` | `turn_end` | 敌方每个敌方回合结束时，全体存活敌人治疗 3% 最大 HP。 |
| `enemy_highest_spd_gets_first_strike_player_first_turn_cannot_act` | `battle_start`, `turn_start` | 强制敌方最高 SPD 单位获得抢攻；玩家第 1 回合不能普攻和出牌。 |
| `player_first_turn_cannot_act` | `turn_start` | 玩家第 1 回合不能普攻和出牌。 |
| `initial_energy_zero_first_turn_basic_attack_no_energy_gain` | `battle_start`, `after_basic_attack` | 战斗初始能量为 0；玩家第 1 回合普攻不回复能量。 |
| `player_energy_recovery_plus_5_each_turn` | `turn_start` | 玩家每回合能量恢复额外 +5，不超过上限。 |
| `player_energy_recovers_to_max_each_turn` | `turn_start` | 玩家每回合开始时能量直接恢复至上限。 |
| `both_sides_can_only_basic_attack_no_cards_no_enemy_skills` | `before_card_play`, `enemy_action` | 玩家不能出牌；敌方不能使用技能，只能普攻。 |
| `all_damage_plus_50_percent_all_healing_plus_50_percent` | `before_damage`, `on_heal` | 全场所有伤害 +50%，所有理论治疗量 +50%。 |
| `after_card_or_skill_use_user_takes_fixed_damage_5x_chapter` | `after_card_play`, `after_action` | 任意单位使用卡牌或技能后，使用者受到 `5 × 当前章节数` 固定伤害。普攻不触发，除非技能定义标记为卡牌/技能。 |
| `on_any_unit_death_deal_fixed_damage_10x_chapter_to_all_units` | `on_death` | 任意单位死亡时，对全场所有存活单位造成 `10 × 当前章节数` 固定伤害。该伤害导致死亡时可继续触发，但必须有触发链深度上限。 |
| `enemy_copies_player_highest_hp_atk_ap_def_values` | `battle_start` | 敌方所有单位的 HP/ATK/AP/DEF 取我方上场角色对应属性最高值。当前 HP 同步为新最大 HP。 |

## 校验要求

实现数据加载器时必须校验：

- 所有 `customId` 都能在 handler registry 中找到。
- handler registry 中的 `customId` 必须在本文登记；测试或 debug handler 不得混入正式内容表。
- 同一 `customId` 多处使用时语义必须一致，不能根据来源装备或卡牌改变含义。
- 所有 custom handler 的随机分支必须使用 RNG 并写入日志。
- 会添加卡牌的 handler 必须引用存在的卡牌 id。
- 会修改奖励、候选节点或商店商品的 handler 必须把生成结果写入 run 存档，读档不得重新随机。

## 实现优先级

MVP 实现建议按以下顺序处理：

1. 先实现第 1 章节点修正常用 handler：候选数量、只出战斗、怪物数量、能量恢复、伤害/治疗倍率、首回合限制。
2. 再实现第 1 章会实际掉落的普通/稀有/史诗装备 handler。
3. Boss 奖励允许选诅咒装备，因此诅咒装备 handler 可以先做最小可运行版本，但必须在 UI 明确展示负面效果。
4. 传说装备和第 2-5 章相关 handler 可以延后，数据已登记，未开放内容不得进入实际掉落池。
