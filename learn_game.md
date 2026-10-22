
## Game and Gameplay code: `Assets/FPS/Scripts/Game/` and `Scripts/Gameplay/`

This guide covers both folders. **Game** holds the shared building blocks: health, damage, weapons, objectives and the event system. **Gameplay** builds the player and the things the player touches on top of those blocks: movement, input, weapon handling, pickups, projectiles and concrete objectives.

Paths are relative to `Assets/FPS/Scripts/`. The **Attached to** column lists the prefabs or scenes that have the script as a component, found by scanning the project:

- **(code only)**: a helper class or base class that other scripts use. It isn't added to objects directly.
- **(unused)**: nothing in the project uses it.

Each script folder is compiled as its own assembly, set up by an `.asmdef` file in the folder. How `.asmdef` and `.asmref` files work, and how Unity builds the project in general, is covered in `learn_build.md`.

### Gameplay builds on Game

Gameplay sits **on top of** Game: Gameplay code can use Game code, but Game code can't use Gameplay code. AI and UI sit on top of Game in the same way.

**What "use" means.** Using another script's code means naming its types in your code: declaring a variable of that type, calling its methods, reading its properties, or inheriting from its class. For example, `HealthPickup` (Gameplay) does all of these with `Health` (Game):

```csharp
class HealthPickup : Pickup                         // inherit from a class
{
    Health playerHealth = player.GetComponent<Health>(); // declare a variable of a type
    playerHealth.Heal(HealAmount);                       // call a method
}
```

To compile code like that, the compiler has to be able to see the assembly that defines the type.

**Who can see whom.** Each folder's `.asmdef` file makes it a separate assembly. An assembly only sees the assemblies listed in its `"references"`. Having the `.asmdef` files side by side isn't enough.

| Assembly | Folder | `references` (what it can see) |
|---|---|---|
| `fps.Game` | `Game/` | Unity.ProBuilder |
| `fps.Gameplay` | `Gameplay/` | **fps.Game**, Unity.InputSystem |
| `fps.AI` | `AI/` | **fps.Game** |
| `fps.UI` | `UI/` | **fps.Game**, fps.Gameplay, fps.AI, Unity.TextMeshPro, Unity.InputSystem |

```
                 fps.UI                  (top: uses everything below it)
               ↙        ↘
     fps.Gameplay      fps.AI            (siblings: never use each other)
               ↘        ↙
                fps.Game                 (bottom: uses none of the above)

   arrow = "uses"
```

`fps.Game` doesn't list `fps.Gameplay`. If a script in `Game/` mentions `PlayerCharacterController`, compiling fails with *"The type or namespace 'PlayerCharacterController' could not be found"*. Adding the reference wouldn't fix it either. Gameplay already references Game, and **Unity doesn't allow circular references**, because each assembly has to finish compiling before the ones that use it. So the one-way direction is enforced by the compiler, not just a style rule.

### What `fps.Game` describes

`fps.Game` isn't the game loop itself. Unity runs the loop: every frame it calls `Update()` on every active component, wherever its code lives. What `fps.Game` describes is **the rules of the game**: what a match is, how combat works, and how parts of the game report what happened. You could read only `Game/` and know how this game is won, lost and fought, without knowing who the player is or what the enemies look like.

1. **The life of a level** (`Managers/GameFlowManager`, `Managers/ObjectiveManager`, `Shared/Objective`):
   - A level is a set of **objectives**. Each objective has a title and description, is either required or optional, and is either done or not done.
   - The level is **won** when no required objective is left unfinished.
   - It's **lost** when the player dies.
   - Either way, the screen and sound fade out and the matching end scene (`WinScene` or `LoseScene`) loads.
2. **The combat model** (`Actor`, `Managers/ActorsManager`, `Shared/Health`, `Shared/Damageable`, `Shared/DamageArea`):
   - Anything that fights is an **actor** on a numbered team. Actors on the same team are friends.
   - Anything that can be hurt has **health** between 0 and a maximum. It can be damaged, healed, made invincible or killed, and it dies exactly once.
   - **Damage** arrives through hit zones that can multiply it (weak spots). Damage you do to yourself is reduced.
   - **Explosions** hurt every target in a radius once, with less damage further from the centre.
3. **The weapon model** (`Shared/WeaponController`, `Shared/ProjectileBase`):
   - A **weapon** fires once per press, automatically while held, or after charging.
   - Its **ammo** works like heat: it drains with each shot and refills on its own after a pause.
   - Each shot fires one or more **projectiles** spread inside a cone.
   - A **projectile** knows who fired it, from where, and at what charge level.
4. **What can happen** (`Events`, `Managers/EventManager`):
   - The list of game events is a vocabulary of everything worth announcing: the player died, an enemy was killed, something was picked up, an objective changed, all objectives are done, show a message.
   - Any part of the game can announce one of these events or listen for it.
5. **Shared services** (`AudioUtility`, `Managers/AudioManager`, `MeshCombiner`, `TimedSelfDestruct`, `MinMaxParameters`, `DebugUtility`):
   - Sound playback routed through the mixer, and master volume.
   - Merging level meshes for faster drawing.
   - Objects that delete themselves after a set time.
   - Min/max value ranges, and editor-only error messages.

What `fps.Game` does **not** describe: how the player moves or aims, what enemies do, what specific objectives, pickups or projectiles exist, or what appears on screen. Those details are in the assemblies that build on Game.

### Why the author organized it this way

1. **The player and the enemies share the same parts.** Health, damage, team membership (`Actor`) and weapons (`WeaponController`) work the same for both. If those were in Gameplay, AI would have to reference Gameplay, so enemy code would depend on player code. Putting the shared parts in Game makes Gameplay and AI **siblings** that both build on the same base and never depend on each other.
2. **Each part only knows what it needs.** `GameFlowManager` only needs to know *that* the player died, not *how*, so it can stay in Game and listen for an event. Enemies only need *something* with a `Damageable` to shoot at, not the player class. That keeps each script small, and you can replace one part, such as the player controller, without changing the others.
3. **Mistakes are caught early.** In one big assembly, any script can call any other, and over time everything ends up connected to everything else. With the split, a wrong-direction dependency is a compile error, so the structure can't slowly fall apart.
4. **Faster recompiles.** Unity only recompiles the assembly you changed and the ones that use it. Editing a Gameplay script recompiles `fps.Gameplay` and `fps.UI`, but not `fps.Game` or `fps.AI`.
5. **Easy to extend.** To add a new objective, projectile or weapon, you write a Gameplay class that inherits from the Game base or uses its components. The existing managers, HUD and win condition pick it up without changes. This suits a template whose tutorials and mods ask you to add your own content.
6. **Outside packages stay where they're needed.** Only Gameplay and UI reference the Input System, because only the player reads controls. Game has no input code, so the enemies, which use Game, carry no input code either.

---

## `Game/` (root files)

General utilities and small components.

| File | What it does | Attached to |
|---|---|---|
| `Actor.cs` | Marks an object as a combatant. `Affiliation` is a team number: same number means friendly. `AimPoint` is where attackers aim. Each actor adds itself to `ActorsManager` on start and removes itself when destroyed, which is how AI finds targets. | Player, Enemy_HoverBot, Enemy_Turret |
| `Events.cs` | The list of every game event (see [Events](#events)) and the data each one carries. Each event is a single shared object that's reused, so don't keep a reference to one after handling it. | (code only) |
| `AudioUtility.cs` | Sound helpers. `CreateSFX()` creates a temporary object that plays a clip at a position and deletes itself when the clip ends. `GetAudioGroup()` finds a mixer group by name (`WeaponShoot`, `Pickup`, `Impact`…). `SetMasterVolume()` converts a 0–1 value to decibels. | (code only) |
| `GameConstants.cs` | Names of the old Input Manager axes and buttons. The current input code uses the Input System instead, so these are mostly leftovers. | (code only) |
| `DebugUtility.cs` | Editor-only error messages such as "X expected to find a component Y but none were found". It compiles to nothing in builds. | (code only) |
| `MinMaxParameters.cs` | `MinMaxFloat`, `MinMaxColor` and `MinMaxVector3`: a min/max pair that you blend between with a 0–1 ratio. The charge weapon uses them to scale effects by charge level. | (code only) |
| `TimedSelfDestruct.cs` | Destroys its object after `LifeTime` seconds. | VFX_HealthPickup, Sign_Pyramid_Orange; added by code in `AudioUtility` |
| `MeshCombiner.cs` | When the level starts, merges all the level meshes under `CombineParents` that share a material into one mesh, which makes the level much cheaper to draw. With `UseGrid` on, it merges per grid cell instead of everything at once. | MainScene, SecondaryScene |
| `MeshCombineUtility.cs` | Does the merging for `MeshCombiner`: groups renderers by material, removes ProBuilder components and creates a `Level_Combined` object for each group. | (code only) |
| `IgnoreHitDetection.cs` | An empty marker. Projectiles pass through any collider that has it (checked in `ProjectileStandard.IsHitValid`). | (unused) |
| `ConstantRotation.cs` | Spins an object around its up axis. Pickups don't use it; they rotate in `Pickup.cs` instead. | (unused) |

## `Game/Managers/`

One-per-scene systems. Except for `EventManager`, they all live on `Prefabs/GameManager.prefab`, which each level scene places once.

| File | What it does | Attached to |
|---|---|---|
| `EventManager.cs` | A simple message system. `AddListener<T>(handler)` subscribes to an event type, `Broadcast(evt)` calls every subscriber of that type, and `RemoveListener` unsubscribes. It's a static class, so no object needs to exist for it to work. | (code only) |
| `GameFlowManager.cs` | Decides when the game ends. It listens for `AllObjectivesCompletedEvent` (win) and `PlayerDeathEvent` (lose). It then unlocks the cursor and fades the screen to black while fading out the volume. When the player wins, it also plays `VictorySound` and shows `WinGameMessage`. After the delay, it loads `WinScene` or `LoseScene`. `GameIsEnding` also stops player input. | GameManager |
| `ObjectiveManager.cs` | Keeps a list of all objectives (each one registers itself when created). Every frame, it checks whether any required objective is still unfinished. When none are left, it broadcasts `AllObjectivesCompletedEvent` once. | GameManager |
| `ActorsManager.cs` | Holds the list of all `Actor`s and a reference to the player. AI uses it to find enemies of other teams. | GameManager |
| `AudioManager.cs` | Wraps the project's audio mixer(s) so `AudioUtility` can find groups and set the master volume. | GameManager |

## `Game/Shared/`

The core gameplay components used by the player, the enemies, or both. "Shared" means "used by more than one assembly".

### Health and damage

How damage moves through these components:

```
projectile hits a collider
   └─► Damageable.InflictDamage(damage, isExplosion, source)    on the collider's object
         • ×DamageMultiplier (e.g. a weak spot) unless it's an explosion
         • ×SensibilityToSelfdamage if you hit yourself
         └─► Health.TakeDamage(amount, source)                  on the object or its parent
               • ignored if Invincible; clamped between 0 and MaxHealth
               ├─► OnDamaged(amount, source)   → enemy flinches, HUD flashes, …
               └─► OnDie()  (once, at 0)        → player: game over / enemy: explode
```

| File | What it does | Attached to |
|---|---|---|
| `Health.cs` | Current and maximum health. `TakeDamage()`, `Heal()` and `Kill()` change it and fire `OnDamaged`, `OnHealed` and `OnDie` (only once). `IsCritical()` (below 30% by default) drives the red screen vignette. `CanPickup()` is false at full health, so health packs stay on the floor. | Player, Enemy_HoverBot, Enemy_Turret |
| `Damageable.cs` | The part that gets hit. It goes on colliders and passes damage to the `Health` on the same object or a parent. Splitting the two lets one `Health` have several hit zones with different multipliers. | Player, Enemy_HoverBot, Enemy_Turret |
| `DamageArea.cs` | Explosion damage. `InflictDamageInArea()` finds every `Damageable` within `AreaOfEffectDistance`, damages each `Health` only once, and scales the damage by distance using `DamageRatioOverDistance`. | Projectile_Disc (the Launcher's projectile) |
| `Destructable.cs` | Destroys its object when its `Health` reaches 0. Meant for breakable props. | (unused) |

### Weapons and projectiles

| File | What it does | Attached to |
|---|---|---|
| `WeaponController.cs` | One gun. It holds all of the weapon's settings and the shooting logic (see below). Both the player and the enemies use it; the only difference is who calls `HandleShootInputs()`. | Weapon_Blaster, Weapon_Shotgun, Weapon_Launcher, Enemy_HoverBot, Enemy_Turret, Pickup_Blaster |
| `ProjectileBase.cs` | The base class for anything a weapon fires. `Shoot(weapon)` records who fired it, where from, the gun's velocity and its charge level, then fires `OnShoot`. Child classes and helper components react to `OnShoot` to set themselves up. | (code only; base of `ProjectileStandard`) |

**`WeaponController` in detail**

- **Three shoot types** (`ShootType`):
  - `Manual`: fires once per click (Blaster).
  - `Automatic`: keeps firing while the button is held.
  - `Charge`: holding the button builds `CurrentCharge` from 0 to 1, and releasing fires (Launcher). `AutomaticReleaseOnCharged` fires on its own once charge is full.
- **Ammo works like heat.** `MaxAmmo` is the capacity and each shot uses 1. With `AutomaticReload` on, ammo refills at `AmmoReloadRate` per second, starting `AmmoReloadDelay` seconds after the last shot. While it refills, `IsCooling` is true. `OverheatBehavior` uses that to show steam and play the cooling sound.
- **Optional physical bullets** (`HasPhysicalBullets`): a clip of `ClipSize`, shell casings thrown out of `EjectionPort` (with a pool of `ShellPoolSize` reused shells), and manual reload through an Animator `Reload` trigger. None of the current weapons use this mode.
- **One shot**, in `HandleShoot()`: creates `BulletsPerShot` projectiles, each aimed in a random direction within `BulletSpreadAngle` (this is how the Shotgun spreads). It calls `Shoot(this)` on each projectile, spawns the muzzle flash, plays `ShootSfx` and fires the `OnShoot` event. For a charge weapon, the number of bullets scales with the charge level.
- **Info for the HUD**: `WeaponName`, `WeaponIcon`, and `CrosshairDataDefault` / `CrosshairDataTargetInSight` (the crosshair changes when you aim at something with health).
- **Aiming settings**: `AimZoomRatio` is how much the camera zooms in, `AimOffset` moves the gun while aiming, and `RecoilForce` controls how hard the gun kicks back.
- `MuzzleWorldVelocity` is calculated every frame so projectiles can carry the gun's movement when `InheritWeaponVelocity` is on.

### Objectives

| File | What it does | Attached to |
|---|---|---|
| `Objective.cs` | The base class for all objectives. It has `Title`, `Description`, `IsOptional` and `DelayVisible`. On start it registers with `ObjectiveManager` (through the static `OnObjectiveCreated` event) and shows its title as a message. `UpdateObjective()` and `CompleteObjective()` broadcast `ObjectiveUpdateEvent`, which the HUD and notifications listen for. `IsBlocking()` is true while the objective is required and not done. | (code only; base of the three objectives in `Gameplay/Objectives/`) |

---

## `Gameplay/` (root files)

### The player

The `Player.prefab` is built from these components working together:

```
Player.prefab
 ├─ CharacterController          (Unity)   capsule collision + Move()
 ├─ PlayerInputHandler           Gameplay  reads the Input System
 ├─ PlayerCharacterController    Gameplay  movement, jump, crouch, look, fall damage, death
 ├─ PlayerWeaponsManager         Gameplay  weapon slots, aiming, switching, bob, recoil
 ├─ Jetpack                      Gameplay  extra flight on a second jump press
 ├─ Health, Damageable, Actor    Game      can be hurt, can die, is a team member
 └─ children: PlayerCamera, WeaponCamera, weapon socket and positions
```

| File | What it does | Attached to |
|---|---|---|
| `PlayerCharacterController.cs` | First-person movement (see below). It also tells `ActorsManager` that this object is the player. | Player |
| `Jetpack.cs` | Locked until you pick up the Jetpack (`TryUnlock()`), unless `IsJetpackUnlockedAtStart` is on. In the air, pressing jump **again** and holding it pushes the player up: it cancels gravity, adds `JetpackAcceleration`, and also cancels any falling speed so it takes effect immediately. The full tank lasts `ConsumeDuration` seconds. After `RefillDelay`, it refills faster on the ground than in the air. It also turns the thruster particles and the loop sound on and off. | Player |

**`PlayerCharacterController` in detail** (the main loop runs in `Update()`):

1. **Kill floor.** Falling below `KillHeight` (−50) kills the player instantly.
2. **Ground check.** A capsule-shaped probe just below the feet detects ground, as long as the slope isn't steeper than the CharacterController's slope limit. It snaps the player down onto the ground so they don't skip down slopes. For 0.2 s after a jump, the check is skipped so the jump isn't cancelled straight away.
3. **Landing.** On the frame the player lands, it looks at the fall speed. Between `MinSpeedForFallDamage` and `MaxSpeedForFallDamage`, damage scales from `FallDamageAtMinSpeed` to `FallDamageAtMaxSpeed` and `FallDamageSfx` plays; otherwise `LandSfx` plays. This only applies when `RecievesFallDamage` is on.
4. **Crouch** toggles on key press. The capsule and camera height blend smoothly to `CapsuleHeightCrouching`. Standing back up is blocked if there's a ceiling in the way (it checks that the standing capsule fits).
5. **Look.** Mouse/stick X turns the whole body; Y tilts only the camera, clamped to ±89°. While aiming, look speed is multiplied by `AimingRotationMultiplier`.
6. **Move.**
   - **On the ground:** the player's speed blends toward the target speed (`MaxSpeedOnGround`, ×`SprintSpeedModifier` when sprinting, ×`MaxSpeedCrouchedRatio` when crouched). The movement direction is tilted to follow slopes. `MovementSharpnessOnGround` sets how quickly the player speeds up and slows down. Sprinting stands the player up from a crouch.
   - **Jump:** sets the upward speed to `JumpForce`.
   - **In the air:** input adds `AccelerationSpeedInAir` (for steering in mid-air), horizontal speed is capped at `MaxSpeedInAir`, and gravity pulls down by `GravityDownForce`.
   - **Footsteps** play once per `1 / FootstepSfxFrequency` metres walked.
7. **Collide.** After moving, if the player hit something, the speed going into that surface is removed, so the player slides along walls instead of stopping dead.
8. **Death.** On `Health.OnDie`, it lowers the weapon and broadcasts `PlayerDeathEvent`, and `GameFlowManager` then loads the lose screen.

`OnStanceChanged` fires on crouch and stand, and the HUD's stance icon listens for it.

### Pickups

All pickups extend `Pickup.cs`, which makes the object a trigger that bobs up and down and spins. When the player touches it, it calls `OnPicked()` and broadcasts `PickupEvent`. Each subclass decides what picking it up does, and only then calls `PlayPickupFeedback()` (sound and effect) and destroys itself. **If it can't be used** (for example, at full health), it stays on the floor.

| File | What it does when picked up | Attached to |
|---|---|---|
| `Pickup.cs` | The base: bobbing, spinning, trigger detection, `PickupEvent`, and the sound/effect feedback. It can also be used on its own as a plain item to collect. | ObjectivePickupItem (subclasses go on the other pickups) |
| `HealthPickup.cs` | Heals by `HealAmount`, but only if the player isn't already at full health. | Pickup_Health |
| `WeaponPickup.cs` | Adds `WeaponPrefab` to the player's weapons, and switches to it if the player had no weapon. Fails if the player already has that weapon or all 9 slots are full. It also moves the floating gun model out of the special weapon layer, so walls hide it correctly. | Pickup_Blaster, Pickup_Shotgun, Pickup_Launcher |
| `JetpackPickup.cs` | Calls `Jetpack.TryUnlock()` on the player. Fails if the jetpack is already unlocked. | Pickup_Jetpack |
| `AmmoPickup.cs` | Adds `BulletCount` physical bullets to a weapon the player already has, then broadcasts `AmmoPickupEvent`. It's only useful with the physical-bullets mode. | (unused) |

### Projectiles

| File | What it does | Attached to |
|---|---|---|
| `ProjectileStandard.cs` | The one projectile type, used by every gun (see below). | Projectile_Blaster, Projectile_Shotgun, Projectile_Disc, Projectile_Hoverbot, Projectile_Turret |
| `ProjectileChargeParameters.cs` | On `OnShoot`, sets the projectile's `Damage`, `Radius`, `Speed` and gravity from min/max ranges using the charge level. So a fully charged Launcher disc is bigger, faster and hits harder. | Projectile_Disc |
| `ChargedProjectileEffectsHandler.cs` | On `OnShoot`, scales the projectile's model and tints its colour by the charge level. It copies the material first so each projectile can have its own colour. | Projectile_Disc |

**`ProjectileStandard` in detail**

- **Movement.** The projectile moves `Speed × deltaTime` every frame. Gravity can bend it downward (`GravityDownAcceleration`, used by the Launcher). It's deleted after `MaxLifeTime` seconds.
- **Hit detection.** Every frame it sweeps a sphere of `Radius` from last frame's position to the tip, so fast projectiles can't skip through thin walls. Hits on the shooter's own colliders, on objects with `IgnoreHitDetection`, and on plain triggers are all ignored.
- **Drift toward the crosshair** (player only). The gun's muzzle is lower and to the right of the camera, so a bullet fired straight from the muzzle would miss the crosshair. Over the first `TrajectoryCorrectionDistance` metres, the bullet gradually moves sideways onto the camera's centre line. `0` moves it there instantly and `-1` turns this off. On firing, it also checks the short gap between camera and muzzle, so a gun poking through a wall still hits the wall.
- **On hit.** With an `AreaOfDamage` (a `DamageArea`, used for the Launcher explosion), it damages everything in range. Otherwise it damages the one `Damageable` it hit. Then it spawns `ImpactVfx`, plays `ImpactSfxClip`, and destroys itself.

### Weapon effects

These are optional add-ons on a weapon prefab. Each reads the `WeaponController` on the same object and changes only how the weapon looks or sounds, never how it plays.

| File | What it does | Attached to |
|---|---|---|
| `OverheatBehavior.cs` | Shows how much "heat" (used ammo) the weapon has. The parts using `OverheatingMaterial` glow along `OverheatGradient`, steam particles get thicker, and a cooling sound plays while the weapon refills. It changes the glow with a "property block", so the shared material isn't changed for everyone. | Weapon_Blaster, Weapon_Launcher |
| `WeaponFuelCellHandler.cs` | Moves the gun's visible fuel cells between "full" and "empty" positions as ammo drains. They either move together or empty one after another (`SimultaneousFuelCellsUsage`). | Weapon_Blaster, Weapon_Shotgun, Weapon_Launcher |
| `ChargedWeaponEffectsHandler.cs` | While the Launcher charges, it grows the charging part of the gun, spins the frame, makes orbiting particles circle faster and wider, and plays a build-up sound then a loop sound. With `UseProceduralPitchOnLoopSfx`, it instead raises the loop's pitch with the charge level. | Weapon_Launcher |

### Misc

| File | What it does | Attached to |
|---|---|---|
| `TeleportPlayer.cs` | A debug tool: press **F12** to move the player to this object's position and fully heal them. | DebugTeleportPlayer (not placed in any scene) |
| `PositionBobbing.cs` | Bobs an object up and down. It's the same motion as `Pickup`, without the pickup behaviour. | (unused) |

## `Gameplay/Managers/`

The two "brains" on the player that sit between the raw input and the movement and weapon code.

| File | What it does | Attached to |
|---|---|---|
| `PlayerInputHandler.cs` | The only place that reads the controls. It uses the project-wide action map in `Assets/FPS/InputSystem_Actions.inputactions` (actions `Player/Move`, `Look`, `Jump`, `Fire`, `Aim`, `Sprint`, `Crouch`, `Reload`, `NextWeapon`), plus number keys 1–9 for weapon slots. It locks the mouse cursor when play starts. **Every getter returns "nothing pressed" unless the cursor is locked and the game isn't ending**, which is how the pause menu and the end screen freeze the player. It also handles look sensitivity, axis inversion, a lower sensitivity on WebGL, and fire down/held/released detection. | Player |
| `PlayerWeaponsManager.cs` | Manages the player's weapons (see below). | Player |

**`PlayerWeaponsManager` in detail**

- **Slots.** There are 9 weapon slots. At start it adds every weapon in `StartingWeapons` (MainScene starts with only the Blaster). `AddWeapon(prefab)` creates the weapon under `WeaponParentSocket`, sets the player as its owner, moves the weapon to the `FpsWeaponLayer` layer, and fires `OnAddedWeapon` for the HUD. `HasWeapon()` compares against the original prefab, so you can't pick up the same gun twice.
- **Two cameras.** The guns are on their own layer and drawn by `WeaponCamera` on top of the world. Your gun never clips into a wall you're standing next to, and its field of view can differ (`WeaponFovMultiplier`).
- **Each frame (`Update`)**, while the weapon is raised:
  - it handles manual reload;
  - sets `IsAiming` from the aim button;
  - passes fire down/held/released to `WeaponController.HandleShootInputs()`;
  - on a shot, adds recoil.
  - When not aiming or charging, the mouse wheel or keys 1–9 switch weapons.
  - It also casts a ray from the centre of the screen; if the ray hits anything with `Health`, `IsPointingAtEnemy` becomes true and the crosshair changes.
- **Switching** runs through four states: `PutDownPrevious` (the old gun lowers to `DownWeaponPosition` over `WeaponSwitchDelay`), the active weapon changes, `PutUpNew` (the new gun rises), then `Up` (usable). `Down` means no weapon is out, for example after death.
- **Gun position (`LateUpdate`)** adds up three offsets:
  - **aim:** the gun blends to `AimingWeaponPosition + AimOffset` and the camera zooms in to `AimZoomRatio × DefaultFov`;
  - **bob:** a figure-8 sway that grows with running speed and shrinks while aiming;
  - **recoil:** a fast kick backward, then a slower return.

## `Gameplay/Objectives/`

Concrete objectives built on `Game/Shared/Objective.cs`. Put one in a scene and the level can't be won until it's complete (unless it's `IsOptional`).

| File | Completes when | Attached to |
|---|---|---|
| `ObjectiveKillEnemies.cs` | Enough enemies have died. It listens for `EnemyKillEvent`, which `AI/EnemyManager.cs` sends. With `MustKillAllEnemies`, the target is every enemy in the level (it updates as enemies die); otherwise it's `KillsToCompleteObjective`. It updates the "3 / 5" counter, and when only `NotificationEnemiesRemainingThreshold` or fewer are left it shows notifications like "2 enemies to kill left". If it has no title, it creates one ("Eliminate all the enemies"). | ObjectiveKillEnemies (placed in MainScene and SecondaryScene) |
| `ObjectivePickupItem.cs` | The player touches `ItemToPickup`. It listens for `PickupEvent`, so it completes **even if the pickup itself fails**, for example a health pack at full health. | ObjectivePickupItem, Loot_Jetpack |
| `ObjectiveReachPoint.cs` | The player walks into this object's trigger collider. It then destroys `DestroyRoot`, which also removes its compass marker. | ObjectiveReachPoint (not placed in any scene) |

To make a new kind of objective, inherit from `Objective`, call `UpdateObjective()` as progress changes, and call `CompleteObjective()` when it's done. `ObjectiveManager`, the HUD and the win condition all pick it up automatically.

---

## Events

Every event is defined in `Game/Events.cs` and sent through `Game/Managers/EventManager.cs`.

| Event | Sent by | Listened to by | Data |
|---|---|---|---|
| `PlayerDeathEvent` | `Gameplay/PlayerCharacterController` | `Game/GameFlowManager` (→ lose) | none |
| `AllObjectivesCompletedEvent` | `Game/ObjectiveManager` | `Game/GameFlowManager` (→ win) | none |
| `EnemyKillEvent` | `AI/EnemyManager` | `Gameplay/ObjectiveKillEnemies` | `Enemy`, `RemainingEnemyCount` |
| `PickupEvent` | `Gameplay/Pickup` | `Gameplay/ObjectivePickupItem` | `Pickup` |
| `ObjectiveUpdateEvent` | `Game/Objective` | `UI/ObjectiveHUDManager`, `UI/NotificationHUDManager` | `Objective`, texts, `IsComplete` |
| `DisplayMessageEvent` | `Game/Objective`, `Game/GameFlowManager` | `UI/DisplayMessageManager` | `Message`, `DelayBeforeDisplay` |
| `AmmoPickupEvent` | `Gameplay/AmmoPickup` (unused) | `UI/AmmoCounter` | `Weapon` |
| `GameOverEvent`, `DamageEvent` | nobody | nobody | defined but never used |

Other changes use **C# events on the component** instead of EventManager, because they belong to one object:

- `Health.OnDamaged`, `OnHealed` and `OnDie`;
- `WeaponController.OnShoot`;
- `ProjectileBase.OnShoot`;
- `PlayerWeaponsManager.OnSwitchedToWeapon`, `OnAddedWeapon` and `OnRemovedWeapon`;
- `PlayerCharacterController.OnStanceChanged`;
- `Jetpack.OnUnlockJetpack`.

The UI listens to these directly on the player.

## End-to-end examples

**The player fires the Blaster and kills a HoverBot**

1. `PlayerInputHandler.GetFireInputDown()` returns true.
2. `PlayerWeaponsManager.Update()` → `WeaponController.HandleShootInputs()` → (`Manual`) `TryShoot()`, which checks ammo ≥ 1 and that enough time has passed since the last shot.
3. `HandleShoot()` creates `Projectile_Blaster` and calls `ProjectileBase.Shoot()`. The `ProjectileStandard` on it sets its speed and starts drifting toward the crosshair.
4. `ProjectileStandard.Update()` sweeps a sphere and finds the HoverBot's collider, then calls `OnHit()` → `Damageable.InflictDamage()` → `Health.TakeDamage()`.
5. `Health.OnDamaged` → the AI flinch animation (`AI/EnemyController`). Once health reaches 0, `Health.OnDie` → the enemy explodes, and `AI/EnemyManager` broadcasts `EnemyKillEvent`.
6. `ObjectiveKillEnemies` raises its count and calls `UpdateObjective()` → `ObjectiveUpdateEvent` → the HUD counter and notification update.
7. Meanwhile, `OverheatBehavior` and `WeaponFuelCellHandler` show the used ammo, and after `AmmoReloadDelay` the gun refills.

**Winning or losing**

- **Last enemy dies** → `ObjectiveKillEnemies.CompleteObjective()` → `ObjectiveManager` sees no required objective left → `AllObjectivesCompletedEvent` → `GameFlowManager.EndGame(true)`: win message and sound, fade out, then `WinScene`.
- **Player health reaches 0**, or the player falls below `KillHeight` → `PlayerCharacterController.OnDie()` → `PlayerDeathEvent` → `GameFlowManager.EndGame(false)`: fade out, then `LoseScene`. While the game is ending, `PlayerInputHandler` ignores all input.

## Notes and quirks

- **Unused code:** `Destructable`, `AmmoPickup`, `ConstantRotation`, `PositionBobbing` and `IgnoreHitDetection` aren't on any object. `GameOverEvent` and `DamageEvent` are never sent.
- **Likely bug:** `WeaponController.AddCarriablePhysicalBullets()` uses `Mathf.Max(carried + count, MaxAmmo)`, so the result is never less than `MaxAmmo` and never capped. `Mathf.Min` was probably intended. It doesn't matter yet, because no weapon uses physical bullets.
- `Health.Kill()` reports `MaxHealth` as the damage amount to `OnDamaged`, whatever health was actually left.
- Events in `Events` are single objects that get reused. If you store an event to read later, its fields may have changed by then.
- `GameConstants.cs` describes the old input system. Change controls in `InputSystem_Actions.inputactions`, not there.
