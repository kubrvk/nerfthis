# NERF THIS!

> **Roguelike Deckbuilder · Third-Person Action** — Unreal Engine 5 · C++ · Solo Development  
> [ArtStation](https://www.artstation.com/kubrik)

> [!WARNING]
> Nerf This! is currently in early development.
> Features, visuals, balance, and core systems are actively evolving.

---

## Overview

Nerf This! is a third-person action roguelike with a deckbuilding layer built on top of real-time combat. Every run drops the player into a procedurally shifted dungeon — encounters, rewards, and pathing change each time.

Cards are the core mechanic. They define abilities, modify attacks, and create synergies that compound across a run. A build can lean into aggressive combo chains, defensive control setups, or high-risk high-reward configurations — the direction is determined entirely by the cards picked and how they interact.

The design challenge is making real-time combat and turn-adjacent card logic feel like a single unified system rather than two separate games bolted together.

---

## Engine & Technical Stack

| Layer | Technology |
|---|---|
| Engine | Unreal Engine 5 |
| Primary Language | C++ (gameplay, AI, systems) |
| Scripting / Prototyping | Unreal Blueprint (visual scaffolding only) |
| Rendering | Lumen (GI), Nanite (static meshes), custom post-process materials |
| Physics | Chaos Physics |
| Platform | PC (Win64), Steam SDK |
| 3D Pipeline | ZBrush → Maya → Substance Painter → UE5 |
| Shader Authoring | UE Material Editor + HLSL custom nodes |

---

## Architecture Overview

```
NerfThis/
├── Source/
│   ├── Core/
│   │   ├── NerfCharacter.h/.cpp               # Base player class
│   │   ├── NerfGameMode.h/.cpp                # Run session, dungeon orchestration
│   │   └── NerfGameState.h/.cpp               # Shared run state
│   ├── Systems/
│   │   ├── CardSystem/                         # Deck, hand, play resolution, upgrades
│   │   ├── CombatSystem/                       # Attack chains, hit detection, stagger
│   │   ├── DungeonGen/                         # Room layout, encounter seeding, pathing
│   │   ├── RewardSystem/                       # Card offers, loot tables, shop
│   │   ├── AIController/                       # Enemy behaviors, state machines
│   │   ├── StatusEffects/                      # Buff/debuff stacks, tick resolution
│   │   ├── RunState/                           # Per-run persistence, meta progression
│   │   └── SaveSystem/                         # Run serialization, checkpoint logic
│   ├── UI/
│   │   ├── CardHUD/                            # Hand display, card preview, play feedback
│   │   ├── DungeonMap/                         # Room graph, current position, pathing UI
│   │   └── Menus/                              # Run start, deck review, meta upgrades
│   └── Tools/                                  # Editor utilities, card authoring tools
```

---

## Core Systems: Technical Detail

### 1. Card System

Cards are the primary expression of player agency — they modify, extend, and reshape combat behavior at runtime.

**Card Data Model:**

Each card is defined by a `UCardDataAsset` with a type tag, effect list, cost, rarity, and upgrade path:

| Card Type | Behavior |
|---|---|
| **Attack** | Deals damage; may apply status effects or trigger combo conditions |
| **Skill** | Non-damaging effect — reposition, buff, draw, resource generation |
| **Power** | Persistent passive; applies an ongoing modifier to combat rules |
| **Modifier** | Attaches to another card; alters its behavior when played together |

**Deck & Hand Management:**
- Player deck stored as `TArray<UCardDataAsset*>` on `UDeckComponent`.
- Draw and discard piles managed separately; shuffle triggered on draw pile exhaustion via `FRandomStream` seeded per-room.
- Hand size, draw count, and play limit are base stats modifiable by Power cards and run relics.

**Card Play Resolution:**
- Played card dispatches a `FCardEffectContext` struct to `UCardResolverComponent`.
- Resolver evaluates target, checks combo conditions, applies effects in declaration order, then fires `OnCardResolved` delegate consumed by combat and status systems.
- Combo detection: cards tag themselves with `FGameplayTag` combo markers; resolver checks active tags against incoming card's combo requirements before resolution.

```cpp
void UCardResolverComponent::PlayCard(UCardDataAsset* Card, AActor* Target)
{
    FCardEffectContext Context;
    Context.Source = CharacterOwner;
    Context.Target = Target;
    Context.ActiveComboTags = ActiveComboTags;

    for (UCardEffect* Effect : Card->Effects)
    {
        Effect->Resolve(Context);
    }

    ActiveComboTags.AppendTags(Card->GrantedComboTags);
    OnCardResolved.Broadcast(Card, Context);
}
```

**Card Upgrades:**
- Each card has one or more upgrade paths defined in its data asset.
- Upgrades are applied at rest sites or via specific reward events — `UCardUpgradeComponent` swaps the base asset reference and broadcasts `OnDeckChanged`.

---

### 2. Combat System

Real-time third-person combat runs in parallel with the card layer. Physical actions (movement, dodge, basic attack) are always available; cards extend and modify what those actions do.

**Base Combat:**
- Light and heavy attack chains authored as ordered `TArray<UAnimMontage*>` per weapon type.
- Input buffering within a configurable frame window queues next combo step.
- Chain advancement is event-driven via `AnimNotify_ComboWindow` / `AnimNotify_ComboEnd`.

**Card-Combat Integration:**
- Active Power cards can inject additional hit effects, change damage type, add status procs, or alter animation montage selection at combo decision points.
- Attack cards trigger as discrete ability activations layered on top of the base combo — they do not interrupt the physical chain.

**Stagger System:**
- `FStaggerState` struct on each actor tracks `CurrentStagger` against `MaxStagger`.
- Hits accumulate stagger; threshold breach triggers stagger animation and brief control interruption.
- Cards can modify stagger dealt, stagger resistance, or trigger effects on stagger break.

```cpp
void UCombatComponent::ApplyStagger(float Amount)
{
    StaggerState.CurrentStagger += Amount;
    if (StaggerState.CurrentStagger >= StaggerState.MaxStagger)
    {
        TriggerStaggerAnimation();
        StaggerState.CurrentStagger = 0.f;
        OnStaggerBreak.Broadcast();
    }
}
```

**Status Effects:**
- Effects (Burn, Bleed, Weaken, Shield, etc.) implemented as `UStatusEffectBase` subclasses with stack count and tick behavior.
- `UStatusEffectComponent` owns the active effect map; tick resolution fires per-effect on a shared timer rather than per-actor per-frame.
- Cards reference status effects by `FGameplayTag`; binding is resolved at runtime against the effect registry.

---

### 3. Dungeon Generation

Each run generates a dungeon layout from a seeded graph system.

- Run initialized with `int32 RunSeed` at `GameMode::BeginPlay`.
- Room graph generated as a directed acyclic graph — branching paths, merge nodes, and a guaranteed boss terminal.
- Each node is typed (`ECombatRoom`, `EEliteRoom`, `ERestSite`, `EShop`, `ETreasure`, `EBoss`) weighted by depth and prior run history.
- Room interiors — enemy composition, obstacle layout, reward tier — seeded from `RunSeed + RoomIndex` via `FRandomStream`.
- Player sees the graph as a navigable map UI; path choices are locked after the current room is entered.

```cpp
void UDungeonGraphGen::GenerateGraph(int32 Seed)
{
    FRandomStream Stream(Seed);
    TArray<FDungeonNode> Nodes;

    for (int32 Depth = 0; Depth < MaxDepth; ++Depth)
    {
        int32 Width = Stream.RandRange(MinWidth, MaxWidth);
        for (int32 i = 0; i < Width; ++i)
        {
            FDungeonNode Node;
            Node.Type = SelectRoomType(Depth, Stream);
            Node.Seed = Stream.GetCurrentSeed();
            Nodes.Add(Node);
        }
    }

    ConnectNodes(Nodes, Stream);
}
```

---

### 4. Reward System

Rewards are offered between rooms and after elite/boss encounters.

- **Card Offers**: 3 cards drawn from a weighted pool filtered by current deck tags and run depth. Player picks one; others are discarded.
- **Relics**: Persistent passive items that modify run-wide rules. Defined as `URelicDataAsset` with trigger hooks into the event bus.
- **Shop**: Appears at designated map nodes — offers cards, relics, and consumables for gold. Gold drops from elite encounters and treasure rooms.
- **Rest Site**: Player chooses between healing HP or upgrading one card in deck.

Offer pools are built from `UDataTable` rows weighted by rarity and filtered via `FGameplayTagQuery` against current deck composition — decks already heavy in a tag see reduced offers of that type to encourage build diversity.

---

### 5. Enemy AI

Enemies use Unreal's Behavior Tree system extended with custom `UBTTask` and `UBTDecorator` nodes in C++.

- Each enemy archetype (`AEnemyCharacter` subclass) has a distinct BT asset defining attack patterns, positioning logic, and retreat thresholds.
- **Intent System**: enemies telegraph their next action one beat before execution — intent icon displayed above enemy head, giving card-play decisions a reactive dimension.
- Intent is resolved by the BT and broadcast via `OnIntentChanged` delegate consumed by the HUD.
- Elite enemies have phase transitions: health threshold triggers `OnPhaseChange` delegate, injecting a new BT subtree with an expanded move set.

---

### 6. Run State & Meta Progression

**Per-Run State:**
- `URunStateComponent` on `GameMode` tracks current HP, gold, deck, relics, dungeon seed, and room history for the active run.
- On run end (death or boss clear), run data is summarized and written to the meta save.

**Meta Progression:**
- Persistent unlocks (new card pool entries, starting relic options, character variants) stored in `UMetaSaveGame`.
- Unlocks gated behind run achievements — first clear of depth X, building a specific tag combination, etc.
- Meta save is separate from run state; run save is discarded on death (roguelike permadeath enforced at save layer).

---

### 7. Save System

- Active run serialized via `USaveGame` subclass on room transition — not mid-room.
- Saved data: run seed, room index, player HP/gold, deck state (card list + upgrade flags), active relics, status effects, map graph with traversal state.
- Async write via `UGameplayStatics::AsyncSaveGameToSlot` to avoid hitch on room load.
- On game launch, presence of a run save triggers resume prompt; on death the slot is cleared before returning to menu.

---

## Performance Targets & Optimization

| Target | Approach |
|---|---|
| 60 fps (PC, 1080p+) | LOD chains on all meshes; Nanite for static geo; draw call budgeting |
| Room streaming | Rooms loaded async via `TSoftObjectPtr`; previous room unloaded on transition |
| Tick optimization | Status effect ticks batched on shared timer; idle actors disable tick |
| AI | BT paused for off-screen enemies; perception system range-gated |
| Card UI | Hand widget pooled — card actors reused across draw cycles rather than respawned |

---

## Development Scope

| Category | Detail |
|---|---|
| Developer count | 1 (solo) |
| Status | Early development — systems, visuals, and balance actively changing |
| Engine | Unreal Engine 5 |
| Languages | C++, HLSL (custom shader nodes) |
| 3D Assets | All original — modeled, textured, rigged, animated by developer |
| Platforms | PC Windows (Steam) |
| Development tools | UE5 Editor, ZBrush, Maya, Blender, Substance Painter, Photoshop |

---

## Related Projects

| Project | Description |
|---|---|
| [TIME SOUL](https://store.steampowered.com/app/2928270/TIME_SOUL) | Souls-like action platformer; C++, UE5, time-as-health resource system |
| [Skysmith Dragons](#) | Sandbox survival; dragon taming, airship base building, aerial exploration |
| [U.N Owen Was Her](https://store.steampowered.com/app/3420540/UN_Owen_Was_Her) | Third-person action shooter; boss AI, bullet-hell patterns |
| [Olympus of the Heavens](https://store.steampowered.com/app/3358020/Olympus_of_the_Heavens) | Isometric co-op ARPG; procedural gen, Steam networking |
| [ArtStation Portfolio](https://www.artstation.com/kubrik) | 3D modeling — characters, creatures, props, environments |

---

## Developer

**Kubrik** — Developer & 3D Artist  

[Steam](https://store.steampowered.com/search/?developer=Kubrik) · [ArtStation](https://www.artstation.com/kubrik)

---

*All code, design, and custom assets produced by a single developer. No third-party gameplay code used in core systems.*
