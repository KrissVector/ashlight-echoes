# 数据草案

本目录中的 JSON 文件是从旧项目迁移来的设计数据草案，不是最终 runtime 资源。

当前包含：

- `characters.json`：当前角色属性和初始卡牌草案。
- `cards.json`：当前卡牌和升级草案。
- `bonds.json`：早期双人联动草案。
- `rewards.json`：旧奖励池原型数据；实现前必须对照 `docs/02-systems/roguelike-run.md`。
- `shop.json`：旧商店和药水原型数据；实现前必须对照 `docs/02-systems/potions.md` 和经济规则。
- `maps.json`：旧项目中刻意留空；仅提醒地图 / 遭遇数据必须按步进式遭遇模式重新设计。

不要把这些文件当作权威实现 schema。权威规则以 `docs/02-systems/` 和后续 `docs/06-implementation-spec/` 为准。
