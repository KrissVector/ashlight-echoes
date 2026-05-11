# Ashlight Echoes / 辉烬残响

《Ashlight Echoes》是一款 2D 二次元风格 roguelike 卡牌构筑游戏。玩家不是世界内的主角，而是以策略指挥视角组织队伍、选择遭遇、构筑共享牌库，并带领角色推进章节。

本仓库是重构后的正式项目文档库。目前只迁移设计文档、概念图和数据草案；旧原型代码不在本项目中，后续技术架构应基于这里的新版规格重新实现。

## 设计入口

- [设计总览](docs/00-overview.md)
- [世界观设定](docs/01-world/world-setting.md)
- [战斗系统](docs/02-systems/battle.md)
- [卡牌系统](docs/02-systems/cards.md)
- [肉鸽流程与奖励](docs/02-systems/roguelike-run.md)
- [地图与遭遇](docs/02-systems/map-encounters.md)
- [美术风格指南](docs/04-art/style-guide.md)
- [数据草案](docs/05-data-drafts/README.md)
- [实现规格](docs/06-implementation-spec/README.md)

## 迁移说明

- 旧原型源码刻意不迁移，避免旧架构和旧规则污染新版实现。
- 旧技术文档刻意不迁移，技术方案以后按新版设计规格重新定义。
- `docs/05-data-drafts/` 只作为迁移参考，不等于最终 runtime schema。
- 正式设计文档使用“角色 / 队伍成员”表述；本作没有世界内的玩家主角，也不是下属随从叙事。
