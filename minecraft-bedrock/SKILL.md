---
name: minecraft-bedrock
description: Expert knowledge of Minecraft Bedrock Edition add-on development — entity pipelines, animation systems, attachables, Script API, and the Modinator tool project. Use when working on Minecraft Bedrock mods, the Modinator app, BlockBench integration, add-on JSON wiring, player animations, entity behaviors, Script API scripting, or anything related to Bedrock pack structure.
---

# Minecraft Bedrock Add-On Expert

## Project Context: Modinator
A Windows desktop app (Python) for kids aged 8-12 to create Bedrock mods without touching JSON.
See [MODINATOR.md](./MODINATOR.md) for full design decisions.

## Bedrock Pack Structure

Every add-on is a **pair** of packs:

```
development_behavior_packs/<ProjectName>_BP/
  manifest.json
  entities/<entity>.json          ← AI behaviors, components
  items/<item>.json
  blocks/<block>.json
  recipes/
  scripts/                        ← Script API JavaScript
  animation_controllers/          ← Server-side state machines

development_resource_packs/<ProjectName>_RP/
  manifest.json                   ← must reference BP UUID in dependencies
  entity/<entity>.json            ← client entity: links everything together
  models/entity/<entity>.geo.json ← geometry (BlockBench export)
  animations/<entity>.animation.json ← keyframes (BlockBench export)
  animation_controllers/<entity>.animation.json ← client state machine
  render_controllers/<entity>.render.json
  textures/entity/<name>.png
  attachables/<item_id>.json      ← cosmetic worn items
```

**com.mojang path:** `C:\Users\Sean McCann\AppData\Roaming\Minecraft Bedrock\Users\Shared\games\com.mojang`

## The 6-File Entity Pipeline

Every entity requires these 6 files with perfectly matched strings:

| # | File | Key Identifiers |
|---|------|----------------|
| 1 | `BP/entities/<e>.json` | `identifier: "ns:<e>"` |
| 2 | `RP/entity/<e>.json` | same identifier, short-name maps for geometry/textures/animations/render_controllers |
| 3 | `RP/models/entity/<e>.geo.json` | `geometry.<e>` identifier |
| 4 | `RP/animations/<e>.animation.json` | `animation.<e>.<state>` identifiers |
| 5 | `RP/animation_controllers/<e>.animation.json` | `controller.animation.<e>.<name>` |
| 6 | `RP/render_controllers/<e>.render.json` | `controller.render.<e>` |

**One wrong string = silent failure, nothing appears in game.**

## Critical String Rules
- All identifiers: lowercase, alphanumeric + underscores + periods only
- Geometry: `geometry.<namespace>_<name>` (e.g. `geometry.modinator_dragon`)
- Animations: `animation.<namespace>_<name>.<state>` (e.g. `animation.modinator_dragon.walk`)
- Animation controllers: `controller.animation.<namespace>_<name>.<purpose>`
- Render controllers: `controller.render.<namespace>_<name>`
- Short names (in RP entity `animations: {}` block) are arbitrary but must match what's used in animation controller states

## manifest.json Format

```json
{
  "format_version": 2,
  "header": {
    "name": "My Pack Name",
    "description": "Description",
    "uuid": "<generate unique UUID>",
    "version": [1, 0, 0],
    "min_engine_version": [1, 21, 0]
  },
  "modules": [
    {
      "type": "data",           // behavior pack
      "uuid": "<generate unique UUID>",
      "version": [1, 0, 0]
    }
  ],
  "dependencies": [
    {
      "uuid": "<RP or BP uuid here>",
      "version": [1, 0, 0]
    }
  ]
}
```
Resource pack module type is `"resources"`. Script API adds a `"javascript"` module.

## RP Client Entity Format (the wiring hub)

```json
{
  "format_version": "1.10.0",
  "minecraft:client_entity": {
    "description": {
      "identifier": "modinator:dragon",
      "materials": { "default": "entity_alphatest" },
      "textures": {
        "default": "textures/entity/dragon/dragon"
      },
      "geometry": {
        "default": "geometry.modinator_dragon"
      },
      "animations": {
        "idle":   "animation.modinator_dragon.idle",
        "walk":   "animation.modinator_dragon.walk",
        "attack": "animation.modinator_dragon.attack",
        "move":   "controller.animation.modinator_dragon.move"
      },
      "scripts": {
        "animate": ["move"]
      },
      "render_controllers": ["controller.render.modinator_dragon"]
    }
  }
}
```

## Animation Controller (State Machine)

```json
{
  "format_version": "1.17.30",
  "animation_controllers": {
    "controller.animation.modinator_dragon.move": {
      "initial_state": "default",
      "states": {
        "default": {
          "animations": ["idle"],
          "transitions": [
            { "walking": "query.modified_move_speed > 0.1" },
            { "attacking": "variable.attack_time > 0.0" }
          ]
        },
        "walking": {
          "animations": ["walk"],
          "transitions": [
            { "default": "query.modified_move_speed <= 0.1" }
          ]
        },
        "attacking": {
          "animations": ["attack"],
          "transitions": [
            { "default": "query.all_animations_finished" }
          ]
        }
      }
    }
  }
}
```

## Key Molang Queries
- `query.modified_move_speed` — movement speed (>0.1 = walking)
- `query.is_jumping` — player jumping
- `variable.attack_time` — swing progress (0.0 to 1.0), key for attack animations
- `query.anim_time` — time since current animation started
- `query.all_animations_finished` — transition trigger
- `query.any_animation_finished` — transition trigger
- `query.is_moving` — 1.0 if moving
- `query.is_on_ground` — 1.0 if grounded
- `query.armor_color_slot(slot)` — armor slot colors
- `query.armor_texture_slot(slot)` — armor texture index

## Attachables (Cosmetics — Dragon Tail Pattern)

Attachables are resource-pack-only cosmetic items worn by the player. The dragon tail pants are a proven working example.

```json
{
  "format_version": "1.10.0",
  "minecraft:attachable": {
    "description": {
      "identifier": "modinator:dragon_tail",
      "materials": { "default": "entity_alphatest" },
      "textures": { "default": "textures/entity/dragon_tail/dragon_tail" },
      "geometry": { "default": "geometry.modinator_dragon_tail" },
      "animations": {
        "idle_wag": "animation.modinator_dragon_tail.idle_wag",
        "move":     "controller.animation.modinator_dragon_tail.move"
      },
      "scripts": { "animate": ["move"] },
      "render_controllers": ["controller.render.modinator_dragon_tail"]
    }
  }
}
```
- Triggered by equipping the corresponding item in the correct armor slot
- Slot 0=head, 1=chest, 2=legs, 3=feet
- No BP entity file needed — RP only

## Script API Patterns

### File Setup (BP/scripts/main.js)
```javascript
import { world, system } from "@minecraft/server";
```

BP manifest needs a `"javascript"` module:
```json
{
  "type": "script",
  "language": "javascript",
  "uuid": "<uuid>",
  "version": [1, 0, 0],
  "entry": "scripts/main.js"
}
```
And dependency on `@minecraft/server`:
```json
{ "module_name": "@minecraft/server", "version": "1.15.0" }
```

### Double Jump Template
```javascript
import { world, system } from "@minecraft/server";

const jumpCounts = new Map();

world.afterEvents.playerSpawn.subscribe(({ player }) => {
  jumpCounts.set(player.id, 0);
});

world.afterEvents.entityHitEntity.subscribe(({ damagingEntity }) => {});

// Track jumps via velocity change
system.runInterval(() => {
  for (const player of world.getAllPlayers()) {
    const vel = player.getVelocity();
    const isRising = vel.y > 0.3;
    const onGround = player.isOnGround;
    
    if (onGround) {
      jumpCounts.set(player.id, 0);
    } else if (isRising) {
      const count = jumpCounts.get(player.id) ?? 0;
      if (count < 2) {
        jumpCounts.set(player.id, count + 1);
        if (count === 1) {
          // Second jump — apply extra impulse
          player.applyKnockback(0, 0, 0, 0.6);
        }
      }
    }
  }
}, 1);
```

### Item-Specific Attack Animation Template
```javascript
import { world, EquipmentSlot } from "@minecraft/server";

const WEAPON_ANIMATIONS = {
  "modinator:katana": "katana_slash",
  "modinator:spear":  "spear_thrust",
};

world.afterEvents.entityHitEntity.subscribe(({ damagingEntity }) => {
  if (damagingEntity.typeId !== "minecraft:player") return;
  const equip = damagingEntity.getComponent("minecraft:equippable");
  const item = equip?.getEquipment(EquipmentSlot.Mainhand);
  if (!item) return;
  const anim = WEAPON_ANIMATIONS[item.typeId];
  if (anim) {
    // Set a Molang variable to trigger the animation controller transition
    damagingEntity.setProperty("modinator:attack_anim", anim);
  }
});
```

## BlockBench Integration
- BlockBench exports: `.geo.json` (geometry), `.animation.json` (animations), textures (`.png`)
- Export destinations map to RP folders above
- `.bbmodel` files are BlockBench's project format — keep these as source of truth
- Plugin should watch for save and copy exports to the correct RP subfolder

## Common Failure Modes
1. **Silent black screen / invisible entity** — geometry identifier mismatch between `.geo.json` and `RP/entity` reference
2. **Entity spawns but T-poses** — animation controller not registered in `RP/entity` scripts.animate
3. **Animations play but wrong one** — short name mismatch between `RP/entity animations{}` and animation controller state
4. **Pack shows in game but does nothing** — BP manifest UUID not in RP manifest dependencies
5. **Script API not running** — missing `"javascript"` module in BP manifest or wrong entry path
