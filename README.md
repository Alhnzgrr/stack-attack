# Stack Attack

A portrait, one-thumb arcade shooter built in **Unity 6 (URP 2D)** as a test case for **OmicronGames**.

Built with Claude Code under a project-local rule-and-hook system committed at [`.claude/`](.claude) —
see [How This Was Built](#how-this-was-built).

You drag your thumb to steer, hold to fire, and break stacked plates before they scroll down into you.
Breaking plates earns points, points buy upgrades mid-run, and each level ends with a boss that holds
its ground and throws stacks at you while a ring of guards spins around it.

---

## Gameplay

https://github.com/user-attachments/assets/c5bfd950-74bd-435e-b0e3-8a9c04aebae6

*Recorded from the Unity Editor: drag movement, auto-fire, plate destruction, a mid-run upgrade choice,
the boss fight and the end screen.*

The full recording is also committed to the repository at [`docs/gameplay.mp4`](docs/gameplay.mp4).

**[Download the Android build (.apk)](../../releases/latest)** — ARM64, IL2CPP, ~40 MB. Sideload to
install; it is signed with a debug keystore, so Android will ask you to allow installation from an
unknown source.

---

## The Core Loop

| Step | What happens |
| --- | --- |
| **Steer** | Drag anywhere on screen. The drag is measured as a fraction of screen width, so the same swipe covers the same playfield distance on every resolution. The ship leans into its own velocity and eases back when you stop. |
| **Fire** | Holding fires automatically at the current fire rate. The cooldown keeps draining while your finger is up, so tapping can never beat the stat. |
| **Break** | Every stack is a tower of plates. A plate takes `hitsPerPlate` shots to break and the tower shrinks from the top. Touching a stack costs health and grants a short invulnerability window. |
| **Earn** | Points come from broken plates only — never from chip damage, and never from hitting the boss. The boss is worth nothing on its own; the stacks it throws at you are the payday. |
| **Upgrade** | When your points cross the next threshold, time freezes and three cards come up. Each threshold costs more than the last, so choices thin out over a level instead of flooding in near the end. |
| **Finish** | The level is won when the last entry has spawned *and* the playfield is empty — not when a timer runs out. Losing all health ends the run. |

Upgrades are **run-scoped**: every level restarts from the configured base stats, so a later level can
never be trivialised by what an earlier one handed out. Only the level you reached is persisted.

### Upgrades

Nine kinds, drawn three at a time with no duplicate stat in a single offer:

`FireRate` · `Damage` · `ProjectileCount` · `ProjectileSize` · `Pierce` · `MaxHealth` ·
`Greed` (point multiplier) · `BoomerangCount` · `RocketDamage`

### Skills

Boomerangs and rockets run on their own timers rather than on the trigger — they come round on a clock,
fire themselves, and are drawn from the same pool the projectiles use.

---

## Level Editor

Levels are authored in a **custom EditorWindow** (`StackAttack > Level Editor`) rather than by typing
numbers into a list. The window draws the level as a top-down strip: horizontal is the playfield width,
vertical is scroll distance, and the blue edges are read from the live `PlayfieldConfig`, so what you see
is exactly what the camera will show.

<img width="1127" height="1235" alt="Stack Attack Level Editor" src="https://github.com/user-attachments/assets/d28bc7c8-f1a8-4d46-9bc4-fe33ff329e35" />

**What it does**

- **Paint and drag.** Click the field to drop a group with the current brush; drag any group to move it in
  both axes at once. `xPosition` and `distance` are written back through `SerializedProperty`, so undo,
  the dirty flag and multi-object editing all behave the way Unity expects.
- **Snapping and zoom.** Snap to `0.1 / 0.25 / 0.5 / 1` world units or turn it off entirely, with a zoom
  slider for scrubbing along a long level.
- **Boss brush.** Toggling **Boss** switches the brush to a boss entry — the guard count, ring radius and
  spin speed in the toolbar become the ring that orbits the boss body at runtime.
- **Reads the real data.** Group footprints are drawn from the actual `StackTypeConfig` plate size and
  plate count, horizontally moving groups have their sweep shaded so you can see the lane they will
  occupy, and overlapping groups are outlined in red before you ever press Play.
- **Live inspector panel.** The right-hand panel edits the selected entry in full — layout, motion, HP,
  spacing, radius, spin — and shows the level duration in seconds next to its length in units.

A matching custom inspector on `LevelConfig` lets the same entries be edited from the Project window
without opening the tool.

---

## Levels

Five hand-authored levels, each a `LevelConfig` asset under
`Assets/_Project/ScriptableObjects/Levels/Levels/`.

| Level | Length | Focus |
| --- | --- | --- |
| **1** | 20 | Teaches the loop — single stacks, then rows, then the first ring. |
| **2** | 25 | Introduces the armored type (4 hits per plate) and moving groups. |
| **3** | 28 | Mixed types and tighter lanes; the first sustained pressure. |
| **4** | 29 | Clusters and orbiting rings — the field starts fighting back. |
| **5** | 35 | The long one: a boss encounter with a spinning guard ring, with everything above layered underneath it. |

Three stack archetypes drive the difficulty curve — **Swarm** (1 hit per plate, many towers),
**Normal** (2) and **Armored** (4).

---

## Architecture

The project is deliberately built the way a shipped title is, not the way a prototype is.

### Dependency injection, no singletons

Everything is composed through **VContainer**. There is no `Instance`, no service locator and no static
access point anywhere in the project. `GameScope` is the single composition root and every feature ships
its own installer:

```csharp
protected override void Configure(IContainerBuilder builder)
{
    builder.RegisterInstance(playfieldConfig);
    builder.RegisterInstance(new GameStateMachine());
    builder.RegisterEntryPoint<ApplicationBootstrap>().WithParameter(targetFrameRate);

    new AudioInstaller(audioBank).Install(builder);
    new PlayerInstaller(playerConfig, weaponConfig, healthConfig).Install(builder);
    new PlayfieldInstaller().Install(builder);
    new UpgradesInstaller(upgradePool).Install(builder);
    new StackEnemyInstaller(stackGroupPrefab, bossPrefab).Install(builder);
    new PersistenceInstaller().Install(builder);
    new LevelInstaller(levelSet).Install(builder);
    new ScreensInstaller().Install(builder);
}
```

### Assembly definitions enforce the boundaries

Each feature has its own `.asmdef` that references **only** `Core`, `VContainer` and `UniTask` — never
another feature. Two features that need to talk do it through an interface or an event defined in `Core`,
which means the compiler, not a convention, is what keeps the dependency graph acyclic.

```
Core  <--  Audio · Level · Persistence · Player · Playfield · Screens · StackEnemy · Upgrades
                                  ^
                                 App   (composition root, references everything)
```

`Core` holds the contracts and the pure logic that has no scene behind it: `IStackSpawner`, `IBossArena`,
`IScoreSink`, `IUpgradeService`, `IAudioService`, `IProgressRepository`, `GameStateMachine`, `RunScore`,
`PlayerHealth`, `WeaponStats` and `HitFeedback`.

### Thin MonoBehaviours

Game logic lives in plain C# classes that can be constructed and tested without a scene —
`PlayerMovement`, `AutoWeapon`, `SkillTimer`, `RunScore`, `UpgradeService`, `HitFeedback`. The
MonoBehaviours only adapt the Unity lifecycle to them:

```csharp
private void Update()
{
    float deltaTime = Time.deltaTime;

    if (_state.Current == GameState.Playing && _weapon.Tick(deltaTime, _input.IsPressed))
        Fire();

    Punch(deltaTime);
}
```

### Fail-fast dependencies

A dependency is guaranteed one of three ways — DI, inspector wiring, or `[RequireComponent]` +
`GetComponent`. None of those is null-checked at runtime. If one is null the setup is wrong, and the
project would rather throw on the first frame than paper over a wiring bug with a fallback. Null checks
appear only where a value is *genuinely* optional: raycast hits, `TryGetComponent`, dictionary lookups,
an audio entry the designer has not filled in yet.

### UniTask, not coroutines

All async work goes through **UniTask** with a `CancellationToken` tied to the object's lifetime. No
coroutines, no `System.Threading.Tasks.Task`, and no `async void` outside Unity's own event methods. The
end screen is a good example: the title animates in, the hint follows it, and input stays gated until the
hint is on screen — all on unscaled time, because a run can end straight out of the upgrade panel, which
leaves `Time.timeScale` at zero.

---

## Performance

- **Object pooling.** Projectiles, rockets and boomerangs all come from a shared generic `ThrownPool<T>`
  built on `UnityEngine.Pool.ObjectPool<T>` with collection checks enabled. An active-list guard makes a
  double release impossible, which is what lets a rocket call `Explode()` twice without corrupting the
  pool. Trails are cleared *after* the reposition, never before, so a recycled projectile cannot draw a
  streak back to where the last one died.
- **No lookups in the hot path.** `GetComponent*`, `Camera.main` and `Find*` are called once in `Awake`
  and never from `Update`, `FixedUpdate` or `LateUpdate`.
- **Frame pacing.** `ApplicationBootstrap` disables vSync before setting `Application.targetFrameRate`,
  because the editor honours vSync and the device ignores it — without that, editor testing would not
  match the build. Screen sleep is disabled for the reason a runner needs it: the level keeps moving while
  the player only reacts.
- **Events, not polling.** The HUD subscribes to `RunScore.Changed` and to the player's health rather than
  re-reading them every frame.

---

## Audio

`IAudioService` and the `GameSound` enum live in `Core`, so any feature can ask for a sound without taking
a dependency on the audio feature. The implementation resolves the clip from an `AudioBank`
ScriptableObject, applies per-entry volume and pitch jitter, and hands it to a round-robin voice pool so a
sound never cuts off the one before it.

Two details worth calling out: a sound with no clip attached is simply silent rather than an error — an
unfinished bank is a normal state for a designer to be in — and the same sound is allowed **one voice per
frame**, because a pierced shot breaking three plates on one frame is a click, not three times the volume.

---

## Project Layout

```
Assets/_Project/
  App/            GameScope (composition root), ApplicationBootstrap
  Art/            Sprites, fonts, shaders, audio — trimmed to what is actually used
  Core/           Contracts and pure logic. Depends on nothing.
  Features/
    Audio/        AudioBank · AudioPlayer · AudioService · AudioInstaller
    Level/        LevelConfig · LevelSet · LevelRunner, plus the Level Editor window
    Persistence/  PlayerPrefs-backed progress repository
    Player/       Movement · Weapon (projectile, rocket, boomerang, pool) · Health
    Playfield/    Camera framing and playfield bounds
    Screens/      Start · HUD · Upgrade · End, and the router that switches them
    StackEnemy/   StackGroup · StackBehaviour · StackSpawner · ShardBurst · Boss
    Upgrades/     UpgradePool · UpgradeService
  Prefabs/
  Scenes/         GameScene
  ScriptableObjects/
```

Every tunable value is a ScriptableObject: `PlayfieldConfig`, `PlayerConfig`, `WeaponConfig`,
`HealthConfig`, `StackTypeConfig` (×3), `BossConfig`, `UpgradePool`, `AudioBank`, `LevelSet` and the five
`LevelConfig` assets. No balance number is compiled into a script.

---

## Getting Started

1. Open the project with **Unity 6000.0.58f2** (or a newer 6000.0.x).
2. Open `Assets/_Project/Scenes/GameScene.unity`.
3. Set the Game view to a **portrait** aspect — 9:16 or 1080×1920.
4. Press Play. Drag to steer, hold to fire.

Progress is stored in PlayerPrefs under `stackattack.last_level_index`; delete that key to start over from
level 1.

To play without opening the project, grab the APK from the
[latest release](../../releases/latest).

### Tech

| | |
| --- | --- |
| Engine | Unity 6000.0.58f2, Universal RP 17.0.4 (2D Renderer) |
| DI | VContainer 1.19.0 |
| Async | UniTask |
| Input | Input System 1.14.2 |
| UI | uGUI + TextMeshPro |

---

## How This Was Built

This project was written with **Claude Code**, driven by a project-local workflow I built to keep the AI
inside an architecture I chose rather than one it drifts into. The whole setup is committed under
[`.claude/`](.claude) — it is part of the repository, not something that happened off-screen.

It has three layers.

**Rules.** Six files under [`.claude/rules/`](.claude/rules) fix the decisions up front: VContainer for
composition, UniTask with lifetime-bound cancellation for async, a ScriptableObject for every tunable
value, plain C# services behind thin MonoBehaviours, and no defensive null-fallback on dependencies that
are guaranteed by construction. These are the same conventions listed at the bottom of this README — the
model reads them as constraints, a human reads them as documentation.

**Hooks that actually block.** A rule in a prompt is advice. These are enforcement: four PowerShell
hooks in [`.claude/hooks/`](.claude/hooks) run on every `Write` and `Edit` and exit non-zero to refuse
the edit outright.

| Hook | Refuses |
| --- | --- |
| `block-coroutine.ps1` | Coroutines — async goes through UniTask or not at all |
| `block-singleton.ps1` | Singletons and service locators — composition belongs in `GameScope` |
| `block-region.ps1` | `#region` — a file that needs folding needs splitting instead |
| `block-scene-prefab-asset-edit.ps1` | Direct text edits to `.unity`, `.prefab` and `.asset` files |

The last one matters most. Unity's serialized files are not safe to hand-edit, so the AI was never
allowed near them: **every scene, prefab and ScriptableObject in this project was wired by hand in the
Unity Editor.** The model wrote C# and nothing else.

**A separate reviewer.** [`code-reviewer`](.claude/agents/code-reviewer.md) is a second agent with
read-only tools that audits changes against the rules before a commit — lifecycle hazards, silent
failures, `GetComponent` in a hot path, fat MonoBehaviours that should delegate. Implementation and
review never share a context, because the model that just wrote the code is the worst available judge
of it.

The constraints are the point. Generating C# stopped being the hard part some time ago; deciding what
the code is *not* allowed to do is where the engineering moved. The same system, generalised and
installable, is published separately as the
[Unity AI Workflow Kit](https://github.com/Alhnzgrr/unity-ai-workflow-kit).

---

## Conventions

- Conventional Commits without a scope — `feat: add drag movement`.
- Explicit types everywhere: no `var`, no region directives, no XML doc comments.
- `private Foo _field;` for pure private state, `[SerializeField] private Foo field;` for anything exposed
  in the inspector — the missing underscore is the signal.
- Comments explain *why* a piece of code is shaped the way it is, never *what* it does.
