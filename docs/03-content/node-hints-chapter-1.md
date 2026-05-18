# 第 1 章节点提示文案

> 状态：正式内容基线
> 适用范围：第 1 章「银誓余烬 / 银誓要塞遗址」候选节点基础描述与节点自带修正文案
> 机器内容表：`data/node-hints.json`

## 生成原则

候选节点完整展示文案由两段组成：

```text
【节点基础描述】，【节点自带修正文案】
```

其中：

- 节点基础描述只暗示前方地点、气氛和可能内容，不直接显示 `nodeType`。
- 节点自带修正文案来自已有 `node_option` 节点自带修正，只描述玩家选择前可见的风险、收益或环境变化。
- 节点结束后随机事件 `post_node_random` 不参与候选节点描述；它只在节点完成后概率触发。

节点生成器必须先按 `run-flow.md` 的现有规则确定具体节点内容，再从该内容绑定的基础描述列表中随机选 1 条作为 `hintText`。基础描述不反向决定节点类型，也不改变节点类型权重、固定步或硬规则。

除固定剧情分支和章节 Boss 外，每条基础描述必须至少绑定 2 个节点，避免单个普通节点被独有描述直接暴露。固定剧情分支和章节 Boss 使用专属描述，不参与该校验。

## 固定专属描述

这些描述只用于固定剧情分支或章节 Boss。

| 节点 | 描述 |
|------|------|
| 第 1 章步 6「血战到底！」 | 血旗残党已经追上来，灼夜的护身符在混乱中发出灼热回响。 |
| 第 1 章步 6「突出重围」 | 后方追兵逼近，前方残墙只留下一条勉强突围的空隙。 |
| 第 1 章 Boss「断誓之盾·莱恩哈特」 | 要塞最深处传来沉重的盾击声，银白残光在断裂王座前凝成最后的守门人。 |

## 共享基础描述

| id | 描述 |
|----|------|
| `hint_ch1_outer_ruins_shadows` | 要塞外缘的灰白废墟里，有几处影子在缓慢游荡。 |
| `hint_ch1_broken_wall_carrion` | 破碎城墙外传来腐肉拖行和低哑啃咬声。 |
| `hint_ch1_circling_vultures` | 空中有黑影盘旋，地面也留下被啃食过的痕迹。 |
| `hint_ch1_rift_beast_growl` | 地下裂隙附近传来压低的兽吼，紫黑色雾气贴着地面流动。 |
| `hint_ch1_rift_claw_marks` | 裂隙边缘布满爪痕和拖拽痕，像有什么东西刚刚经过。 |
| `hint_ch1_plague_mist_steps` | 废墟低处涌出腥甜的疫雾，里面偶尔传来湿重脚步声。 |
| `hint_ch1_corrupted_tunnel_glow` | 被污染的坑道里亮着零星紫光，空气里混着腐败和星辉残味。 |
| `hint_ch1_training_ground_lines` | 前方似乎是昔日的新人骑士训练场，地面还残留着步法刻线。 |
| `hint_ch1_training_flags_armor` | 银白色训练旗倒在废墟边缘，远处传来整齐的甲胄声。 |
| `hint_ch1_wall_crossbow_sounds` | 城墙方向传来弩机拉弦声，残破垛口后有银光闪动。 |
| `hint_ch1_old_formation` | 断墙后的旧阵地仍像在执行命令，盾牌和弩线交错成残阵。 |
| `hint_ch1_armory_rust` | 前方似乎是旧军械库的残区，铁锈和封存药剂的气味混在一起。 |
| `hint_ch1_armory_shield_rack` | 半塌的库门后露出封存箱和旧盾架，金属拖地声从深处传来。 |
| `hint_ch1_bell_tower_echo` | 远处钟楼残影在雾中摇晃，低沉回声沿着石道传来。 |
| `hint_ch1_broken_stele_marks` | 不远处能看到一些碎裂的石碑遗迹，碑面仍有银色刻痕。 |
| `hint_ch1_oath_fragments` | 旧誓言的残句刻在倒塌石碑上，周围安静得不太自然。 |
| `hint_ch1_lamplight_or_lumina` | 废墟深处有微弱灯火，像补给点，也像某种残留星辉。 |
| `hint_ch1_lamplight_voices` | 前方隐约有灯火和低声人语，临时帐布在断墙后晃动。 |
| `hint_ch1_infirmary_supplies` | 破旧医帐和补给箱散落在路边，封条上还能看到银誓纹样。 |
| `hint_ch1_small_campfire` | 前方似乎是个小型营地，隐约可看见篝火和炊烟。 |
| `hint_ch1_old_waystation_tracks` | 前方似乎是个旧驿站，地上有着车辙的痕迹。 |
| `hint_ch1_wasteland_footsteps` | 荒原边缘传来急促脚步声，空中有黑影跟着盘旋。 |
| `hint_ch1_chase_traces` | 断枝和碎石间残留着追逐痕迹，前方像刚爆发过一场混乱。 |
| `hint_ch1_silver_light_path` | 要塞深处的银色残光凝成一条路，像在把人引向旧日誓约的遗留之处。 |
| `hint_ch1_ordered_guard_traces` | 前方的守卫痕迹变得异常整齐，像有人仍在维持骑士团最后的阵列。 |
| `hint_ch1_training_stakes` | 破损木桩沿训练线排开，像刚有人复盘过旧式守势。 |
| `hint_ch1_silver_lamp_device` | 银誓遗留的灯器还在发光，周围没有明显脚印。 |

## 内容绑定

具体绑定关系以 `data/node-hints.json` 为准。每个随机候选节点绑定 2-3 条共享基础描述；第 1 章步 6 固定分支和 Boss 各绑定 1 条专属描述。

实现层必须校验：

- `contentHints.encounters` 引用的 encounter id 存在。
- `contentHints.events` 引用的 event id 存在。
- `contentHints.shops` 引用的 shop id 存在。
- 除 `exclusive: true` 的固定剧情 / Boss 描述外，每条基础描述被至少 2 个内容节点引用。
- 每个第 1 章随机候选内容至少绑定 2 条基础描述。
- 每个 `node_option` 修正至少绑定 1 条节点自带修正文案。
