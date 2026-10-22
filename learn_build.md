
## How Unity builds things

Unity turns a project into a game through two separate pipelines that meet at the end:

| | 1. Code pipeline | 2. Asset pipeline |
|---|---|---|
| **Input** | `.cs` files | All other [asset files](#what-counts-as-an-asset-file): textures, models, audio, prefabs, scenes, materials… (binary and text) |
| **Controlled by** | `.asmdef` / `.asmref` files, which decide which `.dll` each script goes into | Importers, whose settings are stored in each file's `.meta` |
| **Output** | Compiled `.dll` files in `Library/ScriptAssemblies/` | Imported instances in Unity's internal format, cached in `Library/` |
| **Runs when** | A script changes | An asset or its `.meta` changes |

**3. Player build** (*File > Build Profiles*): takes the build scenes, follows their references to collect only the assets the game uses, and packs them together with the `.dll` files. See [Making a player build](#making-a-player-build).

The `.asmdef`/`.asmref` files don't contain code themselves. They only group the `.cs` files. Without them, every script still compiles, into one default `Assembly-CSharp.dll`. See [Assemblies](#assemblies).

Everything in `Library/` can be rebuilt from `Assets/`, `Packages/` and `ProjectSettings/`, which is why it's git-ignored.

---

## GUID and fileID

> **Terms used below.** A **class** is a type defined in code (`Camera`, `Material`, `PlayerWeaponsManager`). An **instance** is one concrete value of a class, saved as data in a file: one particular Camera with its own settings, one particular Material. GUIDs and fileIDs point only at **instances**, never at classes. `UnityEngine.Object` is the name of the base **class** of everything Unity can save; it's not a word for "instance".

### Usages of GUID

**GUIDs tie the two pipelines together.** Every asset file gets a GUID in its `.meta` file:

```yaml
# Assets/FPS/Scripts/Game/Shared/Health.cs.meta
fileFormatVersion: 2
guid: e27364eefd6dd0341ba72a3bfc0d9727
MonoImporter:          # importer type and its settings follow
  ...
```

Every reference between assets uses this GUID, never the file path. A GUID always maps to exactly one file. What varies is how many instances that file holds; the [fileID](#usages-of-fileid) picks one of them:

| Asset file | Instances inside | Example reference |
|---|---|---|
| `.cs` | Always exactly one (a `MonoScript`) | A component pointing at its script |
| `.mat`, `.wav`, `.png`, `.asset` | Usually one | A mesh renderer pointing at its material |
| `.fbx` | Many: meshes, an Avatar, animation clips | An Animator state pointing at one clip |
| `.prefab`, `.unity` | Many: every GameObject and component | A scene pointing at a prefab |

#### What counts as an asset file

An **asset file** is any file or folder under `Assets/` (or inside a package under `Packages/`) that Unity doesn't skip. There's no list of allowed formats. The suffix only decides **which importer** handles the file:

| Kind | Suffixes (examples) | Format | Importer |
|---|---|---|---|
| Scripts | `.cs` | Text | MonoImporter |
| Assembly definitions | `.asmdef`, `.asmref` | Text (JSON) | AssemblyDefinitionImporter |
| Unity's own assets | `.prefab`, `.unity`, `.mat`, `.asset`, `.controller`, `.anim`, `.mask`, `.mixer`, `.lighting` | Text (YAML, because the project uses Force Text) | Native or prefab/scene importers |
| Textures | `.png`, `.jpg`, `.tga`, `.psd`, `.bmp`, `.exr` | Binary | TextureImporter |
| Models | `.fbx`, `.obj`, `.blend` | Binary (or text for `.obj`) | ModelImporter |
| Audio | `.wav`, `.mp3`, `.ogg`, `.aiff` | Binary | AudioImporter |
| Video | `.mov`, `.mp4` | Binary | VideoClipImporter |
| Fonts | `.ttf`, `.otf` | Binary | TrueTypeFontImporter |
| Shaders | `.shader`, `.hlsl`, `.shadergraph` | Text | ShaderImporter and others |
| Plain data | `.txt`, `.json`, `.xml`, `.csv`, `.bytes`, `.md` | Any | TextScriptImporter (becomes a `TextAsset`) |
| Packages' own types | e.g. `.inputactions` (Input System) | Text | Importer supplied by the package |
| Anything else | unknown suffix | Any | DefaultImporter: gets a GUID but can't be used for much |
| Folders | — | — | DefaultImporter with `folderAsset: yes` |

What Unity **skips**: the `.meta` files themselves, hidden names starting with `.` (e.g. `.git`), names ending with `~` (e.g. `Samples~`), folders named `cvs`, and `.tmp` files. Skipped files get no `.meta`, no GUID, and can't be referenced.

### How GUID is generated

**When Unity "sees" a file.** Unity keeps a database in `Library/` of every asset file it knows, with its path, GUID, timestamp and a hash of its contents. It compares that database with the disk during an **asset refresh**, which happens:
- when the editor starts;
- when the editor window gets focus back, if *Preferences > Asset Pipeline > Auto Refresh* is on (the default), so files you saved in another program are picked up;
- when you press *Ctrl+R* or code calls `AssetDatabase.Refresh()`.

The refresh scans `Assets/` and the packages, and then, for each asset file:

| Situation | What Unity does |
|---|---|
| New file, **no** `.meta` next to it | Makes a **new random GUID**, writes a `.meta`, imports the file |
| New file **with** a `.meta` (e.g. copied in or pulled from git) | Reads the GUID from the `.meta`, imports the file |
| Known file, timestamp changed | Hashes the contents. If they changed, re-imports it |
| `.meta` gone but file still there | Treats it as a new file, so it gets a new GUID |

**The GUID isn't a hash of the path or the contents.** It's a random 128-bit number written as 32 hex characters, made once and then kept only in the `.meta`. You can see this from how it behaves:
- Delete a `.meta` and let Unity recreate it, and the file gets a **different** GUID, even though its path and contents didn't change. Every reference to the old GUID breaks.
- Move or rename a file **inside Unity**, and Unity moves the `.meta` with it, so the GUID stays the same and nothing breaks, even though the path changed.
- Move or rename it in Explorer, and the `.meta` is left behind. Unity sees a new file without a `.meta` (new GUID), plus a `.meta` without a file (deleted). References break.

Hashing is used, but for something else: the content hash tells Unity **whether a file needs re-importing**. The imported result in `Library/` is cached under a key made from the file's contents, its `.meta` settings, the importer version and the target platform. So switching back to a platform you've used before can reuse earlier imports.

So: the **GUID** is the file's identity, the **path** is just where it currently lives, and the **hash** only detects changes.

### Usages of fileID

A reference is a GUID plus a fileID:

```yaml
m_Script: {fileID: 11500000, guid: a79441f348de89743a2939f4d699eac1, type: 3}
#          │                 │                                     │
#          │                 └ which FILE (from its .meta)          └ how the file is stored
#          └ which INSTANCE inside that file
```

One file can contain many instances, and the `fileID` picks one. It's used in three forms:

| Form | Meaning | Example from this project |
|---|---|---|
| `{fileID: X, guid: G, type: T}` | Instance `X` in another file `G` | `{fileID: 7400000, guid: 99dc…}`: the clip in `HoverBot@Shoot.fbx` |
| `{fileID: X}` | Instance `X` in **this same** file | `m_GameObject: {fileID: 2703355619797536407}`: the GameObject this component sits on |
| `{fileID: 0}` | Nothing (an empty field) | `m_PrefabInstance: {fileID: 0}` |

`type` says how the target file is stored:
- `type: 2`: a Unity asset loaded directly from its YAML file, e.g. `.mat`, `.asset`, `.controller`, `.mask`.
- `type: 3`: a file that goes through an importer first, e.g. `.cs`, `.fbx`, `.png`, `.wav`, and also `.prefab` and `.unity`, which have their own importers.

In this project, 360 material references use `type: 2`, and all 15,238 prefab references use `type: 3`.

### How fileID is generated

How the number is chosen depends on the kind of file.

**Files with one main instance** use a fixed number: the class ID of that instance × 100000. Every Unity class has a numeric ID:

| Instance of | Class ID | fileID |
|---|---|---|
| `GameObject` | 1 | `100000` |
| `Material` | 21 | `2100000` |
| `Texture2D` | 28 | `2800000` |
| `AnimationClip` | 74 | `7400000` |
| `AudioClip` | 83 | `8300000` |
| `Avatar` | 90 | `9000000` |
| `MonoScript` (a `.cs` file) | 115 | `11500000` |
| Prefab asset | 1001 | `100100000` |

A '.cs' script file has a strict 1:1 relationship with a MonoScript instance.
That's why every reference to a '.cs' script file is `fileID: 11500000` and why the class name must match the file name. (There could be classes other than MonoScript in the same file though.)
A scene points to a prefab always has `m_SourcePrefab: {fileID: 100100000, guid: …}`.

**Prefabs and scenes** give each GameObject and component a random 64-bit number when it's created, and save it in the file. Each instance starts with a header that holds both its class ID and its fileID:

```yaml
--- !u!20 &2286125971047457827     # class 20 = Camera, fileID = 2286125971047457827
Camera:
  m_GameObject: {fileID: 2703355619797536407}
```

Some older instances have short IDs, such as `&806887367` in `Player.prefab`, because earlier Unity versions used smaller numbers.

**Imported files with many instances**, like FBX models, can't store IDs in the file itself. The importer records them in the `.meta` instead, in `internalIDToNameTable`:

```yaml
# HoverBot@Shoot.fbx.meta
internalIDToNameTable:
- first:
    1: 100000        # class 1 (GameObject), fileID 100000
  second: //RootNode
- first:
    74: 7400000      # class 74 (AnimationClip), fileID 7400000
  second: Shoot
```

The table maps names to IDs, so you can re-export the FBX from a modelling program and references stay valid, as long as the names don't change.

### Unity Hierarchy and Component

A **component** is a descendant class of Unity's `Component` class and the instance of which must sits on a GameObject:

```
UnityEngine.Object                  ← base class of every type that can be saved and referenced
 ├─ Component                       ← anything that sits on a GameObject
 │   ├─ Transform, Rigidbody, Collider, AudioSource, Camera…   (built into the engine)
 │   └─ Behaviour                   ← a component that can be enabled/disabled
 │       └─ MonoBehaviour           ← the base for components written in C#
 │           ├─ Health, WeaponController, PlayerCharacterController…   (this project)
 │           └─ Image, Button, TextMeshProUGUI…                       (uGUI / TMP packages)
 ├─ ScriptableObject                ← NOT a component: a data asset
 └─ GameObject, Material, Mesh, Texture…
```

| | `MonoBehaviour` | `ScriptableObject` |
|---|---|---|
| Lives | On a GameObject | As its own `.asset` file in the project |
| Saved inside | The scene or prefab that holds that GameObject | Its own `.asset` file |
| Unity calls `Start()`, `Update()`… | Yes | No |
| Typical use | Behaviour: health, movement, weapons, UI | Shared data and settings, e.g. the URP settings in `Assets/Rendering/*.asset` |
| Name must match file | Yes | Yes |

**Components reference each other.** A component often needs other instances: the GameObject it's on, other components on the same GameObject, instances elsewhere in the same prefab, or objects in other files. Each of these is saved as a fileID reference, with or without a GUID:

| Reference | Same file? | Saved as |
|---|---|---|
| A GameObject listing its components | Yes | `m_Component: - component: {fileID: …}` |
| A component pointing back at its GameObject | Yes | `m_GameObject: {fileID: …}` |
| A script field pointing at a component in the same prefab | Yes | `WeaponCamera: {fileID: …}` |
| A script field pointing at something in another file | No | `{fileID: …, guid: …, type: …}` |
| A scene changing a value on an instance inside a prefab | No | `target: {fileID: …, guid: …}` |

The next section shows all of these in one real chain.

### Example of a reference

This example follows the chain from `MainScene` to the player's weapon camera and starting weapon.

**1. The scene places the Player prefab.** `MainScene.unity` doesn't copy the player. It stores a `PrefabInstance` that points to the prefab file, plus the values it changes:
(the prefix `m_` means member fields of an instance)
```yaml
# MainScene.unity
--- !u!1001 &2729281214152228920
PrefabInstance:
  m_Modification:
    m_Modifications:
    - target: {fileID: 2286125971047457827, guid: 761cdc9e4e883334cbdcea062d0befa7, type: 3}
      propertyPath: m_ClearFlags
      value: 4
  m_SourcePrefab: {fileID: 100100000, guid: 761cdc9e4e883334cbdcea062d0befa7, type: 3}
```

- `guid: 761cdc…` is `Player.prefab`, and `fileID: 100100000` is the prefab asset itself.
- The override's `target` uses the **same GUID with a different fileID**, `2286125971047457827`. That's the Camera component inside the prefab, and the scene changes its `m_ClearFlags`.

**2. Inside the prefab, a GameObject lists its components.**

```yaml
# Player.prefab
--- !u!1 &2703355619797536407       # GameObject "WeaponCamera"
GameObject:
  m_Component:
  - component: {fileID: 3809839388784154446}   # its Transform
  - component: {fileID: 2286125971047457827}   # its Camera
  m_Name: WeaponCamera
--- !u!20 &2286125971047457827      # the Camera component
Camera:
  m_GameObject: {fileID: 2703355619797536407}  # points back to the GameObject
```

**3. A script component references its script and other instances.**

```yaml
# Player.prefab
--- !u!114 &298744117520805562      # class 114 = MonoBehaviour
MonoBehaviour:
  m_GameObject: {fileID: 7253983127304857660}  # the "Player" root GameObject
  m_Script: {fileID: 11500000, guid: c6b4f98e62f1d104f8658bba08e5d56f, type: 3}
  StartingWeapons:
  - {fileID: 8323776800172828795, guid: b318fabc73446b746a1005cede0c4ceb, type: 3}
  WeaponCamera: {fileID: 2286125971047457827}
```

| Field | Points to | How it's resolved |
|---|---|---|
| `m_Script` | `PlayerWeaponsManager.cs` (guid `c6b4…`) | GUID → `.cs` file → its `MonoScript` (`11500000`) → class `PlayerWeaponsManager`, compiled into `fps.Gameplay.dll` |
| `WeaponCamera` | The Camera component from step 2 | No GUID, so it's in the same file |
| `StartingWeapons[0]` | `Weapon_Blaster.prefab` (guid `b318…`) | Instance `8323776800172828795` inside it, which is its `WeaponController` component |

The same field type, a reference to a component, can point inside the file (`WeaponCamera`) or into another file (`StartingWeapons`). The only difference is whether the GUID is there.

**4. A player build follows this chain.** Starting from `MainScene`, Unity reaches `Player.prefab`, from there `Weapon_Blaster.prefab`, then its materials, meshes and sounds, and so on. That's how it decides what to include. See [Making a player build](#making-a-player-build).

---

## Assemblies

References always point at **source files** (by GUID), never at a `.dll`. A prefab's component points to `Health.cs`, and Unity looks up which assembly that script was compiled into.

### What assemblies change, and what they don't

Assemblies are a separate layer from everything above. They only decide **how scripts are grouped physically**: which `.cs` files are compiled together into which `.dll`, and which `.dll` can call which. They don't change **how scripts are mapped logically** to assets:

- A component still points at its script's **`.cs` file GUID**, never at a `.dll`. You can move `Health.cs` from one assembly to another, and every prefab that uses `Health` still works.
- The file-name rule, fileIDs and GUIDs all work the same with or without assemblies.
- The game's behaviour doesn't change. A project with no `.asmdef` files at all puts every script into `Assembly-CSharp.dll` and runs identically.

What assemblies do change:
- **Which code can use which.** A class in `fps.Game` can't mention a class in `fps.Gameplay` unless `fps.Game` references it (see `learn_game.md`).
- **Compile time.** Changing a script recompiles only its assembly and those that reference it, not the whole project.
- **What ships.** Editor-only assemblies are left out of player builds.

### `.asmdef` and `.asmref`

Two small JSON files decide how the split works. They describe how the code is built, like `BUILD` files in Bazel.

**`.asmdef`: defines one assembly.** It goes in a folder and claims every `.cs` file in that folder and its subfolders, except subfolders that contain their own `.asmdef`. Those become separate assemblies, the same way a subdirectory with its own `BUILD` file becomes a separate Bazel package.

| `.asmdef` field | What it does | Bazel equivalent |
|---|---|---|
| `name` | The assembly's name; the output is `<name>.dll` | target name |
| *(no field)* | Source files are picked automatically: every `.cs` under the folder | `srcs = glob(["**/*.cs"])` |
| `references` | Other assemblies this code can use (by name or `GUID:`) | `deps` |
| `includePlatforms` / `excludePlatforms` | Build only for certain platforms (e.g. `["Editor"]` for editor tools) | `target_compatible_with` |
| `defineConstraints` | Build only if certain compile symbols are defined | `select()` on a config |
| `versionDefines` | Define a symbol when a package is installed at a certain version | config based on dependency version |
| `precompiledReferences` (+ `overrideReferences`) | Use specific prebuilt `.dll` files | importing a prebuilt library |
| `autoReferenced` | Whether the default assembly (below) automatically references this one | — |

Inside `references`, an entry like `GUID:9c5543ea…` is the GUID of the other assembly's `.asmdef` file, so renaming an assembly doesn't break references to it.

**`.asmref`: adds a folder to an existing assembly.** It holds one reference, `"reference": "fps.Game"`, which means "compile the scripts in this folder as part of `fps.Game`". It doesn't create a new `.dll`. It's useful when one assembly's code is spread across several folders. This project doesn't have any.

**Scripts outside any `.asmdef`** go into Unity's default assembly, `Assembly-CSharp.dll`. It can use every assembly marked `autoReferenced`, but no `.asmdef` can reference it. In this project every script is covered by an `.asmdef`, so there's no `Assembly-CSharp.dll`.

### This project's assemblies

| `.asmdef` | Output | References | Notes |
|---|---|---|---|
| `FPS/Scripts/Game/fps.Game.asmdef` | `fps.Game.dll` | Unity.ProBuilder | Base layer; see `learn_game.md` |
| `FPS/Scripts/Gameplay/fps.Gameplay.asmdef` | `fps.Gameplay.dll` | fps.Game, Unity.InputSystem | |
| `FPS/Scripts/AI/fps.AI.asmdef` | `fps.AI.dll` | fps.Game | |
| `FPS/Scripts/UI/fps.UI.asmdef` | `fps.UI.dll` | fps.Game, fps.Gameplay, fps.AI, Unity.TextMeshPro, Unity.InputSystem | |
| `NavMeshComponents/Scripts/NavMeshComponents.asmdef` | `NavMeshComponents.dll` | — | |
| `NavMeshComponents/Editor/NavMeshComponentsEditor.asmdef` | `NavMeshComponentsEditor.dll` | NavMeshComponents | `includePlatforms: ["Editor"]`: a nested assembly for editor-only tools, left out of builds |

### How this differs from Bazel

- `.asmdef` files are **optional**. Without any, Unity puts all scripts into `Assembly-CSharp.dll` and the project still works.
- They only cover **C# code**. Textures, models, prefabs and scenes don't go through them; those are imported separately and collected by reference.
- **Compiling is automatic.** Unity recompiles the changed assemblies, and the ones that use them, whenever a script changes. Making a player build is a separate step.
- There's no list of source files. An assembly is defined entirely by which folder its `.asmdef` sits in.

---

## Making a player build

**File > Build Profiles > Build** produces the playable game. Roughly:

1. **Start from the build scenes.** These are the scenes listed in `ProjectSettings/EditorBuildSettings.asset`: here `IntroMenu`, `WinScene`, `LoseScene` and `MainScene`. Scenes not on the list, like `SecondaryScene`, aren't included.
2. **Follow references.** Starting from those scenes, Unity follows every GUID reference: scene → prefab → material → texture, prefab → script, controller → animation clip, and so on (see [Example of a reference](#example-of-a-reference)). **Only assets reached this way are included.** Assets nothing points to, such as all of `ModAssets/` and the unused textures in `learn_binary.md`, stay out of the build.

   A few things are always included, wherever they're referenced from:
   - everything in a `Resources/` folder, which code can load by name (here, TextMesh Pro's default fonts and settings);
   - `StreamingAssets/`, copied as-is;
   - shaders listed under Graphics settings' "Always Included".
3. **Convert assets for the target platform.** For example, textures are compressed to the platform's format and audio is encoded. The editor's `Library/` cache is a different format from what ships.
4. **Include the code.** All assemblies built for that platform are included. Editor-only code is left out: assemblies with `includePlatforms: ["Editor"]` and scripts in folders named `Editor/`. Then the code is prepared in one of two ways:
   - **Mono:** the `.dll` files ship as they are and are compiled to machine code while the game runs.
   - **IL2CPP:** the `.dll` code is converted to C++ and compiled into a native executable ahead of time. This is required for WebGL, iOS and consoles.
5. **Write the output:** the executable plus data files holding the packed scenes, assets and code.

Script compilation (step 4) uses the same `.asmdef` setup as the editor, so a compile error that shows up in the editor also blocks the build.


### From saved instance to live instance

A build packs the scenes and prefabs as **saved instances**: the same data as the YAML in the project, in a compact binary form. When the game runs and a scene loads (and likewise when you press Play in the editor), Unity turns them into live C# instances. This is deserialization, the same idea as loading JSON into objects. It happens in passes:

**Pass 1: create everything.** For each saved instance in the file:
- **Pick the class.** For built-in types, the header says the class (`!u!20` is Camera). For script components (`!u!114`), Unity follows `m_Script` → `MonoScript` → class name → the class in the `.dll`.
- **Create the instance.** Unity creates it itself, keyed by its fileID.
- **Copy in plain values.** Each saved value goes into the field with the same name: `BobFrequency: 10` goes into `BobFrequency`.

"Everything" here means everything **saved in the file**, not every instance the game will ever have. Many instances are created later by code while the game runs: a bullet for every shot, a muzzle flash, a dropped health pack, an enemy's death effect. They're made with [`Instantiate()`](https://docs.unity3d.com/ScriptReference/Object.Instantiate.html), which copies a prefab, or with `AddComponent<T>()` or `new`. They have no fileID and are never saved, so pass 2 never links them. Instead, code passes references to them directly. In this project, `WeaponController.HandleShoot()` holds a saved reference to the bullet prefab (`ProjectilePrefab`), clones it with `Instantiate()` on every shot, and calls `newProjectile.Shoot(this)` so the bullet knows which weapon fired it. Each bullet later removes itself with `Destroy()`. Games that spawn a lot of bullets often reuse a fixed set instead of creating and destroying them each time, which is called object pooling. For more:
- [Instantiating prefabs at runtime](https://docs.unity.com/en-us/engine/6000.6/manual/working-with-gameobjects/prefabs/instantiating/intro) (Unity Manual): uses firing a projectile from a launcher as its example.
- [Use object pooling to boost performance of C# scripts](https://learn.unity.com/tutorial/use-object-pooling-to-boost-performance-of-c-scripts-in-unity) (Unity Learn): a tutorial that pools projectiles. Unity also has a ready-made pool class, [`ObjectPool<T>`](https://docs.unity3d.com/ScriptReference/Pool.ObjectPool_1.html).

**Pass 2: link everything.** Each reference field is pointed at a live instance:
- `{fileID: X}` → the instance created from fileID `X` in pass 1.
- `{fileID: X, guid: G}` → instance `X` of file `G`, which is loaded first if it isn't already.

Two passes are needed because references form loops. A GameObject lists its components, and each component points back to its GameObject, so neither can be finished before the other exists. No searching happens: every reference is a lookup in the table of instances created from this file. **The file itself acts as the instance pool**, a closed list of everything that can be referenced. That pool exists only while the file loads.

**Pass 3: call the lifecycle methods.** `Awake()`, `OnEnable()`, then `Start()` before the first frame.

This is close to dependency injection: `PlayerWeaponsManager` never searches for its camera, because the reference was set in the editor, saved as a fileID, and filled in on load. The difference from a DI container is that nothing is decided while the game runs. Every link was chosen in advance and saved in the file.

#### Why it feels hidden

The loader is part of the engine's native C++ code, which isn't open source, so you can't step through it in a debugger. Your C# code only sees the result, through these hooks:

| Hook | When it runs |
|---|---|
| Field initializers and the parameterless constructor | When Unity creates the instance, **before** saved values are copied in. It may run off the main thread, and more than once, so it's not used for setup. |
| `ISerializationCallbackReceiver.OnAfterDeserialize()` | Right after the saved values are copied in (pass 1) |
| `Awake()` | After all references are linked (pass 2); other components are safe to use |
| `OnEnable()` | Each time the component becomes active |
| `Start()` | Once, before the instance's first frame |

Your code never calls a component's constructor itself. Unity does, but too early to be useful, which is why setup goes in `Awake()` and `Start()`.
