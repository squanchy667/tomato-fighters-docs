# ANIMATION FORGE — Claude Code Implementation Plan
> Feed this entire file to Claude Code to build the project from scratch.
> Every section is an actionable instruction. Follow them in order.

---

## PROJECT IDENTITY

**Name:** animation_forge  
**Purpose:** CLI pipeline that converts AI-generated video files into Unity-ready 2D animation packages  
**Input:** One or more `.mp4` / `.mov` video files (4–10 seconds, 24fps)  
**Output:** Per-animation PNG spritesheets + Unity AnimatorController scaffold + metadata  
**Language:** Python 3.10+  
**Target User:** Game developer running this locally on Mac/Windows/Linux  

---

## REPO STRUCTURE TO CREATE

```
animation_forge/
├── main.py                          # CLI entry point
├── requirements.txt                 # All dependencies
├── README.md                        # Usage guide
├── config/
│   └── animation_types.json         # Master list of animation types + metadata
├── phases/
│   ├── __init__.py
│   ├── p0_bootstrap.py              # Video analysis + frame sampling
│   ├── p1_questionnaire.py          # Interactive Q&A to map videos → animations
│   ├── p2_extract.py                # ffmpeg frame extraction
│   ├── p3_bg_removal.py             # rembg (primary) + numpy fallback
│   ├── p4_segmentation.py           # Frame range slicing per animation
│   └── p5_export.py                 # Spritesheet packing + Unity package
├── utils/
│   ├── __init__.py
│   ├── vision.py                    # Claude API helpers for frame analysis
│   ├── spritesheet.py               # PIL packing utilities
│   ├── unity_export.py              # AnimatorController + .anim file generators
│   └── session.py                   # session_config.json read/write
└── templates/
    ├── animator_controller.json.tmpl
    └── import_guide.md.tmpl
```

---

## PHASE 0 — BOOTSTRAP & ENVIRONMENT CHECK

### File: `requirements.txt`

```
# Core
Pillow>=10.0.0
numpy>=1.24.0
scipy>=1.10.0

# Video
# ffmpeg must be installed on system (not pip) — check in code

# Background removal (primary)
rembg[cpu]>=2.0.50
onnxruntime>=1.16.0

# Pose estimation (segmentation)
mediapipe>=0.10.0

# Claude API (Vision + questionnaire assistance)
anthropic>=0.25.0

# CLI
click>=8.1.0
rich>=13.0.0          # pretty terminal output
```

### File: `config/animation_types.json`

```json
{
  "animations": [
    { "id": "idle",      "category": "movement", "loop": true,  "typical_frames": [8,16],  "description": "Standing still, breathing, minor sway" },
    { "id": "walk",      "category": "movement", "loop": true,  "typical_frames": [10,14], "description": "Walking forward at normal pace" },
    { "id": "run",       "category": "movement", "loop": true,  "typical_frames": [8,12],  "description": "Running / sprinting" },
    { "id": "jump",      "category": "airborne", "loop": false, "typical_frames": [6,10],  "description": "Rising phase of a jump" },
    { "id": "apex",      "category": "airborne", "loop": true,  "typical_frames": [4,6],   "description": "Peak of jump / floating" },
    { "id": "fall",      "category": "airborne", "loop": false, "typical_frames": [4,8],   "description": "Falling down" },
    { "id": "land",      "category": "airborne", "loop": false, "typical_frames": [4,6],   "description": "Landing impact" },
    { "id": "attack_1",  "category": "combat",   "loop": false, "typical_frames": [10,18], "description": "First attack / light attack" },
    { "id": "attack_2",  "category": "combat",   "loop": false, "typical_frames": [10,18], "description": "Second attack / heavy attack" },
    { "id": "attack_3",  "category": "combat",   "loop": false, "typical_frames": [10,18], "description": "Third attack / special" },
    { "id": "dash",      "category": "combat",   "loop": false, "typical_frames": [8,12],  "description": "Quick dash / dodge" },
    { "id": "block",     "category": "combat",   "loop": true,  "typical_frames": [4,8],   "description": "Blocking / guarding stance" },
    { "id": "hurt",      "category": "reaction", "loop": false, "typical_frames": [6,10],  "description": "Taking damage / flinch" },
    { "id": "death",     "category": "reaction", "loop": false, "typical_frames": [16,24], "description": "Death sequence" }
  ]
}
```

### File: `utils/session.py`

Build a simple session state manager:

```python
# session_config.json schema:
{
    "session_id": "string (uuid4)",
    "character_name": "string",
    "created_at": "ISO timestamp",
    "phases_completed": ["p0", "p1", ...],
    "videos": {
        "filename.mp4": {
            "path": "absolute path",
            "fps": 24,
            "duration": 6.04,
            "total_frames": 145,
            "width": 464,
            "height": 688,
            "sampled_frames": ["path/to/frame_0.png", ...],
            "vision_description": "string from Claude Vision"
        }
    },
    "animation_map": {
        "walk": {
            "video": "filename.mp4",
            "frame_start": 0,
            "frame_end": 24,
            "fps": 24,
            "loop": true,
            "confirmed": true
        }
    },
    "output_dir": "string",
    "bg_removal_method": "rembg_isnet-anime | rembg_u2net | numpy_fallback"
}
```

Implement: `load_session(path)`, `save_session(data, path)`, `new_session(output_dir)`.

---

## PHASE 1 — MAIN CLI ENTRY POINT

### File: `main.py`

Use `click` for the CLI. Implement these commands:

```bash
# Full pipeline (interactive)
python main.py run --video mage_walk.mp4 --video mage_attack.mp4 --character "mage"

# Resume an interrupted session
python main.py resume --session ./output/session_abc123/session_config.json

# Run only specific phases
python main.py run --video input.mp4 --skip-questionnaire --phases 2,3,5

# Preview only (no export, shows frame samples)
python main.py preview --video input.mp4
```

**CLI behavior:**
- Print a styled ASCII banner on startup using `rich`
- Show a phase progress bar: `[Phase 1/6] Questionnaire ████░░ 60%`
- All errors must be caught and displayed with `rich` Panel (red border), never raw tracebacks
- On completion, print a summary box with all output file paths and sizes

---

## PHASE 2 — BOOTSTRAP (`p0_bootstrap.py`)

### What it does:
1. Check that `ffmpeg` and `ffprobe` are installed (`subprocess.run(['ffprobe', '-version'])`)
2. Check rembg availability (`try: from rembg import remove`)
3. Check mediapipe availability
4. For each input video, run `ffprobe -v quiet -print_format json -show_streams`
5. Extract: fps, duration, total_frames, width, height
6. Sample 5 evenly-spaced frames using ffmpeg `-ss` seeking
7. Optionally send samples to Claude Vision API for character description
8. Write everything to `session_config.json`

### Claude Vision call (in `utils/vision.py`):

```python
def describe_character_from_frames(frame_paths: list[str], client) -> str:
    """
    Send up to 5 frames to Claude Vision.
    Returns a description of the character and what movement is visible.
    """
    # Build message with images
    # Prompt: "You are analyzing frames from a game character animation video.
    #          Describe: 1) The character's visual style and appearance
    #          2) What movement/action appears to be happening
    #          3) What Unity animation type this most likely corresponds to
    #          (idle/walk/run/jump/attack/dash/hurt/death)
    #          Be concise. 2-3 sentences max."
```

**Implementation note:** Make Vision optional — if `ANTHROPIC_API_KEY` not set, skip and print warning. Never crash if Vision unavailable.

---

## PHASE 3 — QUESTIONNAIRE (`p1_questionnaire.py`)

### What it does:
Interactive terminal Q&A that maps each video → animation type(s).

**For each video, ask in sequence:**

```
Q1: "What is the character's name?" 
    [default: detected from Vision or "character"]

Q2: "What type of animation is in this video?"
    Show numbered list from animation_types.json
    [default: Vision best guess if available]
    Allow "multiple" answer → triggers frame range mode

Q3 (if multiple): "This video has multiple animations. 
    I'll show you timestamps. Enter frame ranges like: walk=0-24, attack=25-60"
    Show frame count and duration to help user

Q4: "Is this animation a loop? (walk/idle/run typically loop)"
    [default: from animation_types.json loop field]

Q5: "Confirm: {animation_id} covers frames {start}-{end} of {video}. Correct? [Y/n]"
```

**Rich formatting:**
- Show a mini "film strip" of sampled frames as file paths with frame numbers
- Use `rich.prompt.Prompt.ask()` for all inputs
- Use `rich.table.Table` to show the animation_map being built in real time

**Save to session after each confirmed answer.**

---

## PHASE 4 — FRAME EXTRACTION (`p2_extract.py`)

### What it does:
Run ffmpeg to extract ALL frames from each video that appears in `animation_map`.

```python
def extract_frames(video_path: str, output_dir: str, fps: int) -> list[str]:
    """
    Runs: ffmpeg -i {video_path} -r {fps} {output_dir}/frame_%04d.png
    Returns list of extracted frame paths.
    Shows progress with rich.progress.
    """
```

**Output structure:**
```
session_dir/
  frames/
    raw/
      {video_name}/
        frame_0001.png
        frame_0002.png
        ...
```

**Validation:** After extraction, verify frame count matches expected (`fps * duration`). Warn if off by more than 5%.

---

## PHASE 5 — BACKGROUND REMOVAL (`p3_bg_removal.py`)

This is the most critical phase. Implement TWO methods with automatic fallback:

### Method A: rembg (PRIMARY)

```python
def remove_bg_rembg(frames_dir: str, output_dir: str, model: str = "isnet-anime") -> dict:
    """
    Use rembg with session reuse for efficiency.
    
    Model selection:
    - "isnet-anime"     → best for stylized/anime/AI-art characters (DEFAULT)
    - "u2net_human_seg" → best for realistic human characters  
    - "u2net"           → general fallback
    
    Process all PNGs in frames_dir → output_dir as RGBA PNGs.
    Returns: {"processed": int, "failed": int, "model": str}
    """
    from rembg import remove, new_session
    session = new_session(model)
    # Loop with rich progress bar
    # Save as RGBA PNG
```

**Quality check after first 5 frames:**
- Sample 3 output frames
- Check alpha channel: warn if >80% opaque (bg not removed) or >80% transparent (character removed)
- Print the 3 frames as paths and ask user: `"Background removal quality OK? [Y/n/retry-with-u2net]"`

### Method B: numpy/PIL (FALLBACK)

```python
def remove_bg_numpy(frames_dir: str, output_dir: str) -> dict:
    """
    Color-distance based removal. Less accurate but always available.
    
    Algorithm:
    1. Sample background color from 4 corners of first frame
    2. For each pixel: compute euclidean distance from bg_mean in RGB space
    3. Mark as transparent if distance < threshold (55) OR brightness < 20
    4. Apply scipy dilation to smooth character edges
    5. Save as RGBA PNG
    
    IMPORTANT: Print warning to user that this method is less accurate.
    Log "bg_removal_method": "numpy_fallback" in session_config.
    """
```

### Method selector:

```python
def remove_backgrounds(session: dict, preview: bool = True) -> dict:
    """
    Try rembg first. Fall back to numpy if import fails.
    If preview=True, stop after 5 frames and ask user to confirm quality.
    """
```

**Output structure:**
```
session_dir/
  frames/
    nobg/
      {video_name}/
        frame_0001.png  # RGBA, transparent background
        ...
```

---

## PHASE 6 — SEGMENTATION (`p4_segmentation.py`)

### What it does:
Slice the nobg frames into per-animation buckets using the confirmed `animation_map`.

```python
def segment_animations(session: dict) -> dict:
    """
    For each animation in session["animation_map"]:
      1. Get frame_start, frame_end, video name
      2. Copy/symlink the corresponding nobg frames into:
         session_dir/frames/animations/{animation_id}/frame_0001.png ...
      3. Re-number frames starting from 0001
      4. Validate frame count is within expected range from animation_types.json
      5. Warn if too few frames (< typical_frames[0]) or too many (> typical_frames[1] * 2)
    
    Returns: { "walk": ["path1", "path2", ...], "attack_1": [...], ... }
    """
```

**Optional MediaPipe auto-detection** (only if `animation_map` has `"auto_detect": true`):

```python
def mediapipe_classify_frame(frame_path: str, pose) -> str:
    """
    Run MediaPipe pose on a single frame.
    Apply heuristics to classify:
    
    Heuristics (use normalized landmark coordinates):
    - idle:   low movement delta from previous frame, both ankles near same Y
    - walk:   alternating ankle heights, moderate body displacement
    - run:    same as walk but larger displacement delta per frame
    - jump:   both ankles Y < hip Y by significant margin (airborne)
    - fall:   body center Y increasing frame-over-frame
    - attack: wrist landmark extends far from body center (>0.3 normalized units)
    - dash:   extreme horizontal body displacement in 2-3 frames
    - hurt:   sudden large displacement opposite to facing direction
    
    Returns animation_id string or "unknown"
    """
```

---

## PHASE 7 — EXPORT (`p5_export.py`)

### 7A: Spritesheet Packing

```python
def pack_spritesheet(frames: list[str], output_path: str) -> dict:
    """
    Pack frames into a fixed-cell grid PNG.
    
    Layout:
    - All frames same size (use first frame dimensions)
    - Grid: cols = ceil(sqrt(n_frames)), rows = ceil(n_frames / cols)
    - Frames ordered left→right, top→bottom
    - Background: fully transparent (RGBA)
    - Save as PNG
    
    Returns: {
        "path": output_path,
        "frame_w": int, "frame_h": int,
        "cols": int, "rows": int,
        "n_frames": int,
        "sheet_w": int, "sheet_h": int
    }
    """
```

### 7B: Metadata JSON

```python
def write_metadata(session: dict, spritesheets: dict, output_path: str):
    """
    Write metadata.json:
    {
      "character": "mage",
      "generated_at": "ISO timestamp",
      "bg_removal_method": "rembg_isnet-anime",
      "animations": {
        "walk": {
          "sheet": "mage_walk.png",
          "frame_w": 464,
          "frame_h": 688,
          "cols": 4,
          "rows": 3,
          "n_frames": 12,
          "fps": 24,
          "loop": true,
          "pivot": "bottom_center"
        },
        ...
      }
    }
    """
```

### 7C: Unity AnimatorController

```python
def generate_animator_controller(character_name: str, animations: list[str], output_path: str):
    """
    Generate a Unity AnimatorController JSON scaffold.
    
    States to create (only for animations that exist in the session):
    - Idle (default state)
    - Walk, Run
    - Jump, Apex, Fall, Land  
    - Attack1, Attack2, Attack3
    - Dash, Block
    - Hurt, Death
    
    Parameters to declare:
    - Speed: Float
    - IsGrounded: Bool
    - AttackTrigger: Trigger
    - HurtTrigger: Trigger
    - DeathTrigger: Trigger
    
    Key transitions:
    - Idle → Walk:    Speed > 0.1
    - Walk → Run:     Speed > 0.8
    - Walk → Idle:    Speed < 0.1
    - Any → Hurt:     HurtTrigger
    - Any → Death:    DeathTrigger
    - Hurt → Idle:    Exit time 1.0
    - Death:          No exit (stays in death)
    - Jump → Apex:    Exit time 0.8
    - Apex → Fall:    IsGrounded = false (after delay)
    - Fall → Land:    IsGrounded = true
    - Land → Idle:    Exit time 1.0
    
    Output format: JSON (human-readable, can be manually imported or used as reference)
    Also write a C# scaffold: {character_name}AnimatorParams.cs with const strings for all params
    """
```

### 7D: Import Guide

Write `IMPORT_GUIDE.md` from the template with actual file names filled in:

```markdown
# {character_name} Animation Import Guide — Generated by Animation Forge

## Step 1: Import Spritesheets
1. Drag all PNG files from `Sprites/` into your Unity Assets folder
2. For each PNG: set Texture Type → "Sprite (2D and UI)"
3. Set Sprite Mode → "Multiple"
4. Click "Sprite Editor" → Slice → "Grid By Cell Size"
5. Set Cell Size to: {frame_w} x {frame_h} pixels
6. Click Apply

## Step 2: Create Animation Clips
For each animation (e.g. walk):
1. Select all sprites for that animation in the Project window
2. Drag them into the Scene — Unity will prompt to save as .anim
3. Name it: {character_name}_{animation_id}.anim
4. Set Sample Rate to {fps}

Loop settings:
- LOOP these: {looping_animations}
- DO NOT loop: {non_looping_animations}

## Step 3: Set Up Animator
1. Add Animator component to your character GameObject
2. Create AnimatorController asset
3. Reference the JSON scaffold in `Animator/` for states and transitions
4. Or copy params from {character_name}AnimatorParams.cs

## Pivot Points
All sprites use Bottom-Center pivot. If your character floats above the ground,
select all sprites → Sprite Editor → move pivot to exact bottom-center.

## Recommended PPU (Pixels Per Unit)
Based on your sprite dimensions ({frame_w}x{frame_h}):
- Recommended PPU: {recommended_ppu}
- This makes your character approx {approx_height_units} Unity units tall
```

### 7E: Final Output Assembly

```python
def assemble_output_package(session: dict):
    """
    Create final directory structure:
    
    output/{character_name}_animations/
      ├── Sprites/
      │   ├── {char}_idle.png
      │   ├── {char}_walk.png
      │   └── ...
      ├── Animator/
      │   ├── {char}_controller.json
      │   └── {char}AnimatorParams.cs
      ├── metadata.json
      └── IMPORT_GUIDE.md
    
    Then ZIP the whole folder → {character_name}_animations.zip
    Print final summary table with all file sizes.
    """
```

---

## IMPLEMENTATION DETAILS & RULES

### Error Handling
- **Never crash silently.** All exceptions must be caught and shown via `rich.console.Console().print_exception()`
- **Phase failures are recoverable.** If a phase fails, save the session state, print the error, and offer: `"Retry this phase? [Y/n]"`
- **ffmpeg errors:** Capture stderr and show relevant lines only (filter for "Error" and "Invalid")
- **rembg errors:** If model download fails (no internet), automatically try numpy fallback

### Progress Reporting
Every phase must use `rich.progress.Progress` with:
- Task name
- Frame counter (e.g. "Frame 45/145")  
- Elapsed time
- ETA

### File Path Rules
- Use `pathlib.Path` everywhere, never string concatenation for paths
- All output paths must be absolute
- Create output dirs with `Path.mkdir(parents=True, exist_ok=True)`

### Frame Numbering
- Raw extracted frames: `frame_0001.png` (1-indexed, 4-digit zero-padded)
- Animation segment frames: re-numbered from `frame_0001.png`
- Spritesheet cells: 0-indexed in metadata

### Pivot Point Calculation
```python
def get_pivot_bottom_center(frame_w: int, frame_h: int) -> tuple[float, float]:
    """Returns normalized pivot: (0.5, 0.0) — Unity bottom-center convention"""
    return (0.5, 0.0)
```

### Recommended PPU Calculation
```python
def recommended_ppu(frame_h: int, target_unity_height: float = 2.0) -> int:
    """
    For a character that should be ~2 Unity units tall:
    PPU = frame_h / target_unity_height
    Round to nearest power of 2 (32, 64, 128, 256, etc.)
    """
```

---

## TESTING CHECKLIST

After building each phase, verify:

### Phase 0 (Bootstrap)
- [ ] `ffprobe` runs successfully on a test video
- [ ] Session JSON is written with correct fps/duration/frames
- [ ] Vision call works if API key set, skips gracefully if not

### Phase 1 (Questionnaire)  
- [ ] All 5 questions render cleanly in terminal
- [ ] Defaults are pre-filled correctly
- [ ] `animation_map` in session is updated after each confirmation
- [ ] "multiple animations in one video" flow works with manual frame ranges

### Phase 2 (Extraction)
- [ ] Correct number of frames extracted
- [ ] Frames are valid PNGs with correct dimensions
- [ ] Progress bar shows correctly

### Phase 3 (BG Removal)
- [ ] rembg produces RGBA PNGs with transparent backgrounds
- [ ] Quality check triggers after 5 frames
- [ ] numpy fallback produces acceptable (not perfect) results
- [ ] Method used is logged in session_config

### Phase 4 (Segmentation)
- [ ] Animation frames are correctly sliced from frame ranges
- [ ] Re-numbering starts from 0001
- [ ] Frame count warnings trigger correctly

### Phase 5 (Export)
- [ ] Spritesheet is correct grid with transparent background
- [ ] All frames are same dimensions
- [ ] metadata.json has correct frame counts
- [ ] AnimatorController JSON has all declared states
- [ ] C# params file compiles (syntactically valid)
- [ ] IMPORT_GUIDE.md has all placeholder values filled
- [ ] Output ZIP is created and complete

---

## STRETCH GOALS (implement after core works)

### Loop Detection
```python
def detect_loop_candidate(frames: list[str]) -> tuple[int, int]:
    """
    Find frame pair (i, j) where i < j and the frames are most similar.
    Use pixel-level MSE between RGBA arrays.
    Suggest these as loop start/end to user.
    """
```

### Frame Deduplication
```python
def deduplicate_frames(frames: list[str], threshold: float = 0.98) -> list[str]:
    """
    Remove consecutive near-duplicate frames (SSIM > threshold).
    Keeps animation smooth while reducing unnecessary frame count.
    """
```

### Flip/Mirror Export
```python
def export_flipped(spritesheet_path: str) -> str:
    """
    Export a horizontally-flipped version of the spritesheet.
    Naming: {char}_{anim}_flipped.png
    Useful for left-facing character variants.
    """
```

### Batch Mode (no questionnaire)
```bash
# JSON config file instead of interactive questionnaire
python main.py run --config batch_config.json

# batch_config.json format:
{
  "character": "mage",
  "videos": {
    "mage_walk.mp4": { "animation": "walk", "fps": 24 },
    "mage_attack.mp4": { "animation": "attack_1", "fps": 24 }
  }
}
```

---

## DEPENDENCY INSTALL COMMANDS

```bash
# System dependencies (must be installed separately)
# macOS:
brew install ffmpeg

# Ubuntu/Debian:
sudo apt-get install ffmpeg

# Windows:
# Download from https://ffmpeg.org/download.html and add to PATH

# Python dependencies:
pip install -r requirements.txt

# GPU acceleration (optional, requires NVIDIA GPU + CUDA):
pip install rembg[gpu] onnxruntime-gpu
```

---

## KEY DESIGN DECISIONS (DO NOT CHANGE)

| Decision | Rationale |
|---|---|
| rembg `isnet-anime` as default model | Specifically trained for stylized/anime art — matches AI-generated game characters |
| Separate spritesheet per animation | Easier Unity import; Sprite Editor slices by Grid Cell Size with zero config |
| Fixed-cell grid layout (not packed atlas) | Unity Sprite Editor can auto-slice; no TexturePacker needed |
| Bottom-center pivot | Unity 2D standard for ground-based characters |
| session_config.json persistence | Pipeline can be interrupted and resumed at any phase |
| numpy fallback always available | Tool works without internet/GPU, just with lower bg removal quality |
| Per-phase error recovery | Single phase failure doesn't lose all work |

---

## EXAMPLE USAGE (what success looks like)

```bash
$ python main.py run \
    --video /videos/mage_walk.mp4 \
    --video /videos/mage_attack.mp4 \
    --character mage

╔══════════════════════════════════════════╗
║      🎮  ANIMATION FORGE  v1.0           ║
║  Video → Unity Animation Pipeline        ║
╚══════════════════════════════════════════╝

[Phase 0/6] Bootstrap ━━━━━━━━━━ 100% 
  ✓ ffmpeg found (version 6.0)
  ✓ rembg available (isnet-anime)
  ✓ 2 videos loaded: mage_walk.mp4 (6.0s, 145 frames), mage_attack.mp4 (4.2s, 101 frames)
  ✓ Vision analysis: "Stylized female mage character with purple energy aura"

[Phase 1/6] Questionnaire ━━━━━━ 
  Video: mage_walk.mp4
  ? Animation type [walk]: walk ✓
  ? Is this a loop? [Y]: Y ✓
  ? Frame range [0-145]: 0-145 ✓

  Video: mage_attack.mp4
  ? Animation type [attack_1]: attack_1 ✓
  ? Is this a loop? [N]: N ✓

[Phase 2/6] Frame Extraction ━━━━ 100%
  ✓ Extracted 145 frames from mage_walk.mp4
  ✓ Extracted 101 frames from mage_attack.mp4

[Phase 3/6] Background Removal ━━ 100%
  Method: rembg (isnet-anime)
  ✓ 246 frames processed

[Phase 4/6] Segmentation ━━━━━━━━ 100%
  ✓ walk: 145 frames
  ✓ attack_1: 101 frames

[Phase 5/6] Export ━━━━━━━━━━━━━ 100%
  ✓ mage_walk.png       (13x12 grid, 6032x8256px)
  ✓ mage_attack_1.png   (11x10 grid, 5104x6880px)
  ✓ metadata.json
  ✓ mage_controller.json
  ✓ mageAnimatorParams.cs
  ✓ IMPORT_GUIDE.md

╔══════════════════════════════════════════════════════╗
║  ✅ ANIMATION FORGE COMPLETE                          ║
║  Output: ./output/mage_animations/                    ║
║  Package: ./output/mage_animations.zip (12.4 MB)      ║
║  2 animations exported. Ready for Unity import.       ║
╚══════════════════════════════════════════════════════╝
```

---

## START HERE

Build in this exact order:

1. `requirements.txt` + `config/animation_types.json`
2. `utils/session.py`
3. `phases/p2_extract.py` (ffmpeg wrapper — simplest, test the pipeline early)
4. `phases/p3_bg_removal.py` (core value — rembg + numpy fallback)
5. `phases/p5_export.py` (spritesheet packing + metadata)
6. `utils/unity_export.py` (AnimatorController + C# params)
7. `phases/p0_bootstrap.py` (video analysis + Vision)
8. `phases/p1_questionnaire.py` (interactive Q&A)
9. `phases/p4_segmentation.py` (frame slicing)
10. `main.py` (wire everything with click + rich)

**At step 5 you should have a testable end-to-end path:** extract → remove bg → export spritesheet.
