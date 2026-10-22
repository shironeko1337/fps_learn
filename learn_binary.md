
## Binary files in `Assets/`

These are every binary file under `Assets/` except the ignored tutorials: 186 files, not counting `.meta` files. Git can't diff or merge them, so when two people change the same one, one person's version has to win. If the repo gets large, consider tracking these types with [Git LFS](https://git-lfs.com/).

Paths are relative to `Assets/`. The **Used in** column comes from scanning which scenes, prefabs and materials reference each file:

- **Game**: used by a build scene (MainScene or the menus).
- **Secondary**: used only via SecondaryScene, which isn't in the build.
- **Unused**: nothing references it.

### Baked scene data: `FPS/Scenes/`

Unity regenerates these when you bake lighting or the NavMesh. After a re-bake, commit them together with the scene.

| File | Used for | Used in |
|---|---|---|
| `MainScene/LightingData.asset` | Baked lightmaps and light probes for MainScene. | Game |
| `MainScene/ReflectionProbe-0.exr`, `ReflectionProbe-2.exr` | Baked reflection cubemaps (referenced through `LightingData.asset`). | Game |
| `MainScene/NavMesh-NavMeshSurface.asset` | Baked walkable area for enemies in MainScene. | Game |
| `MainScene/NavMesh-NavMeshSurface 1.asset`, `2.asset`, `3.asset` | Old bakes; nothing references them. | Unused |
| `SecondaryScene/LightingData.asset`, `ReflectionProbe-0.exr` | Baked lighting and reflections for SecondaryScene. | Secondary |
| `SecondaryScene/NavMesh-Level.asset` | Baked walkable area for SecondaryScene. | Secondary |

### Enemy rigs and animations: `FPS/Animation/` (FBX)

| File | Used for | Used in |
|---|---|---|
| `Rigs/HoverBot.fbx` | HoverBot model, skeleton and Avatar. | Game |
| `Rigs/Turret.fbx` | Turret model, skeleton and Avatar. | Game |
| `HoverBot/HoverBot@Alerted.fbx`, `@Hurt`, `@IdleClosed`, `@RunFoward`, `@RunFowardClosed`, `@Shoot` | HoverBot clips, played by `HoverBot_AnimationController`. | Game |
| `HoverBot/HoverBot@Idle.fbx` | Open-shell idle clip; the controller doesn't use it. | Unused |
| `Turret/Turret@ActivatedHurt.fbx`, `@AttackIn`, `@AttackLoop`, `@AttackOut`, `@Idle`, `@PassiveHurt` | Turret clips, played by `Turret_AnimationController`. | Game |

### Models: `FPS/Art/Models/` (FBX, all static)

| File | Used for | Used in |
|---|---|---|
| `Mesh_Weapon_Primary.fbx` | Blaster and Shotgun models. | Game |
| `Mesh_Weapon_Secondary.fbx` | Launcher and its disc projectile. | Secondary |
| `Mesh_Projectile.fbx` | Blaster, Shotgun, HoverBot and Turret projectiles. | Game |
| `Mesh_ExplosionDisk.fbx` | Shockwave disk in explosion effects. | Game |
| `Mesh_Health_Pickup.fbx` | Health pickup. | Game |
| `Mesh_Jetpack.fbx` | Jetpack pickup. | Secondary |
| `Mesh_WireframeRangeCage.fbx` | Range cage in the `VFX_DiskRange` effect. | Secondary |
| `Mesh_Sun.fbx` | Sun disc on the directional light prefab. | Game |
| `Mesh_ReachPoint_Objective.fbx` | "Reach this point" objective marker (its prefab isn't placed anywhere). | Unused |

### Textures: `FPS/Art/Textures/`

Many models use a standard material set: **Albedo** (base colour), **AO** (ambient occlusion shading), **MetallicSmooth** (shininess), **Normal** (surface bumps) and **Emissive/Emission** (glowing parts).

#### Environment

| File | Used for | Used in |
|---|---|---|
| `Backgrounds/UnityTutorial_Skybox05_4k.png` | Level skybox (`Skybox.mat`). | Game |
| `Backgrounds/Unity_skyboxBG01.png` | Intro menu background. | Game |
| `Backgrounds/Unity_skyboxBG02.png` | Win and lose screen background. | Game |

#### Enemies

| File | Used for | Used in |
|---|---|---|
| `Characters/HoverBot_Body_Albedo.png`, `_AO.png`, `_MetallicSmooth.png` | HoverBot body material. | Game |
| `Characters/HoverBot_Light__Emission.png` | HoverBot eye and glass glow. | Game |
| `Characters/HoverBot_Glass_Albedo.png` | `HoverBot_Glass.mat`, which nothing uses. | Unused |
| `Characters/Turret_Body_Albedo.png`, `_AO.png`, `_MetallicSmooth.png` | Turret body material. | Game |
| `Characters/Turret_Light_Emission.png` | Turret eye glow. | Game |
| `Characters/Turret_Glass_Albedo.png` | `Turret_Glass.mat`, which nothing uses. | Unused |

#### Weapons

| File | Used for | Used in |
|---|---|---|
| `Weapons/Blaster_Albedo.png`, `_AO.png`, `_Emissive.png`, `_MetallicSmooth.png`, `_Normal.png` | Blaster and Shotgun materials. | Game |
| `Weapons/Launcher_Albedo.png`, `_AO.png`, `_Emissive.png`, `_MetallicSmooth.png`, `_Normal.png` | Launcher material. | Secondary |

#### Pickups

| File | Used for | Used in |
|---|---|---|
| `Pickups/HealthPickup_Albedo.png`, `_AO.png`, `_Emissive.png`, `_Normal.png` | Health pickup material. | Game |
| `Pickups/Jetpack_Albedo.png`, `Jetpack_AO.png`, `Jetpack__MetallicSmooth.png`, `Jetpack__Normal.png` | Jetpack pickup material. | Secondary |

#### Props (used only by ModAssets totems)

| File | Used for | Used in |
|---|---|---|
| `Props/StatueBig_Albedo.png`, `_AO.png`, `_Emissive.png`, `_MetallicSmooth.png`, `_Normal.png` | `ModAssets/Materials/Totems/M_StatueBig.mat`. | Unused |
| `Props/WallStatue_Albedo.png`, `_AO.png`, `_Emissive.png`, `_MetallicSmooth.png`, `_Normal.png` | `ModAssets/Materials/Totems/M_StatueWall.mat`. | Unused |

#### HUD and menus

| File | Used for | Used in |
|---|---|---|
| `UI/Crosshair_Blaster_Center.png` | Default HUD crosshair. | Game |
| `UI/Crosshair_Hexa.png` | Blaster crosshair. | Game |
| `UI/Crosshair_Shotgun_Circle.png` | Shotgun crosshair. | Secondary |
| `UI/Crosshair_Drop.png` | Launcher crosshair. | Secondary |
| `UI/Crosshair_Blaster_Side.png` | Not referenced. | Unused |
| `UI/HUD_Weapon03.png` | Blaster icon in the weapon slots. | Game |
| `UI/HUD_Weapon02.png` | Shotgun icon. | Secondary |
| `UI/HUD_Weapon01.png` | Launcher icon. | Secondary |
| `UI/HUD_LifeBar_Icon.png`, `HUD_Jetpack_Icon.png` | Health and jetpack-fuel icons. | Game |
| `UI/HUD_Key_Bg.png` | Ammo counter key-hint background. | Game |
| `UI/HUD_KeyScroll_Icon.png` | Not referenced. | Unused |
| `UI/Stance_Stand_Icon.png`, `Stance_Crouch_Icon.png` | Player stance indicator. | Game |
| `UI/Stance_Sprint_Icon.png` | Not referenced. | Unused |
| `UI/Notification_Enemy_Icon.png`, `Notification_MainObj_Icon.png` | Objective popup and compass objective icons. | Game |
| `UI/Radar_Enemy_Icon.png`, `Radar_Enemy_IconBg.png` | Enemy markers on the compass. | Game |
| `UI/ControlScheme_PopUp.png`, `CloseBtn_Icon.png` | Controls help popup and its close button. | Game |
| `UI/Gradiants.png` | Message banner gradient. | Game |
| `UI/TEX_White_16x16.png`, `TEX_Black_16x16.png` | Plain fills for bars and panels (including enemy health bars). | Game |
| `UI/Transparent_Center.png` | Screen vignette in the HUD and menus. | Game |

#### Visual effects

| File | Used for | Used in |
|---|---|---|
| `VFX/Flare.png` | Muzzle flash, impact flare and pickup sparkles. | Game |
| `VFX/Dot.png` | Gun glow cylinders and health glow. | Game |
| `VFX/ExplosionRing.png` | Explosion shockwave (also on the reach-point material). | Game |
| `VFX/ThickSmoke.png`, `ThickSmoke_N.png` | Angry-enemy smoke (`_N` is its normal map). | Game |
| `VFX/OverheatSmoke.png` | Weapon overheat smoke. | Game |
| `VFX/HealthPickUp.png` | Health pickup glow. | Game |
| `VFX/JetPackPulse.png` | Jetpack trail. | Game |
| `VFX/Falloff.psd`, `Shadow.psd` | Fake blob shadow under enemies and pickups. | Game |
| `VFX/Symbol_ExclamationMark.tga` | "!" above an enemy that spots the player. | Game |
| `VFX/Symbol_Speedlines_Atlas.tga` | Speed lines effect. | Game |
| `VFX/ShrapnelTriangle.png` | Explosion shrapnel. | Secondary |
| `VFX/WireframeCage.png` | Range cage effect. | Secondary |
| `VFX/ProjectileRamp.png`, `Projectile_Trail.bmp` | Coloured projectile and trail materials that no prefab uses. | Unused |
| `VFX/ExplosionRange.bmp` | `DiskExplosionRange.mat`, which no prefab uses. | Unused |
| `VFX/Projectile.bmp`, `Symbol_Anger01.tga`, `Symbol_Speedlines.tga` | Not referenced. | Unused |

### Audio: `FPS/Audio/` (WAV/MP3)

#### Music and ambience

| File | Used for | Used in |
|---|---|---|
| `Music/Wind Ambience.mp3` | Level background ambience (`GameManager`). | Game |

#### Player

| File | Used for | Used in |
|---|---|---|
| `SFX/Player/Footstep.wav`, `Jump.wav`, `Land.wav` | Walking, jumping and landing. | Game |
| `SFX/Player/Land_Damage.wav` | Landing from a fall that causes damage. | Game |
| `SFX/Player/Jetpack_Use.wav` | Jetpack thrust loop. | Game |
| `SFX/Player/Damage_Tick2.mp3` | Hit tick when the player damages an enemy (`DamageTick` on both enemy prefabs). | Game |
| `SFX/Player/Damage_Tick.mp3` | Not referenced. | Unused |

#### Weapons

| File | Used for | Used in |
|---|---|---|
| `SFX/Weapons/Blaster_Shot.wav` | Blaster firing. | Game |
| `SFX/Weapons/Weapon_Cooling.wav` | Weapon cooling after overheating. | Game |
| `SFX/Weapons/Shotgun_Shot.wav` | Shotgun firing. | Secondary |
| `SFX/Weapons/Launcher_Charge_BuildUp.wav`, `Launcher_Charge_Loop.wav`, `Launcher_Release.wav` | Launcher charging and firing. | Secondary |
| `SFX/Weapons/Launcher_Explosion.wav` | Launcher disc exploding. | Secondary |

#### Enemies

| File | Used for | Used in |
|---|---|---|
| `SFX/Enemies/Hoverbot_Alert.wav`, `Hoverbot_Attack.wav`, `Hoverbot_Movement.wav` | HoverBot spotting the player, shooting and its hover loop. | Game |
| `SFX/Enemies/Hoverbot_Death.wav` | HoverBot explosion. | Game |
| `SFX/Enemies/Turret_Alert.mp3`, `Turret_Attack.wav` | Turret spotting the player and shooting. | Game |
| `SFX/Enemies/Turret_Death.wav` | Turret explosion. | Game |

#### Pickups

| File | Used for | Used in |
|---|---|---|
| `SFX/Pickups/Pickup_Health.wav` | Health pickup. | Game |
| `SFX/Pickups/Pickup_Weapon_Small.wav` | Blaster and Shotgun pickup. | Game |
| `SFX/Pickups/Pickup_Weapon_Medium.wav` | Launcher pickup. | Secondary |
| `SFX/Pickups/Pickup_Jetpack.wav` | Jetpack pickup. | Secondary |

#### UI

| File | Used for | Used in |
|---|---|---|
| `SFX/UI/Notification_Chime.wav`, `Notification_Snap.wav` | Objective popups. | Game |
| `SFX/UI/Notification_End.wav` | Victory sound (`GameManager`). | Game |

### Mod models: `ModAssets/Models/` (FBX, all static, all unused)

| File | Used for |
|---|---|
| `AncientProps/Archway.fbx`, `MoaiHead.fbx`, `Obelisk.fbx`, `Pillar.fbx`, `Pyramid.fbx` | Ancient-themed props (no prefab; drag the model in directly). |
| `FoodProps/Cookie.fbx`, `Donut.fbx`, `GummyBear.fbx`, `Licorice.fbx`, `Lollipop.fbx` | Candy-themed props (no prefab). |
| `NatureProps/Cactus.fbx`, `FallenTree.fbx`, `FirTree.fbx`, `FlowerBush.fbx`, `Mushroom.fbx`, `PalmTree.fbx` | Nature props (no prefab). |
| `SpaceProps/Asteroid.fbx`, `Crystal.fbx`, `Rocket.fbx`, `SatelliteDish.fbx` | Space props (no prefab). |
| `HatMeshes/PartyHatMesh.fbx`, `GlassesMesh.fbx`, `WizardHatMesh.fbx` | Hats for enemies, via `ModAssets/Prefabs/Hat Prefabs/PartyHat` and `WizardHat`. |
| `Totems/Mesh_Statue_Big.fbx`, `Mesh_Statue_Wall.fbx` | Statues, via `ModAssets/Prefabs/Totems/`. |
| `BillboardMeshes/billboard_3D_basic.fbx`, `billboard_3D_tall.fbx` | Billboard frames, via `ModAssets/Prefabs/BillboardPrefabs/`. |
| `Weapons/Mesh_Weapon_1.fbx`, `Mesh_Weapon_2.fbx` | Simple alternative gun models (no prefab). |

### Mod textures: `ModAssets/Textures/` (PNG, all unused)

| File | Used for |
|---|---|
| `billboard_2D_basic.png`, `billboard_2D_tall.png`, `billboard_2D_wide.png` | Example images for the billboard signs (not referenced by any material). |
| `billboard_2D_basic_mask.png`, `billboard_2D_tall_mask.png`, `billboard_2D_wide_mask.png` | Masks for those images (not referenced). |
| `rainbowtexture.png` | Rainbow trail materials. |

### Fonts: `TextMesh Pro/Fonts/`

| File | Used for | Used in |
|---|---|---|
| `LiberationSans.ttf` | Source font for TextMesh Pro's default font asset (the fallback UI font). | Game |

