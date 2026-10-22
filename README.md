# fps1

A first-person shooter built on Unity's **FPS Microgame** template.

- **Unity version:** 6000.6.3f1 (Unity 6)
- **Render pipeline:** Universal Render Pipeline (URP 17.6). The pipeline asset `Assets/Rendering/Default_PipelineAsset.asset` is assigned in both Graphics and Quality settings.
- **Input:** Input System package
- **Asset serialization:** Force Text. Scenes, prefabs, materials and most `.asset` files are saved as YAML text, so git can diff and merge them.

## Getting started

1. Open this folder (`D:\fps1\fps1`) in Unity Hub with Unity 6000.6.3f1.
2. Open `Assets/FPS/Scenes/IntroMenu.unity` (the start menu) or `MainScene.unity` (the main level) and press Play.

## Folder overview

### Top level

| Folder | What it is | Commit to git? |
|---|---|---|
| `Assets/` | All game content: scenes, scripts, models, materials, audio and prefabs. This is where you do your work. | Yes |
| `Packages/` | `manifest.json` lists the Unity packages the project uses (URP, Input System, ProBuilder, AI Navigation and others). | Yes |
| `ProjectSettings/` | Project-wide settings: graphics/URP, quality, input, physics, tags/layers and the build scene list. | Yes |
| `UserSettings/` | Your personal editor preferences. | No |
| `Library/` | Unity's cache of imported assets. It is rebuilt automatically, so deleting it is safe (the next open will be slow). | No |
| `Temp/`, `Logs/` | Temporary build files and editor logs. | No |
| `*.csproj`, `*.sln` | Files Unity generates for Visual Studio and Rider. | No |

Everything marked "No" is listed in `.gitignore`. `Assets/FPS/Tutorials/` is ignored too (see below).

### Inside `Assets/`

**Text** means the file is readable and diffable in git. **Binary** means it isn't; see [Binary files](#binary-files-in-assets) for the full list. Every asset and folder also has a `.meta` file beside it, and those are always text.

| Folder | What it is | Contents |
|---|---|---|
| `FPS/` | The core game (see below). | Mixed |
| `ModAssets/` | Optional add-on content (props, hats, trails, billboards). None of it is used by the game right now; see [ModAssets](#modassets). | Binary models and textures; text materials and prefabs |
| `NavMeshComponents/` | NavMesh tools (`NavMeshSurface` and others) used to bake the walkable areas enemies path-find on; see [NavMesh](#what-baking-a-navmesh-means). | Text (C# scripts) |
| `Rendering/` | URP setup: the pipeline asset, the forward renderer and the post-processing profile. Edit these to change shadows, anti-aliasing, bloom and similar effects. | Text (`.asset`) |
| `TextMesh Pro/` | Fonts, shaders and resources for TextMesh Pro, which renders UI text. | Text, except one **binary** font file (`.ttf`) |
| `DefaultVolumeProfile.asset`, `UniversalRenderPipelineGlobalSettings.asset` | Global URP defaults: the post-processing volume used when a scene doesn't override it, and pipeline-wide settings. | Text |

### Inside `Assets/FPS/`

| Folder | What it is | Contents |
|---|---|---|
| `Scenes/` | Game scenes: `IntroMenu`, `MainScene`, `SecondaryScene`, `WinScene` and `LoseScene`. See [Scenes](#scenes). | Text (`.unity`, `.lighting`) |
| `Scenes/MainScene/`, `Scenes/SecondaryScene/` | Data Unity bakes for each level: lighting, NavMesh and reflection probes. | **Binary** (baked `.asset` and `.exr` files) |
| `Scripts/AI/` | Enemy behaviour: detection, navigation, patrol paths, and mobile and turret enemies. | Text (C# scripts) |
| `Scripts/Game/` | Shared core systems: actors, game events, constants, managers, audio helpers and utilities. | Text (C# scripts) |
| `Scripts/Gameplay/` | Player-facing mechanics: player controller, weapons, projectiles, pickups and objectives. | Text (C# scripts) |
| `Scripts/UI/` | HUD and menus: health bar, ammo, crosshair, objectives display and menu logic. | Text (C# scripts) |
| `Prefabs/` | Reusable object templates: enemies, level pieces, pickups, projectiles, weapons, UI and VFX. See [Prefabs](#prefabs). | Text (`.prefab`, plus `.obj`/`.mtl` meshes, which are also text) |
| `Art/Models/` | Static 3D models (no animation) for weapons, pickups, projectiles and effects. | **Binary** (FBX models) |
| `Art/Textures/` | All game textures. | **Binary** (PNG, TGA, BMP, PSD images) |
| `Art/Materials/`, `Art/Shaders/`, `Art/Fonts/` | Materials, shaders (`.shader`, `.shadergraph`, `.hlsl`) and TextMesh Pro font assets (Roboto). | Text |
| `Animation/` | The two animated enemies: their models, animation clips, Animator controllers and masks. See [Animation](#animation). | **Binary** FBX files; text `.controller`, `.mask`, `.mat` |
| `Audio/` | Music and sound effects, plus an audio mixer. | **Binary** (WAV and MP3 sound files); the `.mixer` is text |
| `Tutorials/` | Unity's in-editor tutorials. The game doesn't use them, and they're excluded by `.gitignore`. | Ignored |

Each script folder has its own assembly definition (`fps.AI`, `fps.Game`, `fps.Gameplay`, `fps.UI`), so the code compiles as separate assemblies. `Game` is the base layer that the other three depend on.

## Scenes

Only scenes in the build list (**File > Build Profiles > Scene List**) end up in the game. The build list is `IntroMenu`, `WinScene`, `LoseScene` and `MainScene`.

| Scene | In build? | What it is |
|---|---|---|
| `IntroMenu` | Yes | Start menu. |
| `MainScene` | Yes | The playable level. It's assembled from nested level prefabs (`Prefabs/Level/MainLvl/Section1–3`: Observatory, Entry, Balcony), so the scene file itself is small (about 190 KB). The player has only the Blaster. There are HoverBots, turrets, health pickups and a "kill all enemies" objective. |
| `SecondaryScene` | **No** | A second, separate level. Its geometry is stored directly in the scene file (about 40 MB) rather than in prefabs. It adds Shotgun, Launcher and Jetpack pickups. To ship it, add it to the build list and load it from a menu or the end of MainScene. |
| `WinScene`, `LoseScene` | Yes | End screens. |

### What baking a NavMesh means

Enemies need to know where they can move. Working that out from raw level geometry every frame would be slow, so Unity does it once in the editor. This is called **baking**. A `NavMeshSurface` component looks at the level's floors, walls and ramps. Using the agent's size, it works out every spot an enemy could stand on and reach, given its radius, height, the steepest slope it can climb and the tallest step it can take. The result is a simplified polygon map of the walkable floor, saved as a `NavMesh-*.asset` file. At runtime, enemies with a `NavMeshAgent` (via `Scripts/AI/NavigationModule.cs`) find paths on that map.

**Re-bake whenever you move floors, walls or ramps.** Select the object that has the `NavMeshSurface` component and click **Bake**, or enemies will walk through new walls or ignore new floors.

In `Scenes/MainScene/`, only `NavMesh-NavMeshSurface.asset` is referenced by the scene. `NavMesh-NavMeshSurface 1/2/3.asset` look like leftovers from earlier bakes.

## Prefabs

A prefab is a template, not a copy. A prefab is only in the game if something in a build scene uses it. That can be a scene placing it directly, a prefab nested inside another placed prefab, or a field pointing to it (for example, an enemy's projectile). You **reference** prefabs; you don't copy them. Placing one in a scene creates a linked instance, and editing the prefab updates every instance.

Of the 160 prefabs in `FPS/Prefabs/`:

- 66 are used by the build scenes (directly or nested).
- 48 more are used only by `SecondaryScene`, so they're **not** in the build right now. Examples are the Shotgun, Launcher and Jetpack, and many level pieces.
- 46 aren't used anywhere. These are spare parts for building levels: the ProBuilder `PBObjects/*` pieces, dungeon/stairwell/room pieces, `ObjectiveReachPoint`, `Pickup_Blaster` and `DebugTeleportPlayer`.

The `.obj`/`.mtl` files in `Prefabs/RootObjects/` and `Prefabs/PBObjects/Signs/` are the source meshes for the `Prefabs/Level/BaseObjects/*_Mesh` and sign prefabs.

## Animation

`FPS/Animation/` holds the two animated enemies, **HoverBot** and **Turret**. Files are grouped by **type**, not by model:

```
Animation/
  Rigs/         HoverBot.fbx, Turret.fbx         ← one model per enemy
  HoverBot/     HoverBot@Idle.fbx, @Shoot.fbx …  ← that enemy's clips
  Turret/       Turret@Idle.fbx, @AttackIn.fbx …
  Controllers/  HoverBot_AnimationController.controller, Turret_…   ← one per enemy
  Masks/        HoverBot.mask, Turret.mask                          ← one per enemy
```

Each enemy has one rig FBX, one clip folder, one controller and one mask, and each file has its own `.meta`. Nothing else is required.

**Rig vs. clip files**

- `Rigs/HoverBot.fbx` is the **full model**: the mesh, the skeleton it's skinned to, and the Avatar (Unity's description of the skeleton). It contains no usable animation. The enemy prefab (`Prefabs/Enemies/Enemy_HoverBot.prefab`) uses this file.
- `HoverBot/HoverBot@Shoot.fbx` and the other clip files contain **only animation**: skeleton motion with no mesh. Their import settings reuse the rig's Avatar ("Copy From Other Avatar"), so the motion drives the rig's bones. The `model@clip` naming is a common Unity convention.
- `HoverBot@Idle.fbx` isn't used. The controller's Idle state plays `HoverBot@IdleClosed` instead.

**`Rigs/Materials/Default_Material.mat` isn't needed by the rigs.** Animation doesn't use materials. This folder is something Unity creates when it pulls materials out of an FBX on import. The rigs are remapped to the real materials in `Art/Materials/Enemies/` (`HoverBot_Body`, `HoverBot_Eye`, `HoverBot_Lens`, and so on). The only thing that points at `Default_Material` is the unused `Art/Models/Mesh_ReachPoint_Objective.fbx`.

### How a controller works (example: HoverBot)

An Animator **controller** is a state machine. It decides which clip plays at any moment. Code changes the controller's **parameters**, and **transitions** between states react to those parameters. The HoverBot controller has:

- **Parameters** (set from `Scripts/AI/EnemyMobile.cs`):
  - `MoveSpeed` (float): set every frame from the bot's velocity.
  - `Alerted` (bool): set to true when the bot spots the player.
  - `Attack` (trigger): fired when the bot shoots.
  - `OnDamaged` (trigger): fired when the bot is hit.
- **Base layer states**: `Idle` → `HoverBot@IdleClosed`, `Patrolling` → `@RunFowardClosed`, `Alerted` → `@Alerted`, `Following` → `@RunFoward`.
- **Transitions**, for example:
  - `Idle → Patrolling` when `MoveSpeed > 0`.
  - `MoveSpeed > 0.05` **and** `Alerted == true` → `Alerted` (the bot opens up and plays the alert clip), then on to `Following`.
  - `Alerted == false` → back to `Patrolling`.

So when the player walks into view, `EnemyMobile.cs` calls `Animator.SetBool("Alerted", true)`. The controller leaves `Patrolling`, plays the alert animation, and switches to the open-shell chasing loop. None of this needs any animation code.

The controller also has two extra **layers** stacked on top of the base layer:

- `Attack` (weight 0.7, additive): `Attack` trigger → plays `@Shoot`.
- `Dammaged` (weight 1, additive): `OnDamaged` trigger → plays `@Hurt`.

These layers let the bot shoot or flinch **while** still moving or patrolling, instead of interrupting that movement.

### How a mask works

An **Avatar mask** is a list of the rig's bones, each switched on or off. When a layer uses a mask, that layer's animation only moves the switched-on bones.

The HoverBot mask (`Masks/HoverBot.mask`) switches on 16 of 22 bones: the wings, ears, front panel and eyeball. It switches off the root, the bottom cap and the iris. Both extra layers use it. So when `@Shoot` plays on the Attack layer, only the upper parts of the bot recoil. The root bone keeps following the base layer, so the bot doesn't stop or jump while firing. Without the mask, the shoot clip would also take over the root and the bot would twitch in place.

### Models with and without animation

Every FBX in `Art/Models/` is **static**: no animation curves and no skeleton (Animation Type is "None"). The rule holds in this project: the two animated models (with their clips) live in `FPS/Animation/`, and everything static lives in `Art/Models/` (or `ModAssets/Models/`). Unity doesn't enforce this; it's just how the template is organized. Simple motion like the spinning and bobbing of pickups is done in code (`Gameplay/Pickup.cs`), not with clips.

## ModAssets

`ModAssets/` is extra content from the FPS Microgame "mods" (the creative add-ons in the Unity Learn tutorials). **None of it is used by any scene or prefab in `FPS/`**, so it isn't in the build. Nothing in it replaces a file in `FPS/`, either: the game never loads a ModAssets file instead of an FPS file. You use it by **adding** things, such as dragging a prop prefab into a level, parenting a hat prefab onto an enemy, adding a trail prefab to a projectile, or applying a material.

- The mod weapons (`Mesh_Weapon_1/2.fbx`) are different, much simpler models than `Art/Models/Mesh_Weapon_Primary/Secondary.fbx`, not copies. To use one, swap the mesh in a weapon prefab yourself.
- The two totems (`Mesh_Statue_Big/Wall.fbx`) use textures from `FPS/Art/Textures/Props/`. Those textures are only used by the totems.

## Notes

- Always move or rename assets **inside the Unity editor**, not in Explorer, so the `.meta` files (which hold asset IDs) stay paired with their assets.
- The "Unused" labels come from scanning asset references. No script in `FPS/` loads assets by name (no `Resources.Load` or Addressables), so an unreferenced file really is unused. Still, check with **right-click > Find References In Scene** or a search before deleting anything.
- `FPS/Tutorials/` is git-ignored. It's still on disk, and the Unity tutorial package (`com.unity.learn.iet-framework`) may show its window on startup. Delete the folder and remove that package from **Window > Package Manager** to get rid of it completely.
