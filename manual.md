# SaiZ — Box Pickup, Drop & Stacking Manual

## 1. Controls (In-Game)

- **Move:** Left-click (or hold left mouse button) on the floor using the TopDown controller (`Content/TopDown/Blueprints/BP_TopDownController`).
- **Pick Up Box:** Walk close to any pickupable box (within `200 cm`) and press **`Space`**. The nearest box attaches to the character's right hand (`hand_rSocket`).
- **Drop on Floor:** Walk to an open area and press **`Space`** again.
- **Stack on Another Box:** Walk directly up facing an existing box (~`85 cm` ahead) and press **`Space`** again. The carried box snaps centered onto the target box's `(X, Y)` and rests flush on top of its bounding box (`Origin.Z + BoxExtent.Z`).

---

## 2. What Was Broken & Why

1. **Boxes disappeared on touch (`BP_GrowthBox`, `BP_ShrinkBox` -> duplicated to `BP_GrowthBox1`, `BP_ShrinkBox1`):**
   - Their `EventGraph` ran `OnComponentBeginOverlap (GrowthTrigger) -> Cast To BP_TopDownCharacter -> SetActorScale3D -> DestroyActor`, destroying the box on contact.
   - `StaticMeshComponent0` had `Mobility = Static` and `Collision = BlockAll`, which prevented runtime movement and socket attachment.
2. **Broken & incomplete pickup in `BP_WoodenBox`:**
   - `BP_WoodenBox`'s `EventGraph` failed to compile because `AttachComponentToComponent` lacked a valid `Parent` target (`KeepRelative` attachment on overlap) and had no collision toggle, drop logic, or stacking support.
3. **No character-side pickup/drop/stack system in `BP_TopDownCharacter`:**
   - `BP_TopDownCharacter` had an empty `EventGraph` with no `Space` key handling, carry state tracking, surface detection, or height calculation.
4. **Different box pivots, scales, and socket scale (`hand_rSocket`):**
   - `BP_GrowthBox1` / `BP_ShrinkBox1` have center-pivoted meshes (bounds half-extent `32 x 32 x 32`, default scale `0.5`), whereas `BP_WoodenBox` has a bottom-pivoted mesh (`50 x 50 x 50` half-extent).
   - `hand_rSocket` on `SKM_Manny_Simple` (`bone: hand_r`) carries a `0.5` relative scale and hand bone rotation. Dropping without restoring original scale (`HeldBoxOrigScale`) and upright rotation (`DropRot`) *before* calling `GetActorBounds` caused distorted Z offsets and floating boxes.
5. **TopDown NavMesh cutouts & trace reliability:**
   - Default `Can Ever Affect Navigation = True` on boxes carved NavMesh holes around them, stopping click-to-move characters short of the boxes.
   - Using `MultiSphereTraceByChannel` for pickup could terminate early on floor/wall `Visibility` blockers, and a `110 cm` drop trace could overshoot a `64 cm` box when standing directly against it. Switching pickup to `GetAllActorswithTag("PickupBox")` (nearest within `200 cm`) and drop trace offset to `85 cm` resolved both issues.

---

## 3. Changes Applied (Paths Relative to `Game Design Mastery/SaiZ/Content/`)

*(Originals `Content/Blueprints/BP_GrowthBox` and `Content/Blueprints/BP_ShrinkBox` were left untouched.)*

### Box Blueprints
- `Content/Blueprints/BP_GrowthBox1`
- `Content/Blueprints/BP_ShrinkBox1`
- `Content/Blueprints/BP_WoodenBox`

Changes applied to each:
- **Static Mesh Component (`StaticMeshComponent0` / `Interact`):**
  - `Mobility`: **`Movable`**
  - `Collision Presets`: **`BlockAllDynamic`**
  - `Simulate Physics`: **Unchecked (`False`)**
  - `Can Ever Affect Navigation`: **Unchecked (`False`)** on `StaticMeshComponent0`
- **Class Defaults -> Actor -> Tags:** Added tag **`PickupBox`**.
- **EventGraph:** Cleared all old overlap/destroy/broken-attach nodes (`0` nodes in EventGraph) so colliding with boxes leaves them intact until picked up with `Space`.

### Skeleton Socket
- `Content/Characters/Mannequins/Meshes/SKM_Manny_Simple` (`SK_Mannequin` skeleton):
  - Uses socket **`hand_rSocket`** attached to bone **`hand_r`** (relative scale `0.5, 0.5, 0.5`).

### Player Character (`Content/TopDown/Blueprints/BP_TopDownCharacter`)
- **GameMode & Controller:** `Content/TopDown/Blueprints/BP_TopDownGameMode` uses `BP_TopDownCharacter` as `Default Pawn Class` and `BP_TopDownController` as `Player Controller Class`, preserving all existing mouse/touch controls.

#### Variables Added
| Name | Type | Purpose |
|---|---|---|
| `HeldBox` | `Actor` (Object Reference) | Currently carried box (`None` when hands are empty) |
| `HeldBoxOrigScale` | `Vector` | Original actor scale saved on pickup and restored on drop |
| `NearestBox` | `Actor` (Object Reference) | Closest candidate box found during pickup search |
| `BestDist` | `Float` | Distance cutoff (`200.0 cm`) / shortest distance tracker |
| `DropSurfaceZ` | `Float` | Target surface top Z coordinate (floor or target box top) |
| `DropX` | `Float` | Target world X coordinate for drop/stack |
| `DropY` | `Float` | Target world Y coordinate for drop/stack |
| `DropRot` | `Rotator` | Target upright rotation on drop/stack |

#### `EventGraph`
- `Input -> Keyboard Events -> Space Bar` (`Pressed`) ---> **`ToggleCarryBox`**

#### Function `ToggleCarryBox`
1. **Branch on `Is Valid (HeldBox)`:**
2. **If `HeldBox` is Not Valid (Pick Up):**
   - Sets `NearestBox = None`, `BestDist = 200.0`.
   - Calls **`Get All Actors with Tag`** (`Tag = PickupBox`).
   - Loops over `Out Actors`, checks **`Get Distance To`** (`self` to candidate): if `< BestDist`, updates `BestDist` and `NearestBox`.
   - If `Is Valid (NearestBox)`:
     1. Stores `Get Actor Scale 3D (NearestBox)` into `HeldBoxOrigScale`.
     2. Calls `Set Actor Enable Collision (NearestBox, false)` so the carried box does not push the player.
     3. Sets `Set Actor Scale 3D` to `(0.25, 0.25, 0.25)` while held in hand.
     4. Calls `Attach Actor To Component` on `NearestBox`: `Parent = Mesh`, `Socket Name = hand_rSocket`, `Location/Rotation Rule = Snap to Target`, `Scale Rule = Keep World`, `Weld Simulated Bodies = false`.
     5. Sets `HeldBox = NearestBox`.
3. **If `HeldBox` is Valid (Drop / Stack):**
   - Runs `Line Trace By Channel` (`Visibility`) `85.0 cm` in front of the character (`Start = ActorLocation + Forward * 85 + (0,0,250)`, `End = Start - (0,0,600)`, ignoring `self` and `HeldBox`):
     - **Hit Actor has tag `PickupBox` (Stack):** Reads `Get Actor Bounds (Hit Actor, bOnlyCollidingComponents = true)`. Sets `DropSurfaceZ = Origin.Z + BoxExtent.Z`, `DropX = Origin.X`, `DropY = Origin.Y`, `DropRot = Get Actor Rotation (Hit Actor)`.
     - **Hit non-box surface (Floor Drop):** Uses `Break Hit Result -> Impact Point` for `DropX`, `DropY`, `DropSurfaceZ`, and `Make Rotator (0, 0, Character Yaw)` for `DropRot`.
     - **No trace hit (Fallback):** Places at `ActorLocation + Forward * 85` and `Character Yaw`.
   - Final placement sequence:
     1. `Detach From Actor (HeldBox)` (`Keep World` for location, rotation, and scale).
     2. `Set Actor Scale 3D (HeldBox, HeldBoxOrigScale)`.
     3. `Set Actor Location And Rotation (HeldBox, CurrentLocation, DropRot)` — levels rotation **before** measuring bounds.
     4. `Set Actor Enable Collision (HeldBox, true)`.
     5. Computes `TargetZ = DropSurfaceZ + BoxExtent.Z + (ActorLocation.Z - BoundsOrigin.Z)` using `Get Actor Bounds (HeldBox, true)` so both center-pivoted and bottom-pivoted boxes sit flush.
     6. `Set Actor Location And Rotation (HeldBox, Make Vector(DropX, DropY, TargetZ), DropRot, Sweep = false, Teleport = true)`.
     7. Clears `HeldBox = None`.

---

## 4. How to Make Any New Box Pickupable & Stackable

For any new box Blueprint or actor in a level:
1. Ensure its mesh component has **Mobility = `Movable`**, **Collision Presets = `BlockAllDynamic`**, **Simulate Physics = unchecked**, and **Can Ever Affect Navigation = unchecked**.
2. In **Class Defaults -> Actor -> Tags** (or the actor instance's **Details -> Tags**), add **`PickupBox`**.
3. Do not attach `DestroyActor` or auto-attach overlap events to it; `BP_TopDownCharacter` handles pickup, drop, and stacking automatically.
