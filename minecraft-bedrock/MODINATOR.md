# Modinator — Full Design Reference

## What It Is
A Windows desktop app (Python) for kids aged 8-12 to create Minecraft Bedrock Edition mods.
Kids make assets in BlockBench; Modinator handles ALL the JSON wiring automatically.

## Core Problem
Kids can create models, animations, textures in BlockBench but can't wire them into the game.
The 6-file entity pipeline with perfectly matched string identifiers is the black box Modinator solves.

## Key Paths
- **com.mojang root:** `C:\Users\Sean McCann\AppData\Roaming\Minecraft Bedrock\Users\Shared\games\com.mojang`
- **Behavior packs:** `...\com.mojang\development_behavior_packs\`
- **Resource packs:** `...\com.mojang\development_resource_packs\`
- **Modinator projects:** `C:\AI_Projects\Minecraft_Tools\Projects\` (app's own clean store)

## Design Decisions

| Decision | Choice | Reason |
|---|---|---|
| Target | Bedrock Edition only | JSON add-ons, no Java compiler needed |
| Platform | Windows desktop app (Python) | Familiar-ish to Lua dev, fast to build |
| Scripts generated | JavaScript (Script API) | Only option Bedrock supports |
| Behavior approach | Pre-built template library | No token costs ("Dad can we get more tokens?") |
| Project storage | App owns clean folder, Deploy copies to com.mojang | Prevents com.mojang chaos |
| UI model | Project-based ("Dragon Mod") | Mirrors real pack structure, gives kids ownership |
| Organization | Hard naming rules enforced | Eliminate `armerdilo`, `b`, `t` style chaos |
| Profiles | Creator name selector (no login) | Shared workspace, tagged ownership |
| BlockBench | Bidirectional: app opens file, plugin sends back on save | Removes all file-hunting friction |

## Scope

### Supported Content Types
- Custom entities (mobs) — full 6-file pipeline generated automatically
- Custom items — BP item + RP attachable + textures
- Custom blocks — BP block + RP geometry + textures
- Player cosmetic attachables — proven working (dragon tail pants)
- Player movement animation overrides — override player.json
- Item-specific attack animations — Script API template
- Entity AI behaviors — behavior components via dropdowns/sliders
- Script API behaviors — pre-built templates (double jump, etc.)

### Template Library (Script API, pre-built)
- Double jump
- Custom weapon swing (body animation per item)
- Weapon model animation on swing
- Entity patrol / follow / flee
- Custom entity interactions (ride, tame)
- More added over time by Sean

## UI Flow

### Project Creation
1. "Who are you?" → select/enter creator name (Kate, etc.)
2. "New Project" → enter project name (enforced: no single letters, no duplicates, no typos)
3. Project tagged with creator, stored in clean project folder

### Adding an Entity
1. Pick "Add Entity" → enter entity name
2. Import .bbmodel from BlockBench OR click "Create in BlockBench" to open BlockBench
3. App generates all 6 files automatically with matching identifiers
4. Select behavior template (patrol, follow player, etc.) → configure sliders
5. Select animation states (idle, walk, attack, death) → link to animations from BlockBench
6. "Deploy" → copies both packs to com.mojang, live in game immediately

### BlockBench Integration
- "Edit in BlockBench" button → opens correct .bbmodel file
- BlockBench plugin detects save → auto-imports geometry + animations back into project
- No manual file copying ever

### Deploy
- One button copies BP + RP folders to com.mojang development packs
- Validates all string references before deploying
- Shows what changed since last deploy

## Import Flow (for existing com.mojang mess)
- Scan development_behavior_packs and development_resource_packs
- Detect existing packs (by manifest.json)
- Offer to import into a named project
- Suggest merging duplicates (armerdilo + armerdillo + armadilo → "Armadillo Mod")

## Naming Rules Enforced
- Minimum 3 characters
- No single letters or numbers alone
- Must be unique within creator's projects
- Spaces allowed in display name, sanitized to underscores in identifiers
- Warning if name is very similar to existing project (Levenshtein distance check)

## Kids / Profiles
- Creator names: stored in app config (Kate, etc.)
- "Who are you?" prompt on launch or settings
- All projects visible to all creators (shared family workspace)
- Projects tagged with creator name for ownership/pride
- No login, no passwords

## Tech Stack
- **App:** Python desktop app
- **Minecraft scripts:** JavaScript (Script API, pre-written templates)
- **BlockBench plugin:** JavaScript (BlockBench plugin API)
- **Project store:** JSON files in `C:\AI_Projects\Minecraft_Tools\Projects\`
- **UI framework:** TBD (tkinter for speed, or a lightweight web framework)
