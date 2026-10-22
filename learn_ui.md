
## UI: `Assets/FPS/Scripts/UI/` and `Prefabs/UI/`

This guide explains how the game's screens and HUD are built, which files act as templates, and how UI is drawn compared with the 3D scene.

### The three kinds of UI in this project

| Kind | Where it's built | Drawn as |
|---|---|---|
| **Menu screens** (start menu, win, lose) | Directly inside `Scenes/IntroMenu.unity`, `WinScene.unity` and `LoseScene.unity`. Each scene has its own Canvas, buttons and EventSystem. | Flat overlay |
| **In-level HUD and pause menu** | `Prefabs/UI/GameHUD.prefab` and `Prefabs/UI/InGameMenu.prefab`, both nested inside `Prefabs/GameManager.prefab`, which each level places once. | Flat overlay |
| **UI attached to objects in the world** | A world-space Canvas inside `Prefabs/Enemies/Enemy_HoverBot.prefab` and `Enemy_Turret.prefab` (the floating health bar). | A real object in the 3D scene |

### Building blocks every UI uses

These are Unity's built-in UI pieces (from the uGUI package, `com.unity.ugui`). They aren't files in this project:

| Piece | What it does |
|---|---|
| **Canvas** | The root of a UI tree. It decides *how* everything under it is drawn (see [Is UI a flat layer?](#is-ui-a-flat-layer)). Several canvases are drawn in order of their **Sort Order**. |
| **Canvas Scaler** | Sets a reference resolution (960×540 for the HUD and menus) and scales the layout to the real screen size. |
| **Graphic Raycaster** | Lets the canvas receive mouse clicks. Without it, buttons can't be clicked. |
| **EventSystem** | One per scene. It turns mouse, keyboard and gamepad input into UI clicks and selection. `MainScene` has one, and each menu scene has its own. |
| **RectTransform** | The 2D Transform every UI object has: anchors, position, size and pivot. |
| **Image**, **TextMeshProUGUI**, **Button**, **Toggle** | The visible parts. |
| **Vertical/Horizontal Layout Group** | Positions its children automatically in a column or a row. |
| **RectMask2D** | Hides anything that goes outside its rectangle. The compass uses it to clip its scrolling letters. |
| **Canvas Group** | Fades or disables a whole branch at once. The end-of-game fade and the ammo counters use it. |

### Is there a template?

Unity itself has no template file for UI. You start from the **GameObject > UI** menu (Canvas, Image, Text – TextMeshPro, Button…). Adding the first UI element creates a Canvas and an EventSystem for you.

In this project, **`Prefabs/UI/GameHUD.prefab` is the template for the in-level HUD**. `GameManager.prefab` is only a wrapper that bundles the managers together with the HUD, the pause menu and the fade screen:

```
GameManager.prefab
 ├─ GameFlowManager, ObjectiveManager, ActorsManager, AudioManager (Game)
 ├─ InGameMenuManager (UI)                  ← opens/closes the pause menu
 ├─ EndGameFadeCanvas  (Canvas + black Image)  ← GameFlowManager fades this in at the end
 ├─ GameHUD.prefab     (nested)             ← the whole HUD
 └─ InGameMenu.prefab  (nested)             ← pause menu: look sensitivity, shadows, invincibility, framerate counter, controls
```

The other prefabs in `Prefabs/UI/` are **small templates that scripts copy at runtime**, each one into a layout group or the compass:

| Prefab | Copied by | Into | When |
|---|---|---|---|
| `AmmoCounter.prefab` | `WeaponHUDManager` | the ammo panel on the right | Each time the player gets a weapon |
| `ObjectiveToastPrimary.prefab`, `ObjectiveToastSecondary.prefab` | `ObjectiveHUDManager` | `ObjectivesRect` (top-left) | Each time an objective starts (primary for required, secondary for optional) |
| `NotificationToast.prefab` | `NotificationHUDManager` | `NotificationsRect` (bottom-left) | On notifications like "2 enemies to kill left" |
| `GameMessageGeneric.prefab` | `DisplayMessageManager` | `DisplayMessageRect` (middle) | On `DisplayMessageEvent` (objective titles, win message) |
| `CompassMarkerDirection.prefab` | `Compass` | the compass bar | N, NE, E… letters at start |
| `CompassMarkerEnemy.prefab`, `CompassMarkerObjective.prefab` | `CompassElement` on an enemy or objective | the compass bar | When that enemy or objective appears |
| `WinGameMessage.prefab` | nothing | — | Unused (older version of the win message) |

### Building the in-level UI from scratch (without `GameManager.prefab`)

If you removed `GameManager.prefab` from a level, these are the pieces to put back. Start with the managers, because the HUD scripts look them up.

1. **Managers.** On an empty object: `GameFlowManager`, `ObjectiveManager`, `ActorsManager` and `AudioManager` (from `Scripts/Game/Managers/`).
2. **The HUD.** Drag in `Prefabs/UI/GameHUD.prefab`. That's the quickest way, because the prefab already wires every script to its Images and to the small prefabs above. To build it fully by hand instead:
   1. Create a Canvas: *Screen Space – Overlay*, Canvas Scaler *Scale With Screen Size* at 960×540.
   2. Add the HUD scripts from `Scripts/UI/` to an object. Each one controls one area:

      | Script | Needs in the Inspector |
      |---|---|
      | `PlayerHealthBar` | an Image set to *Filled* |
      | `StanceHUD` | an Image plus the stand and crouch sprites |
      | `JetpackCounter` | a fill Image and a Canvas Group (to hide the bar until the jetpack is unlocked) |
      | `CrosshairManager` | the crosshair Image |
      | `WeaponHUDManager` | an ammo panel with a Vertical Layout Group, plus `AmmoCounter.prefab` |
      | `ObjectiveHUDManager` | a panel, plus the two objective toast prefabs |
      | `NotificationHUDManager` | a panel, plus `NotificationToast.prefab` |
      | `DisplayMessageManager` | a rect, plus `GameMessageGeneric.prefab` |
      | `Compass` | a masked bar, plus `CompassMarkerDirection.prefab` |
      | `FeedbackFlashHUD` | a full-screen flash Image and a vignette Image, on their own canvas below the HUD |
      | `FramerateCounter` | a TextMeshPro text |

   3. Lay the areas out with anchors (see below).
3. **The end-of-game fade.** A separate Canvas with a full-screen black Image and a Canvas Group. Put the Canvas Group into `GameFlowManager.EndGameFadeCanvasGroup`, or the game will throw an error when it ends.
4. **The pause menu.** `Prefabs/UI/InGameMenu.prefab` plus `InGameMenuManager` on an object, with its `MenuRoot` field set to the menu.
5. **An EventSystem** in the scene (*GameObject > UI > Event System*), so the pause menu's buttons can be clicked.

The menu scenes don't need any of this. They only need a Canvas, buttons with `LoadSceneButton` (which loads a scene by name), `MenuNavigation` (for keyboard and gamepad selection), and an EventSystem.

### How elements are placed on the screen

Three mechanisms decide where things go:

1. **Anchors** pin an element to a point or an edge of its parent, as fractions from 0 to 1.
   - Crosshair `(0.5, 0.5)`: always in the centre.
   - Compass `(0.5, 1)`: top centre, 25 px down.
   - `FramerateCounter` `(1, 1)`: top-right corner.
   - `BottomLeftcorner` `(0, 0)`: bottom-left corner.
   - If the min and max anchors differ, the element **stretches**. `FlashImage` goes from `(0,0)` to `(1,1)`, so it always covers the whole screen.
2. **The Canvas Scaler** designs everything at 960×540 and scales it to the real screen, so the HUD keeps its proportions at any resolution.
3. **Layout groups** place their children automatically:

   ```
   BottomLeftcorner        VerticalLayoutGroup   → stacks its children top to bottom
    ├─ JetpackCounter
    └─ HorizontalContent   HorizontalLayoutGroup → puts its children side by side
        ├─ PlayerHealth    (Background, FillImageHealth, HealthIcon)
        └─ Stance          (StanceIcon)
   ```

   This is why scripts can create copies of a prefab (ammo counters, objective toasts) without setting any position. The layout group arranges each new child.

### How scripts drive the UI

UI scripts never create the layout. They only **push game data into elements the prefab already contains**, or create copies of small UI prefabs.

**Example 1: the health bar, updated every frame.** `PlayerHealthBar.cs`:

```csharp
public Image HealthFillImage;                   // set in the Inspector → FillImageHealth
void Start()  { m_PlayerHealth = FindAnyObjectByType<PlayerCharacterController>().GetComponent<Health>(); }
void Update() { HealthFillImage.fillAmount = m_PlayerHealth.CurrentHealth / m_PlayerHealth.MaxHealth; }
```

`FillImageHealth` is an Image of type *Filled, Horizontal*, so a `fillAmount` of `0.7` draws 70% of the bar from the left.

**Example 2: an element created when something happens.** `WeaponHUDManager.cs` subscribes to `PlayerWeaponsManager.OnAddedWeapon`. When the player gets a weapon, it runs `Instantiate(AmmoCounterPrefab, AmmoPanel)` and calls `AmmoCounter.Initialize(weapon, slot)`, which fills in the weapon's icon and slot number. The panel's Vertical Layout Group puts the new counter in place.

**Example 3: UI driven by events.** `ObjectiveHUDManager` listens for `ObjectiveUpdateEvent`, and `DisplayMessageManager` listens for `DisplayMessageEvent`. Neither knows which objective or system sent the event. See the Events table in `learn_game.md`.

Because of this split, you can move, resize or restyle any HUD element in the prefab without touching code, as long as the Inspector fields stay connected.

### Is UI a flat layer?

It depends on the Canvas's **Render Mode**.

**Screen Space – Overlay** is used by the HUD, the pause menu, the fade screen and all menu scenes. Here the UI is a flat layer that doesn't interact with the 3D scene:

- It's drawn **after** the camera has finished the 3D scene, including URP post-processing. So bloom, vignette and colour grading don't affect it, and it's always on top.
- It has **no depth**. 3D objects can't hide it, and it can't hide 3D objects except by covering pixels.
- It has **no lighting or shadows**. Scene lights don't affect it, and it casts no shadows.
- It's **not part of physics**. Projectiles, raycasts and NavMesh ignore it completely. Clicks are handled separately, through the Graphic Raycaster and the EventSystem.
- Between canvases, only the **Sort Order** matters. `FeedbackFlashCanvas` (order 0) is drawn below `HUD` (order 1), so the red damage flash never covers the health bar.
- It **does still cost time**. Unity rebuilds a canvas's meshes when something inside it changes, so very busy canvases cost CPU time. That doesn't change how anything else is drawn.

**World Space** is used by the enemy health bars. Here the canvas is a **real object in the 3D scene**:

- It has a position, rotation and size in the world. `WorldspaceHealthBar.cs` turns it to face the camera every frame and hides it while the enemy is at full health.
- It's drawn **with the scene's transparent objects**, so walls can hide it and post-processing affects it.
- It's still **unlit** and casts no shadows, and physics still ignores it.

**Screen Space – Camera** (not used here) sits in between. The UI is flat but placed at a set distance in front of a camera, so it can be affected by that camera's post-processing and can be sorted against 3D objects.

**The first-person gun is not UI.** It's a normal 3D model drawn by a second camera (`WeaponCamera` in `PlayerWeaponsManager`) on its own layer, so it never sinks into walls. It looks like an overlay, but it's lit, shaded and post-processed like the rest of the scene.

### `Scripts/UI/` at a glance

All of these compile into `fps.UI`, which can use Game, Gameplay and AI (see `learn_game.md`).

| Script | Role | Attached to |
|---|---|---|
| `PlayerHealthBar`, `StanceHUD`, `JetpackCounter`, `CrosshairManager`, `FeedbackFlashHUD`, `FramerateCounter` | Show one piece of player state each frame | GameHUD |
| `WeaponHUDManager`, `AmmoCounter`, `FillBarColorChange` | Weapon slots and ammo bars | GameHUD, AmmoCounter |
| `ObjectiveHUDManager`, `ObjectiveToast`, `NotificationHUDManager`, `NotificationToast`, `DisplayMessageManager`, `DisplayMessage` | Objectives, notifications and messages | GameHUD and its small prefabs |
| `Compass`, `CompassElement`, `CompassMarker` | Compass bar; `CompassElement` goes on anything that should show up on it | GameHUD, enemies, objectives |
| `WorldspaceHealthBar` | Floating enemy health bar | Enemy_HoverBot, Enemy_Turret |
| `InGameMenuManager`, `TakeScreenshot` | Pause menu and its options | GameManager, InGameMenu |
| `LoadSceneButton`, `MenuNavigation`, `ToggleGameObjectButton` | Menu scene buttons and keyboard/gamepad navigation | IntroMenu, WinScene, LoseScene |
| `UITable` | Lays out its children as a table | GameHUD (`DisplayMessageRect`) |
| `EnemyCounter` | Shows how many enemies are left | (unused) |
