# Ashlight Echoes - Design Overview

> Status: formal project design baseline  
> Source: migrated from the MyGame prototype project  
> Migration scope: design documents, concept art, and configuration drafts only

## Project Positioning

Ashlight Echoes is a 2D anime-style roguelike deckbuilder. The player does not exist as an in-world protagonist; the player acts from a strategy/director view, building a team from the cast and making combat and route decisions.

The game is set on Hui Jin / Ashlight, a continent once nourished by Lumina. Two hundred years after the Starfall shattered the Celestial Core, Void power continues to corrupt ruins, creatures, and old guardians. Characters from different regions gather for their own reasons to seek Lumina Shards and resist the Void.

## Current Naming

- Chinese working title: 辉烬残响
- English working title: Ashlight Echoes
- World name in legacy docs: 辉烬大陆 / Luminash

`Luminash` may remain as an internal world-name candidate, but the project title uses `Ashlight Echoes` for readability.

## Design Pillars

1. Character-driven ensemble narrative, with no player avatar or singular chosen-one protagonist.
2. Roguelike deckbuilding built around team composition, card play, equipment, potions, and per-run growth.
3. Step-based encounter selection: each step offers two or three imperfect choices instead of a full Slay-the-Spire-style node map.
4. Each battle starts with HP restored; long-term pressure comes from environment modifiers, rewards, economy, and build decisions.
5. Anime fantasy presentation with warm character interaction over a ruined, post-calamity world.

## Core Loop

```text
Title / preparation
  -> choose team members
  -> choose next encounter option
  -> apply node modifiers
  -> battle / shop / event
  -> reward, potion use, equipment decisions
  -> repeat until chapter boss
  -> chapter clear, unlocks, or run failure
```

## Canonical Design Documents

World and narrative:

- [World Setting](01-world/world-setting.md)

Systems:

- [Battle](02-systems/battle.md)
- [Cards](02-systems/cards.md)
- [Run and Rewards](02-systems/roguelike-run.md)
- [Map and Encounters](02-systems/map-encounters.md)
- [Node Modifiers](02-systems/node-modifiers.md)
- [Equipment](02-systems/equipment.md)
- [Potions](02-systems/potions.md)

Content:

- [Characters](03-content/characters.md)
- [Chapter 1 Enemies](03-content/enemies-chapter-1.md)
- [Build Archetypes](03-content/build-archetypes.md)

Art and data:

- [Art Style Guide](04-art/style-guide.md)
- [Concept Art Index](04-art/concept-index.md)
- [Data Drafts](05-data-drafts/README.md)

## Migration Decisions

The old prototype implementation is intentionally excluded from this repository. It was built against older assumptions, including a full node-map flow and old runtime structures. If code is reused later, treat it as reference material only and port individual algorithms after checking them against the documents above.

Old placeholder documents were also excluded when they had been superseded by newer detailed documents. In particular, the old combined item-system placeholder is superseded by separate equipment and potion documents.

## Terminology Note

Formal design documents use playable characters / team members. Some migrated data drafts and legacy file names may still reflect old prototype terminology, but there is no in-world player protagonist commanding a subordinate cast.
