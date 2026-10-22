Almost all non binary resources can be modified by AI without toughing Unity UI.

**Walkable areas** are baked by scanning level's floors, walls and ramps to get the path that an enemy can walk along.
It is rebaked whenever geometry in main scene is moved.

**Scene**
Scene is a high level abstraction of a set of GameObjects(Game objects are compositions of Components).

- Projects build needs at least a scene in `Asset/Scenes`, and there is always a scene in an editor.
- This sample game has 5 scenes
  - **IntroMenu** for start menu
  - **MainScene** for the playable level
  - **WinScene/LoseScene** for end screens
  - **SecondaryScene** which is just a WIP scene outside of the build
- The logic of scene loading is non binary. E.g ProjectSettings/EditorBuildSettings.asset.
- Several scenes can be loaded at once.

**Prefabs**
Prefabs are templates for reference. It is optionally included in the build only when referenced.

**Models**
For this project, models with skeleton and animations are placed under `Assets/Animation`
(the folder sturcture is the owners choice not a convention, models are tracked by their `.meta` file.)
- `Rigs/` contains models only (`Rigs/Materials` is unused and it's not a good practice to put materials here).
- `Controllers/` a controller is a set of states where animation can be controlled to transform from one state to another.
- `Mask/` a mask limits the shoot and hurt animation to the upper bones, so that different animations don't affect each other.
- `HoverBot/` and `Turret/` only hold animation @clips.

For other static models, they are placed under `Art/Models/`.
