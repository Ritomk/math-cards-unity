# Math Cards

**A turn-based strategy card game** built in **Unity (C#)** where players construct **Reverse Polish Notation (RPN)** expressions to attack and defend against an AI opponent. Developed as an engineering thesis at Rzeszów University of Technology.

![Screenshot of Math Cards gameplay](ReadmeImages/GameScreen.png)

> **Note**: This project originated from a [C++ prototype](https://github.com/Ritomk/math-cards-v2) built as a university assignment, later evolved into a fully-featured 3D Unity game with custom AI, shader effects, and a complete game loop.

---

## Table of Contents

- [Key Highlights](#key-highlights)
- [Architecture Overview](#architecture-overview)
  - [Dual State Machine System](#dual-state-machine-system)
  - [ScriptableObject Event Bus](#scriptableobject-event-bus)
  - [Container Hierarchy](#container-hierarchy)
  - [AI Behaviour Tree](#ai-behaviour-tree)
- [Custom Shaders & Visual Effects](#custom-shaders--visual-effects)
- [Gameplay Design](#gameplay-design)
  - [Objective](#objective)
  - [Cards & RPN](#cards--rpn)
  - [Merge System (Card Chest)](#merge-system-card-chest)
  - [AI Opponent](#ai-opponent)
  - [Round System](#round-system)
- [Technologies Used](#technologies-used)
- [Project Structure](#project-structure)
- [Installation](#installation)
- [License](#license)
- [Acknowledgements](#acknowledgements)

---

## Key Highlights

| Area | Details |
|---|---|
| **Architecture** | Generic dual state machine (Game + Player), ScriptableObject-based event bus, polymorphic container system |
| **AI System** | Behaviour tree with recursive RPN expression generation, probability-weighted decisions, async computation, and sabotage logic |
| **Shader Pipeline** | 6 custom ShaderGraph shaders — dissolve, highlight with rain effect, distortion, dithering post-process |
| **Coroutine Framework** | Custom `CoroutineHelper` — a centralized coroutine manager with pause/resume, ID-based tracking, and `WaitForSecondsPauseable` |
| **Game Systems** | RPN evaluator with animated card visualization, card merging, deck generation with balanced distribution, spline-based hand layout |
| **Input** | Unity Input System with action maps, hold interactions, context-sensitive state filtering |

---

## Architecture Overview

The project follows a **decoupled, event-driven architecture** designed to minimize coupling between game systems. Core systems communicate through ScriptableObject events rather than direct references, making components independently testable and easily extensible.

### Dual State Machine System

The game runs **two concurrent generic state machines** — one for the overall game flow, and one for the player's interaction state. Both are managed by a single `GameManager` orchestrator.

```
Game State Machine                              Player State Machine
┌───────────┐                                   ┌───────────────────┐
│  Setup     │──────►┌────────────┐              │  TurnIdle         │
└───────────┘        │ BeginRound │              │  CardPicked       │
                     └────────────┘              │  CardPlacedTable  │
                          │                      │  CardPlacedMerger │
             ┌────────────┼────────────┐         │  AllCardsPlaced   │
             ▼                         ▼         │  OpponentTurnIdle │
      ┌────────────┐           ┌──────────────┐  │  LookAround       │
      │ PlayerTurn │◄─────────►│ OpponentTurn │  │  PauseGame        │
      └────────────┘           └──────────────┘  │  BeginRound       │
             │                         │         │  EndRound          │
             └────────────┬────────────┘         └───────────────────┘
                          ▼
                   ┌────────────┐
                   │  EndRound  │
                   └────────────┘
```

**Key design decisions:**
- **`StateMachine<TStateEnum>`** — generic type parameter allows reuse for both game-level and player-level flows without code duplication
- **Coroutine-based transitions** — `Enter()` / `Exit()` return `IEnumerator`, enabling animated transitions and async operations within state changes
- **Queue-based processing** — state changes are enqueued and processed sequentially, preventing race conditions during rapid transitions
- **State history with rollback** — maintains a bounded history (10 states) supporting `RevertToPreviousState()` for undo-like operations (e.g., releasing a picked card)
- **Duplicate detection** — guards against re-entering the current state and consecutive duplicates in the queue

### ScriptableObject Event Bus

Instead of tight coupling or a global singleton event system, game systems communicate via **ScriptableObject-based events** — a pattern inspired by Ryan Hipple's Unite talk. Each event channel is a `ScriptableObject` asset configured in the inspector.

```
┌──────────────┐    SoCardEvents       ┌──────────────┐
│ CardPick     │──────────────────────►│ CardManager  │
│ Controller   │                       └──────────────┘
└──────────────┘    SoGameStateEvents   ┌──────────────┐
                  ──────────────────────►│ GameManager  │
┌──────────────┐    SoContainerEvents   └──────────────┘
│ Table        │──────────────────────►┌──────────────┐
│ Container    │                       │ EndRoundState│
└──────────────┘                       └──────────────┘
```

**Event channels implemented:**
| ScriptableObject | Responsibility |
|---|---|
| `SoGameStateEvents` | Game/Player state transitions, round lifecycle, pause system |
| `SoCardEvents` | Card movement between containers, draw, selection |
| `SoContainerEvents` | RPN evaluation triggers, merge operations, table clearing, data exchange for AI |
| `SoAnimationEvents` | Chest animation triggers |
| `SoTimerEvents` | Turn timer control |
| `SoUIEvents` | UI updates (crystals, sliders, end-game screen) |
| `SoUniversalInputEvents` | Decoupled input broadcasting (mouse moves, camera, card pick) |

This pattern provides:
- **Zero-coupling between publisher and subscriber** — systems only depend on the SO asset, not on each other
- **Inspector-driven wiring** — event connections are visible and configurable in the Unity editor
- **Play-mode resilience** — ScriptableObjects persist across scene reloads
- **`out` parameter pattern** — events like `OnCardMove` use `out bool success` to allow subscribers to communicate results back synchronously

### Container Hierarchy

All card locations (hand, deck, tables, merger) inherit from a common **`CardContainerBase`** abstract class, creating a polymorphic container system:

```
CardContainerBase (abstract)
├── DeckContainer : IDrawableContainer
│   └── Procedural deck generation with balanced operand distribution
│   └── Fisher-Yates shuffle
├── HandContainer : IDrawableContainer
│   └── Spline-based card positioning (SplineContainer)
│   └── Duplicate card grouping with weighted random burn
│   └── Per-owner behavior (Player: visible + animations, Enemy: hidden)
├── TableContainer
│   └── Dynamic spacing with max-width constraint
│   └── RPN expression evaluation + animated visualization
│   └── Real-time card placement validation (stack height tracking)
└── MergerContainer
    └── Two-card merge with chest animation synchronization
    └── Automatic result card repatriation to hand
```

**Design highlights:**
- **`ContainerKey` struct** — a composite key `(OwnerType, CardContainerType)` uniquely identifies each container, enabling a clean card-move API: `RaiseCardMove(card, fromKey, toKey)`
- **`IDrawableContainer` interface** — shared contract for containers that support drawing cards
- **AI data gathering** — each container feeds data to the AI via the `HandleCardData(EnemyKnowledgeData)` virtual method, allowing the behaviour tree to build a complete world model

### AI Behaviour Tree

The AI opponent uses a **NodeCanvas behaviour tree** with custom nodes organized into three categories:

```
Behaviour Tree
├── GatherData/
│   ├── GetCardsDataFromContainer — builds EnemyKnowledgeData from all containers
│   └── GenerateMoves — recursive RPN expression generation (async, off main thread)
│
├── Condition/
│   ├── CheckCanPlace — validates card placement legality
│   ├── CheckPlayerHasEndedRound — adjusts strategy when player passes
│   └── CheckToMessWithPlayer — probabilistic sabotage with escalating chance
│
└── Action/
    ├── CalculateChance — piecewise-linear probability mapping
    ├── IsWinning — evaluates board advantage, adjusts attack/defense priority
    ├── PlaceCard — places cards on AI's own tables
    ├── PlaceCardPlayerTable — interferes with player's expressions
    ├── MergeCards — strategic card merging
    ├── EndTurn / EndRound — turn lifecycle
    └── HasWon — final evaluation
```

**AI strategy capabilities:**
- **Recursive expression optimization** — `RpnExpressionGenerator` uses backtracking to generate all valid RPN expressions from available cards and picks the one yielding the maximum value. Computation runs **asynchronously via `Task.Run()`** to avoid frame hitches
- **Sabotage system** — the AI can place subtraction/division operators or low-value operands on the *player's* tables to sabotage their expressions. The chance escalates on each failed roll (quadratic growth) and resets to a base value (20%) after succeeding
- **Adaptive priorities** — `IsWinning` evaluates the board state differential and dynamically shifts between offensive and defensive strategies, multiplying the base priority by up to 4× depending on the advantage gap
- **Card pooling** — AI maintains a `Dictionary<int, Card>` available cards pool, managing which cards are committed to expressions vs. still available for placement

---

## Custom Shaders & Visual Effects

The project includes **6 custom ShaderGraph shaders** and **3 C# shader controllers** providing polish and game feel:

| Shader | Type | Purpose |
|---|---|---|
| **Dissolve** | Surface | Card burn/destruction with procedural noise and randomized seed |
| **DissolveRope** | Surface | Variant for rope/connection dissolve effects |
| **Highlight** | Surface | Animated rain/glow effect for card hover states with per-card color control |
| **Distortion** | Surface | Heat-haze distortion effect |
| **Dithering** | Full Screen | Post-processing dithering for stylized rendering |
| **Test** | Surface | Development/iteration shader |

**Shader controllers** drive these effects from C# with smooth color lerping, pauseable animations, and coroutine-managed transitions:

- **`DissolveEffect`** — drives dissolution over time with random seed variation, includes auxiliary object hiding at configurable thresholds
- **`HighlightEffect`** — manages outline and rain color channels with smooth interpolation, supports pause-aware animation speed, and an `AddCardToHand` flash animation (color ping-pong on add)
- **`TimerDissolveEffect`** — timer-specific dissolution variant

---

## Gameplay Design

### Objective
Each round, both players simultaneously build RPN expressions to maximize the gap between their attack and the opponent's defense:

```
Score = max(YourAttack − OpponentDefense, 0) − max(OpponentAttack − YourDefense, 0)
```

### Cards & RPN
| Type | Examples | Encoded As |
|---|---|---|
| **Operands** | 0–9 | `float` (face value) |
| **Operators** | + − × ÷ | `float` (0.05–0.08 range) |

Cards are placed to build RPN expressions (e.g., `3 5 +` = 8). The game includes real-time **stack-height based validation** — invalid placements are automatically burned with a dissolve animation.

**Special rule — "0" card**: Placing a 0 on the table converts it to 0.1, preventing expression collapse from multiplication/division by zero while keeping strategic viability.

### Merge System (Card Chest)
The merger chest allows combining two number cards across consecutive turns (e.g., `8` + `9` = `89`). The merge is:
- Animated with a physical chest open/close sequence
- Validated to only accept single-digit operands
- Irreversible — the merged card returns to hand automatically

### AI Opponent
The AI utilizes a behaviour tree implementing multiple strategic layers:
- **Expression optimization** — generates optimal RPN sequences through recursive backtracking
- **Sabotage** — probabilistically interferes with the player's board using subtraction, division, or small operands
- **Adaptive strategy** — dynamically shifts attack/defense priority based on board advantage
- **Async computation** — heavy calculations run off the main thread via `Task.Run()`

### Round System
- **Best of 3** format — first to 2 round wins takes the match
- Round ties award a point to both sides
- Final-round ties resolved randomly
- Progress tracked via **crystal indicators** (green = player, red = AI)

---

## Technologies Used

| Technology | Usage |
|---|---|
| **Unity Engine (URP)** | Core platform with Universal Render Pipeline |
| **C#** | All game logic, ~30 scripts, ~4000 lines |
| **ShaderGraph + HLSL** | 6 custom shaders (surface + full-screen post-processing) |
| **Unity Input System** | Action maps with hold interactions and context-aware filtering |
| **Unity Splines** | Procedural hand card layout along configurable curves |
| **Unity Timeline** | Sequenced animation playback |
| **NodeCanvas** | AI behaviour tree framework *(paid asset, not included in repo)* |
| **TextMeshPro** | Card value rendering |
| **async/await + Task.Run** | Offloading AI expression generation to background threads |

---

## Project Structure

```
math-cards-unity/
├── Assets/
│   ├── Assets/
│   │   ├── Materials/          # Card & environment materials
│   │   ├── Models/             # 3D models (cards, chests, table)
│   │   ├── Prefabs/            # Card prefab, UI prefabs
│   │   ├── Shaders/            # 6 ShaderGraph shaders
│   │   │   ├── Dissolve.shadergraph
│   │   │   ├── DissolveRope.shadergraph
│   │   │   ├── Distortion.shadergraph
│   │   │   ├── Highlight.shadergraph
│   │   │   └── FullScreen/
│   │   │       └── Dithering.shadergraph
│   │   ├── Textures/           # Card textures, UI elements
│   │   └── UI/                 # UI assets
│   │
│   ├── Scripts/
│   │   ├── GameManager/        # Core game loop
│   │   │   ├── GameManager.cs          # Orchestrator — wires both state machines
│   │   │   ├── StateMachine.cs         # Generic state machine with queue & history
│   │   │   ├── StateBase.cs            # Abstract base with coroutine Enter/Exit
│   │   │   ├── Game States/            # 8 game states (Setup → EndRound)
│   │   │   └── Player States/          # 10 player states (Idle → Pause)
│   │   │
│   │   ├── Cards/              # Card logic
│   │   │   ├── Card.cs                 # Card entity — token, state, shader control
│   │   │   ├── CardData.cs             # ScriptableObject — state-to-color mapping
│   │   │   ├── CardManager.cs          # Card movement dispatcher
│   │   │   ├── CardPickController.cs   # Drag-drop with state-aware validation
│   │   │   ├── CardHighlightController.cs
│   │   │   └── CardSelectionController.cs
│   │   │
│   │   ├── Containers/         # Polymorphic card container system
│   │   │   ├── CardContainerBase.cs    # Abstract base with shared API
│   │   │   ├── ContainerKey.cs         # (OwnerType, ContainerType) composite key
│   │   │   ├── DeckContainer.cs        # Procedural generation + Fisher-Yates shuffle
│   │   │   ├── HandContainer.cs        # Spline-based layout + duplicate grouping
│   │   │   ├── TableContainer.cs       # RPN eval + animated visualization
│   │   │   └── MergerContainer.cs      # Two-card merge with chest sync
│   │   │
│   │   ├── Events/             # ScriptableObject event channels (7 SO types)
│   │   │
│   │   ├── Behaviour Tree/     # AI — custom NodeCanvas nodes
│   │   │   ├── Action/         # 8 action nodes
│   │   │   ├── Condition/      # 3 condition nodes
│   │   │   └── GatherData/     # 3 data-gathering nodes
│   │   │
│   │   ├── Visuals/            # Shader controllers + animations
│   │   │   ├── Shaders/                # DissolveEffect, HighlightEffect, TimerDissolve
│   │   │   └── Animations/             # ChestAnimation, CoinFlipAnimation, TimerAnimation
│   │   │
│   │   ├── Player Actions/     # Input handling
│   │   │   ├── InputManager.cs         # Unity Input System bindings
│   │   │   ├── CameraController.cs     # Look-around camera
│   │   │   └── InputUIManager.cs       # UI input layer
│   │   │
│   │   └── Utils/              # Shared utilities
│   │       ├── CoroutineHelper.cs      # Centralized coroutine manager (pause/resume/track)
│   │       ├── RpnExpressionHelper.cs  # RPN stack evaluator (multiple evaluation modes)
│   │       └── RpnExpressionGenerator.cs # Recursive backtracking expression optimizer
│   │
│   ├── ScriptableObjects/      # SO assets (events, data, behaviour tree)
│   ├── Scenes/                 # Game scenes
│   └── Settings/               # URP settings, quality profiles
│
├── Packages/                   # Unity package manifest
├── ProjectSettings/            # Unity project configuration
└── README.md
```

---

## Installation

1. **Clone the repository:**
   ```bash
   git clone https://github.com/Ritomk/math-cards-unity.git
   ```

2. **Open in Unity:**
   - Open via Unity Hub
   - Requires Unity with URP support (check `ProjectSettings` for exact version)

3. **Import NodeCanvas:**
   - Purchase and import [NodeCanvas](https://assetstore.unity.com/packages/tools/visual-scripting/nodecanvas-14914) from the Unity Asset Store
   - Required for AI functionality

4. **Run:**
   - Press Play in the Unity Editor, or build via `File > Build Settings`

---

## License

This project is intended for educational purposes and is not licensed for commercial use. All rights reserved by the author.

## Acknowledgements

- **NodeCanvas** — [Unity Asset Store](https://assetstore.unity.com/packages/tools/visual-scripting/nodecanvas-14914)
- **Rzeszów University of Technology** — academic supervision and thesis framework
