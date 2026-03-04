# Setup Guide

## Prerequisites

- Unity 2022 LTS (2D URP template)
- Git
- Claude Code CLI (for AgentPilot meta-commands)
- Node.js >= 18 (for AgentPilot)
- Python / uv (for Unity MCP server)

## Installation

```bash
# Clone both repos
git clone <your-remote>/tomato-fighters.git
git clone <your-remote>/tomato-fighters-docs.git

# Open Unity project
# Unity Hub → Open → select tomato-fighters/ directory
# Wait for initial import and compilation
```

## Project Structure

```
TomatoFighters/
├── tomato-fighters/          ← Open this in Unity
│   ├── Assets/Scripts/       ← Your code goes here
│   ├── .claude/CLAUDE.md     ← Project conventions (agents read this)
│   └── config-bank/          ← AgentPilot configs
└── tomato-fighters-docs/     ← Open this for reference
    ├── TASK_BOARD.md          ← Your task assignments
    ├── TASK_LOGBOOK.md        ← Log completed work here
    └── developer/dev{N}*.md   ← Your personal guide
```

## Building Game Assets (Editor Scripts)

We don't create prefabs, ScriptableObjects, or animation controllers by hand. Instead, we use Unity editor scripts that generate everything from code. If something in the design changes — new attacks, rebalanced stats, different combo flow — we update the relevant editor script and re-run it. The scripts are the single source of truth for all generated assets.

After cloning, run these from the Unity menu bar to set up all assets:

```
1. TomatoFighters > Import Sprite Sheets       ← slice animation PNGs
2. TomatoFighters > Build Animations            ← create .anim clips + AnimatorController
3. TomatoFighters > Create Player Prefab        ← Player.prefab with all components
4. Tools > TomatoFighters > Create Mystica Attacks          ← base attack data
5. Tools > TomatoFighters > Create All Character Attacks    ← all 26 attack assets
6. Tools > TomatoFighters > Create Combo Definitions        ← combo trees + configs
7. TomatoFighters > Create All Path Assets                  ← 12 upgrade paths
8. Tools > TomatoFighters > Assign All Hitbox IDs           ← wire attacks to hitbox shapes
9. Tools > TomatoFighters > Setup Mystica Character         ← hitbox colliders on prefab
10. TomatoFighters > Create Movement Test Scene             ← playable test arena
```

Full details, dependency graph, and workflow recipes (e.g., "what to re-run when new animation GIFs arrive") are in [Unity Editor Scripts](unity-editor-scripts.md).

---

## Unity MCP (Model Context Protocol)

The project uses [MCP for Unity](https://github.com/CoplayDev/unity-mcp) to give Claude Code direct access to the Unity Editor — reading scene hierarchies, inspecting components, running editor scripts, and more.

### How It Works

1. **Unity side:** The `com.coplaydev.unity-mcp` package runs inside the Unity Editor and exposes an HTTP bridge on `http://127.0.0.1:8080`.
2. **Claude Code side:** An MCP server process (`mcp-for-unity`) connects to that bridge and translates Claude tool calls into Unity Editor actions.

### Setup

The MCP server is already configured in `~/.claude/settings.json`:

```json
"mcp-for-unity": {
  "command": "C:\\Users\\taldi\\.local\\bin\\uvx.exe",
  "args": [
    "--from", "mcpforunityserver==9.4.7",
    "mcp-for-unity",
    "--transport", "http",
    "--http-url", "http://127.0.0.1:8080",
    "--project-scoped-tools"
  ]
}
```

### Usage

1. **Open the Unity project** in the Unity Editor and wait for compilation to finish.
2. **Verify the MCP bridge is running** — in Unity, check `Window > MCP for Unity` or look for the MCP status indicator. The bridge should be listening on port 8080.
3. **Start Claude Code** — the `mcp-for-unity` server will launch automatically and connect to Unity.
4. Claude Code now has access to Unity-specific tools (scene inspection, component queries, asset operations, etc.) through the MCP protocol.

### Troubleshooting

| Problem | Fix |
|---------|-----|
| MCP tools not appearing in Claude Code | Restart Claude Code after verifying Unity is open and the bridge is running |
| Connection refused on port 8080 | Open the Unity project first — the bridge only runs while the Editor is active |
| `uvx` not found | Install [uv](https://docs.astral.sh/uv/getting-started/installation/) — `uvx` is included |
| Version mismatch | Update the version in `~/.claude/settings.json` to match the Unity package version |

---

## Running Tasks

All 3 devs use the same meta-commands from the workspace root (`/Users/ofek/Projects/Claude/`):

```bash
# Execute a single task
/do-task "T002: CharacterController2D — 8-dir movement, jump, armored dash for Brutor"

# Execute from the task board
/task-execute

# Run an entire phase
/execute-phase 1

# Full orchestration (plan → execute → document)
/build-app "TomatoFighters" unity-game "Phase 1 Foundation sprint"
```

## Git Workflow

```bash
# Create your feature branch
git checkout -b pillar1/combat-controller    # Dev 1
git checkout -b pillar2/roguelite-paths      # Dev 2
git checkout -b pillar3/world-enemies        # Dev 3

# Work, commit
git add Assets/Scripts/Combat/CharacterController2D.cs
git commit -m "[Phase 1] T002: CharacterController2D with armored dash"

# Push to your pillar branch
git push origin pillar1/combat-controller

# Integration (all 3 devs together)
git checkout integration/sprint-1
git merge pillar1/combat-controller
git merge pillar2/roguelite-paths
git merge pillar3/world-enemies
```

## Useful Commands

| Command | Description |
|---------|-------------|
| `/do-task "..."` | Execute single task through 8-step pipeline |
| `/task-execute` | Execute task from task board |
| `/execute-phase N` | Run all tasks in phase N |
| `/scan-repo` | Re-index project after major changes |
| `/capture-learnings TomatoFighters` | Extract patterns after a phase |
