# TOOL-001: Prompt-Driven VFX Particle System Generator

## Metadata
| Field | Value |
|-------|-------|
| **Phase** | Tools |
| **Type** | tooling |
| **Priority** | P1 |
| **Owner** | Dev 1 |
| **Depends On** | None |
| **Blocks** | None |
| **Status** | PLANNED |

## Objective
Build a Unity Editor tool that generates fully configured ParticleSystem prefabs from natural language prompts using the Claude API. The tool provides a Generate + Refine workflow, saving prefabs to `Assets/Prefabs/VFX/` and dropping a preview instance into the active scene.

## Context
Tomato Fighters currently has zero particle system infrastructure — all visual feedback uses sprite color modulation (TelegraphVisualController). The game will need dozens of VFX effects across combat (hit impacts, ability visuals), roguelite (ritual effects, path ability auras), and world (environmental particles, status effect indicators) pillars.

Manual ParticleSystem tuning is tedious. This tool accelerates VFX iteration by letting developers describe effects in plain English and get a ready-to-use prefab in seconds.

## Architecture

### Data Flow
```
ParticleSystemGenerator (EditorWindow)
  │
  ├── [Generate] button
  │     → ClaudeApiClient.SendPrompt(userPrompt, jsonSchema)
  │     → Claude returns JSON matching ParticleConfig schema
  │     → ParticleConfigSchema deserializes JSON → ParticleConfig object
  │     → ParticleSystemApplier.Apply(config) → creates GameObject with ParticleSystem
  │     → Save as prefab to Assets/Prefabs/VFX/{Name}.prefab
  │     → Instantiate preview in active scene
  │
  └── [Refine] button
        → ClaudeApiClient.SendRefinement(previousConfigJson, refinementPrompt)
        → Same pipeline: deserialize → apply → overwrite prefab → update scene instance
```

### File Plan

| # | File | Purpose |
|---|------|---------|
| 1 | `Assets/Editor/VFX/ParticleSystemGenerator.cs` | EditorWindow — UI layout, button handlers, orchestration |
| 2 | `Assets/Editor/VFX/ParticleConfigSchema.cs` | C# data classes matching the JSON schema Claude returns |
| 3 | `Assets/Editor/VFX/ClaudeApiClient.cs` | `System.Net.Http.HttpClient` wrapper — prompt → JSON response |
| 4 | `Assets/Editor/VFX/ParticleSystemApplier.cs` | Maps deserialized `ParticleConfig` onto Unity `ParticleSystem` modules |

All files live in `Assets/Editor/` — they are **Editor-only** and excluded from runtime builds.

## Requirements

### 1. ParticleSystemGenerator.cs — Editor Window

**Window layout:**
```
┌─ VFX Generator ──────────────────────────┐
│                                           │
│  Prompt: [fire burst on impact        ]   │
│                                           │
│  Name:   [FireBurstImpact             ]   │
│          (auto-generated if blank)        │
│                                           │
│  [Generate]              [Refine]         │
│                                           │
│  Refine: [make it bigger, more gravity]   │
│  (shown after first Generate)             │
│                                           │
│  ── Status ────────────────────────────── │
│  ✓ Generated: Assets/Prefabs/VFX/        │
│    FireBurstImpact.prefab                 │
│                                           │
│  ── API Settings (foldout) ────────────── │
│  API Key: [••••••••••••••]  [Save]        │
│                                           │
└───────────────────────────────────────────┘
```

**Behavior:**
- Menu item: `TomatoFighters/VFX Generator`
- Prompt field: `EditorGUILayout.TextField`, multi-line
- Name field: auto-populated by stripping spaces/special chars from prompt in PascalCase. Editable.
- Generate button: disabled while API call in progress. Calls `ClaudeApiClient.SendPrompt()`, pipes result through `ParticleConfigSchema` deserialization, then `ParticleSystemApplier.Apply()`, saves prefab, instantiates in scene.
- Refine button: visible only after a successful Generate. Takes refinement prompt + previous config JSON. Overwrites existing prefab and replaces scene instance.
- Status area: scrollable log showing success/error messages with timestamps.
- API Settings foldout: API key stored in `EditorPrefs.SetString("TomatoFighters_ClaudeAPIKey", key)`. Masked display. Save button writes to EditorPrefs.

**Error handling:**
- No API key → status shows "Set your Claude API key in API Settings"
- Network failure → status shows error message, re-enables Generate button
- JSON parse failure → status shows "Failed to parse response" + raw response for debugging

### 2. ParticleConfigSchema.cs — JSON Contract

The schema defines what Claude returns. Must be simple enough for reliable LLM output.

```csharp
[System.Serializable]
public class ParticleConfig
{
    public MainModuleConfig main;
    public EmissionConfig emission;
    public ShapeConfig shape;
    public ColorOverLifetimeConfig colorOverLifetime;
    public SizeOverLifetimeConfig sizeOverLifetime;
}

[System.Serializable]
public class MainModuleConfig
{
    public float duration;           // seconds (default 1.0)
    public bool looping;             // default false
    public float startLifetime;      // seconds (default 1.0)
    public float startSpeed;         // units/sec (default 5.0)
    public float startSize;          // units (default 0.5)
    public ColorRGBA startColor;     // RGBA 0-1
    public float gravityModifier;    // 0 = no gravity, 1 = full (default 0)
    public string simulationSpace;   // "Local" or "World" (default "Local")
    public int maxParticles;         // default 100
}

[System.Serializable]
public class EmissionConfig
{
    public float rateOverTime;       // particles/sec (default 10)
    public BurstConfig[] bursts;     // optional burst emissions
}

[System.Serializable]
public class BurstConfig
{
    public float time;               // time in seconds within duration
    public int count;                // number of particles
}

[System.Serializable]
public class ShapeConfig
{
    public string shapeType;         // "Cone", "Sphere", "Circle", "Edge", "Box"
    public float angle;              // cone angle in degrees (default 25)
    public float radius;             // shape radius (default 1.0)
    public float arc;                // arc degrees (default 360)
}

[System.Serializable]
public class ColorOverLifetimeConfig
{
    public GradientStop[] gradient;  // array of color stops
}

[System.Serializable]
public class GradientStop
{
    public float time;               // 0-1 normalized
    public ColorRGBA color;          // RGBA 0-1
}

[System.Serializable]
public class SizeOverLifetimeConfig
{
    public CurvePoint[] curve;       // array of size curve points
}

[System.Serializable]
public class CurvePoint
{
    public float time;               // 0-1 normalized
    public float value;              // size multiplier 0-1
}

[System.Serializable]
public class ColorRGBA
{
    public float r, g, b, a;        // all 0-1
}
```

**Design rationale:** Arrays of simple structs (GradientStop, CurvePoint) instead of Unity's complex `MinMaxGradient`/`MinMaxCurve` types. This keeps the JSON schema LLM-friendly. The Applier converts these to Unity types.

### 3. ClaudeApiClient.cs — API Integration

**HTTP client:** `System.Net.Http.HttpClient` (static, reused).

**API call:**
```
POST https://api.anthropic.com/v1/messages
Headers:
  x-api-key: {key from EditorPrefs}
  anthropic-version: 2023-06-01
  content-type: application/json
Body:
  model: claude-sonnet-4-20250514
  max_tokens: 2048
  system: {SYSTEM_PROMPT}
  messages: [{ role: "user", content: {prompt} }]
```

**System prompt** (baked into ClaudeApiClient):
```
You are a Unity VFX designer. Given a description of a visual effect, return a JSON
object matching the ParticleConfig schema. Return ONLY valid JSON, no markdown fencing,
no explanation.

Schema:
{paste full schema here}

Guidelines:
- For fire effects: warm colors (orange/red/yellow), upward velocity, short lifetime
- For ice/frost: cool colors (blue/white/cyan), slow speed, gravity near 0
- For explosions: use bursts not rateOverTime, sphere shape, high startSpeed
- For auras/buffs: looping=true, circle shape, low speed, long duration
- For trails: edge shape, world space, stretched billboard
- For impacts: short duration, burst emission, cone shape pointing away from surface
- For electric/lightning: bright blue/white, high speed, short lifetime, many particles
- Keep startSize between 0.1 and 2.0 for most effects
- Keep maxParticles between 20 and 500
- Use 3-5 gradient stops for smooth color transitions
- Use 2-4 curve points for size over lifetime
```

**Refinement call:** Same endpoint, but the user message includes:
```
Previous configuration:
{previousConfigJson}

Refinement request: {userRefinementPrompt}

Return the complete updated ParticleConfig JSON.
```

**Response parsing:**
1. Extract `content[0].text` from API response
2. Strip any accidental markdown fencing (```json ... ```)
3. Deserialize via `JsonUtility.FromJson<ParticleConfig>()`
4. On failure, throw with raw response text for debugging

### 4. ParticleSystemApplier.cs — Config → ParticleSystem

**Responsibility:** Takes a `ParticleConfig` and creates/configures a `GameObject` with a `ParticleSystem`.

**Module mapping:**

```csharp
public static GameObject Apply(ParticleConfig config, string name)
{
    var go = new GameObject(name);
    var ps = go.AddComponent<ParticleSystem>();

    // Main module
    var main = ps.main;
    main.duration = config.main.duration;
    main.loop = config.main.looping;
    main.startLifetime = config.main.startLifetime;
    main.startSpeed = config.main.startSpeed;
    main.startSize = config.main.startSize;
    main.startColor = ToColor(config.main.startColor);
    main.gravityModifier = config.main.gravityModifier;
    main.simulationSpace = ParseSimulationSpace(config.main.simulationSpace);
    main.maxParticles = config.main.maxParticles;

    // Emission module
    var emission = ps.emission;
    emission.rateOverTime = config.emission.rateOverTime;
    if (config.emission.bursts != null)
    {
        foreach (var burst in config.emission.bursts)
            emission.SetBurst(i, new ParticleSystem.Burst(burst.time, burst.count));
    }

    // Shape module
    var shape = ps.shape;
    shape.shapeType = ParseShapeType(config.shape.shapeType);
    shape.angle = config.shape.angle;
    shape.radius = config.shape.radius;
    shape.arc = config.shape.arc;

    // Color over Lifetime
    var col = ps.colorOverLifetime;
    col.enabled = true;
    col.color = BuildGradient(config.colorOverLifetime.gradient);

    // Size over Lifetime
    var sol = ps.sizeOverLifetime;
    sol.enabled = true;
    sol.size = BuildCurve(config.sizeOverLifetime.curve);

    // Renderer — default particle material
    var renderer = go.GetComponent<ParticleSystemRenderer>();
    renderer.material = AssetDatabase.GetBuiltinExtraResource<Material>(
        "Default-Particle.mat");

    return go;
}
```

**Helper methods:**
- `ToColor(ColorRGBA)` → `Color`
- `ParseSimulationSpace(string)` → `ParticleSystemSimulationSpace`
- `ParseShapeType(string)` → `ParticleSystemShapeType`
- `BuildGradient(GradientStop[])` → `ParticleSystem.MinMaxGradient` from `Gradient` with color/alpha keys
- `BuildCurve(CurvePoint[])` → `ParticleSystem.MinMaxCurve` from `AnimationCurve` with keyframes

## Design Decisions

### DD-1: LLM-Powered Generation (vs Rule Engine)
**Decision:** Use Claude API to generate particle configs from natural language prompts.
**Rationale:** Maximum expressiveness — handles novel descriptions without a predefined keyword vocabulary. The tradeoff is API dependency, which is acceptable for an Editor-only tool.

### DD-2: System.Net.HttpClient (vs UnityWebRequest)
**Decision:** Use `System.Net.Http.HttpClient` for API calls.
**Rationale:** Editor-only context doesn't need Unity's async infrastructure. HttpClient gives cleaner synchronous code. Avoids `EditorCoroutineUtility` dependency.

### DD-3: Focused Module Set (5 modules in v1)
**Decision:** Support Main, Emission, Shape, ColorOverLifetime, SizeOverLifetime only.
**Rationale:** These 5 modules cover ~80% of common particle effects. Deferred modules (Noise, Trails, SubEmitters, TextureSheetAnimation, VelocityOverLifetime) can be added incrementally by extending the schema and applier.

### DD-4: Simple Array Schema (vs Unity Native Types)
**Decision:** JSON schema uses `GradientStop[]` and `CurvePoint[]` instead of Unity's `MinMaxGradient`/`MinMaxCurve` serialization.
**Rationale:** LLMs generate simple arrays reliably. Unity's native curve/gradient serialization is complex and error-prone for LLM output. The Applier handles the conversion.

### DD-5: Prefab + Scene Instance Output
**Decision:** Generate saves a prefab AND drops an instance into the active scene.
**Rationale:** Fastest iteration loop — developer sees the result immediately without manual drag-and-drop. Prefab is saved for permanent use.

### DD-6: Refine Workflow
**Decision:** Refine sends previous config JSON + new instruction, returns complete updated config.
**Rationale:** Sending the full previous config ensures Claude has complete context. Returning a complete config (not a diff) simplifies the applier — just overwrite everything.

### DD-7: API Key in EditorPrefs
**Decision:** Store API key in `EditorPrefs`, not in any file.
**Rationale:** EditorPrefs is per-machine and never committed to git. No risk of key leakage.

## Execution Order

1. **ParticleConfigSchema.cs** — Define the data classes first (the contract everything depends on)
2. **ClaudeApiClient.cs** — API integration (depends on schema for system prompt)
3. **ParticleSystemApplier.cs** — Config → ParticleSystem mapping (depends on schema)
4. **ParticleSystemGenerator.cs** — Editor Window that wires everything together (depends on all above)

## Future Extensions (Out of Scope for v1)
- Additional modules: Noise, Trails, SubEmitters, VelocityOverLifetime, TextureSheetAnimation
- Preset library: save/load named configs as ScriptableObjects
- Batch generation: generate multiple variants from one prompt
- Custom materials/textures: let the prompt specify particle texture
- Integration with Animation Events: auto-wire VFX to attack hitbox timing
