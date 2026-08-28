# EmuVR Modding Docs

## Table of Contents

* [Prerequisites](#prerequisites)
* [Environment Setup](#environment-setup)
* [Adding Assembly References](#adding-assembly-references)
* [Fundamental Mod Architecture](#fundamental-mod-architecture)
* [Core Mod Examples](#core-mod-examples)

  * [Cube Spawner](#example-1-cube-spawner)
  * [Free Cam](#example-2-free-cam)
  * [Menu Screen](#example-3-menu-screen)
  * [Object Grabber](#example-4-object-grabber)
* [Compilation and Deployment](#compilation-and-deployment)

---

# Prerequisites

Before creating an EmuVR mod, you will need:

* EmuVR
* MelonLoader
* Visual Studio 2022
* .NET Framework 4.7.2 or .NET Framework 3.5
* Basic knowledge of C#
* Basic knowledge of Unity

---

# Environment Setup

## MelonLoader Installation

Download the MelonLoader installer from the official MelonLoader GitHub repository.

1. Run `MelonLoader.Installer.exe`.
2. Select `SELECT GAME`.
3. Select `emuvr.exe` inside your EmuVR installation directory.
4. Select MelonLoader version `v0.6.1` or `v0.5.7`.
5. Click `Install`.

Launch `emuvr.exe` once to allow MelonLoader to initialize.

After initialization, MelonLoader should generate the following directories:

```text
EmuVR/
├── Mods/
└── MelonLoader/
```

Close EmuVR after these directories have been created.

---

# Visual Studio Environment Setup

Open Visual Studio 2022 and create a new project using:

```
Class Library (.NET Framework)
```

Set the target framework to:

```
.NET Framework 4.7.2
```

.NET Framework 3.5 can also be used where required.

Set the active solution platform to:

```
x64
```

---

# Adding Assembly References

EmuVR mods need references to the game's compiled assemblies.

In Visual Studio:

```text
Solution Explorer
→ References
→ Add Reference...
→ Browse
```

Navigate to your EmuVR installation directory and add the required DLLs.

```text
EmuVR/MelonLoader/net35/MelonLoader.dll

EmuVR/emuvr_Data/Managed/
├── UnityEngine.dll
├── UnityEngine.CoreModule.dll
├── UnityEngine.PhysicsModule.dll
├── UnityEngine.UI.dll
├── UnityEngine.UIModule.dll
└── UnityEngine.InputLegacyModule.dll
```

These references allow your mod to communicate with MelonLoader and Unity's runtime APIs.

---

# Fundamental Mod Architecture

A typical MelonLoader mod consists of several important components.

## Assembly Attributes

Assembly attributes register your mod's metadata with MelonLoader.

Example:

```csharp
[assembly: MelonInfo(typeof(EmuVRMods.CubeSpawner), "Cube Spawner", "1.0.0", "YourName")]
[assembly: MelonGame("EmuVR", "EmuVR")]
```

The metadata includes:

* Mod name
* Version
* Author
* Target game

## Class Inheritance

Your main mod class should inherit from:

```csharp
MelonMod
```

Example:

```csharp
public class CubeSpawner : MelonMod
{
}
```

## Execution Lifecycle

MelonLoader provides several lifecycle methods.

### OnUpdate

`OnUpdate()` runs every frame.

It is useful for:

* Keyboard input
* Mouse input
* Frame-based logic
* UI state
* Gameplay systems

### OnLateUpdate

`OnLateUpdate()` runs after standard `OnUpdate()` calls.

It can be useful for:

* Camera manipulation
* Camera tracking
* Movement overrides
* Systems that need to run after normal updates

## Direct Unity API Access

Mods can directly interact with Unity APIs such as:

```csharp
GameObject.CreatePrimitive()
Camera.main
Physics.Raycast()
Input.GetKeyDown()
```

---

# Core Mod Examples

# Example 1: Cube Spawner

## Mechanics

Pressing `F8` creates a Unity cube one meter in front of the main camera.

The cube is:

* Scaled to `0.3`
* Given a Rigidbody
* Assigned a mass of `1.0`
* Affected by Unity physics

## Implementation Architecture

The mod:

1. Detects `F8` using `Input.GetKeyDown()`.
2. Gets the main camera.
3. Calculates a position one meter in front of the camera.
4. Creates a cube using `GameObject.CreatePrimitive()`.
5. Adds a Rigidbody.
6. Configures the Rigidbody.

```csharp
using MelonLoader;
using UnityEngine;

[assembly: MelonInfo(typeof(EmuVRMods.CubeSpawner), "Cube Spawner", "1.0.0", "YourName")]
[assembly: MelonGame("EmuVR", "EmuVR")]

namespace EmuVRMods
{
    public class CubeSpawner : MelonMod
    {
        public override void OnUpdate()
        {
            Camera mainCam = Camera.main;

            if (mainCam == null)
                return;

            if (Input.GetKeyDown(KeyCode.F8))
            {
                Vector3 spawnPosition =
                    mainCam.transform.position +
                    (mainCam.transform.forward * 1.0f);

                GameObject cube =
                    GameObject.CreatePrimitive(PrimitiveType.Cube);

                cube.transform.position = spawnPosition;

                cube.transform.localScale =
                    new Vector3(0.3f, 0.3f, 0.3f);

                Rigidbody rb =
                    cube.AddComponent<Rigidbody>();

                rb.mass = 1.0f;
            }
        }
    }
}
```

## Feature Expansions

The example can be expanded by:

* Replacing the cube with a sphere.
* Replacing the cube with a cylinder.
* Applying an initial force.
* Changing the object's mass.
* Changing the object's scale.
* Adding custom materials.

For example:

```csharp
rb.velocity = mainCam.transform.forward * 10f;
```

---

# Example 2: Free Cam

## Mechanics

Pressing `Shift + P` toggles a free camera.

Camera movement is handled inside `OnLateUpdate()` to reduce jitter caused by other player-controller updates.

## Implementation Architecture

The mod:

1. Tracks whether free camera mode is active.
2. Gets `Camera.main`.
3. Stores the camera's original FOV.
4. Tracks camera rotation.
5. Processes mouse input.
6. Processes keyboard movement.
7. Updates the camera position during `OnLateUpdate()`.

```csharp
using MelonLoader;
using UnityEngine;

[assembly: MelonInfo(typeof(EmuVRMods.FreeCam), "Free Cam", "1.0.0", "YourName")]
[assembly: MelonGame("EmuVR", "EmuVR")]

namespace EmuVRMods
{
    public class FreeCam : MelonMod
    {
        private bool isActive = false;
        private Camera cam;

        private Vector3 camPos;

        private float rotX = 0f;
        private float rotY = 0f;

        private float defaultFov = 60f;
        private float currentFov = 60f;

        public override void OnUpdate()
        {
            if ((Input.GetKey(KeyCode.LeftShift) ||
                 Input.GetKey(KeyCode.RightShift)) &&
                Input.GetKeyDown(KeyCode.P))
            {
                isActive = !isActive;

                if (cam == null)
                    cam = Camera.main;

                if (cam != null)
                {
                    if (isActive)
                    {
                        camPos = cam.transform.position;

                        defaultFov = cam.fieldOfView;
                        currentFov = defaultFov;

                        rotX = cam.transform.eulerAngles.y;
                        rotY = cam.transform.eulerAngles.x;
                    }
                    else
                    {
                        cam.fieldOfView = defaultFov;
                    }
                }
            }

            if (isActive && cam != null && Input.GetKey(KeyCode.Z))
            {
                float scroll = Input.GetAxis("Mouse ScrollWheel");

                if (scroll > 0f)
                    currentFov -= 5f;
                else if (scroll < 0f)
                    currentFov += 5f;

                currentFov = Mathf.Clamp(currentFov, 5f, 100f);

                cam.fieldOfView = currentFov;
            }
            else if (Input.GetKeyUp(KeyCode.Z) && cam != null)
            {
                cam.fieldOfView = defaultFov;
            }
        }

        public override void OnLateUpdate()
        {
            if (!isActive || cam == null)
                return;

            if (Input.GetMouseButton(1))
            {
                rotX += Input.GetAxis("Mouse X") * 2f;
                rotY -= Input.GetAxis("Mouse Y") * 2f;

                rotY = Mathf.Clamp(rotY, -90f, 90f);
            }

            cam.transform.rotation =
                Quaternion.Euler(rotY, rotX, 0f);

            float speed =
                Input.GetKey(KeyCode.LeftShift) ? 9f : 3f;

            Vector3 move = Vector3.zero;

            if (Input.GetKey(KeyCode.W))
                move += cam.transform.forward;

            if (Input.GetKey(KeyCode.S))
                move -= cam.transform.forward;

            if (Input.GetKey(KeyCode.D))
                move += cam.transform.right;

            if (Input.GetKey(KeyCode.A))
                move -= cam.transform.right;

            if (Input.GetKey(KeyCode.E))
                move += Vector3.up;

            if (Input.GetKey(KeyCode.Q))
                move -= Vector3.up;

            camPos += move * speed * Time.deltaTime;

            cam.transform.position = camPos;
        }
    }
}
```

---

# Example 3: Menu Screen

## Mechanics

Pressing `Tab` creates and toggles a basic screen-space UI overlay.

The example creates:

* A Canvas
* A CanvasScaler
* A GraphicRaycaster
* A background Image
* A Text element

The Canvas uses:

```csharp
RenderMode.ScreenSpaceOverlay
```

## Implementation Architecture

The UI is created entirely at runtime.

```csharp
using MelonLoader;
using UnityEngine;
using UnityEngine.UI;

using Image = UnityEngine.UI.Image;
using Text = UnityEngine.UI.Text;

[assembly: MelonInfo(typeof(EmuVRMods.MenuScreen), "Menu Screen", "1.0.0", "YourName")]
[assembly: MelonGame("EmuVR", "EmuVR")]

namespace EmuVRMods
{
    public class MenuScreen : MelonMod
    {
        private GameObject canvasObj;
        private bool isVisible = false;

        public override void OnUpdate()
        {
            if (Input.GetKeyDown(KeyCode.Tab))
            {
                isVisible = !isVisible;

                if (canvasObj == null)
                    BuildUi();

                if (canvasObj != null)
                    canvasObj.SetActive(isVisible);
            }
        }

        private void BuildUi()
        {
            canvasObj = new GameObject("ModCanvas");

            Object.DontDestroyOnLoad(canvasObj);

            Canvas canvas =
                canvasObj.AddComponent<Canvas>();

            canvas.renderMode =
                RenderMode.ScreenSpaceOverlay;

            canvas.sortingOrder = 999;

            canvasObj.AddComponent<CanvasScaler>();
            canvasObj.AddComponent<GraphicRaycaster>();

            GameObject panel =
                new GameObject("Panel");

            panel.transform.SetParent(
                canvasObj.transform,
                false);

            Image img =
                panel.AddComponent<Image>();

            img.color =
                new Color(0f, 0f, 0f, 0.85f);

            RectTransform panelRect =
                panel.GetComponent<RectTransform>();

            panelRect.sizeDelta =
                new Vector2(260f, 90f);

            panelRect.anchoredPosition =
                Vector2.zero;

            GameObject textObj =
                new GameObject("Text");

            textObj.transform.SetParent(
                panel.transform,
                false);

            Text txt =
                textObj.AddComponent<Text>();

            txt.text =
                "Mod Menu Loaded";

            txt.font =
                Resources.GetBuiltinResource<Font>(
                    "Arial.ttf");

            txt.fontSize = 22;
            txt.alignment =
                TextAnchor.MiddleCenter;

            txt.color = Color.white;

            RectTransform textRect =
                textObj.GetComponent<RectTransform>();

            textRect.sizeDelta =
                panelRect.sizeDelta;

            textRect.anchoredPosition =
                Vector2.zero;
        }
    }
}
```

## Feature Expansions

The menu can be expanded with:

* Buttons
* Toggles
* Tabs
* Mod settings
* Custom themes
* Dynamic colors
* Additional controls

For example, a Unity `Button` component can be attached to a UI object to trigger actions when clicked.

---

# Example 4: Object Grabber

## Mechanics

Holding the Middle Mouse Button performs a raycast from the camera.

The raycast has a maximum distance of 10 meters.

If the ray hits an object with a Rigidbody, that Rigidbody is stored and moved toward a position approximately 1.5 meters in front of the camera.

Releasing the Middle Mouse Button stops grabbing the object.

## Implementation Architecture

The mod uses:

```csharp
Physics.Raycast()
```

to detect objects.

It then stores the Rigidbody:

```csharp
private Rigidbody targetRb;
```

The Rigidbody is continuously moved toward the desired position.

```csharp
using MelonLoader;
using UnityEngine;

[assembly: MelonInfo(typeof(EmuVRMods.ObjectGrabber), "Object Grabber", "1.0.0", "YourName")]
[assembly: MelonGame("EmuVR", "EmuVR")]

namespace EmuVRMods
{
    public class ObjectGrabber : MelonMod
    {
        private Rigidbody targetRb;

        public override void OnUpdate()
        {
            Camera cam = Camera.main;

            if (cam == null)
                return;

            if (Input.GetMouseButtonDown(2))
            {
                Ray ray = new Ray(
                    cam.transform.position,
                    cam.transform.forward);

                if (Physics.Raycast(
                    ray,
                    out RaycastHit hit,
                    10f))
                {
                    if (hit.rigidbody != null)
                    {
                        targetRb = hit.rigidbody;
                    }
                }
            }

            if (Input.GetMouseButtonUp(2))
            {
                targetRb = null;
            }

            if (targetRb != null)
            {
                Vector3 holdPos =
                    cam.transform.position +
                    (cam.transform.forward * 1.5f);

                Vector3 dir =
                    holdPos - targetRb.transform.position;

                targetRb.velocity = dir * 10f;
            }
        }
    }
}
```

## Feature Expansions

The grabber can be expanded with:

* Adjustable grab distance
* Scroll-wheel distance control
* Object highlighting
* Throwing
* Forward impulse when released
* Mass restrictions

---

# Compilation and Deployment

## 1. Build the Project

In Visual Studio, select:

```
Build → Build Solution
```

Or press:

```
Ctrl + Shift + B
```

---

## 2. Locate the Compiled DLL

Depending on your configuration, the compiled DLL will be located in:

```text
bin/Release/
```

or:

```text
bin/Debug/
```

---

## 3. Copy the DLL to EmuVR

Copy the compiled `.dll` into:

```text
EmuVR/Mods/
```

For example:

```text
EmuVR/
└── Mods/
    └── YourMod.dll
```

---

## 4. Launch EmuVR

Launch:

```text
emuvr.exe
```

MelonLoader should detect and load the mod automatically.

---

# Contributing

Contributions are welcome.

You can contribute by:

* Adding new examples
* Improving documentation
* Fixing errors
* Adding tutorials
* Adding compatibility information
* Improving existing mod implementations
