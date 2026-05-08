---
name: cheat-ui-design
description: Best practices for designing cheat/trainer menus and UIs. Use this skill when deciding how to structure tabs, name features, group settings, choose GUI rendering libraries (ImGui, egui, etc.), or design internal vs external overlays for game modification tools. Covers the industry meta for feature naming (aimbot, anti-aim, ESP, resolver, etc.) and layout conventions used across the game hacking community.
---

# Cheat UI Design & Best Practices

## Overview

This skill covers the design conventions, feature naming standards, tab organization patterns, and GUI library choices for cheat and trainer menus in the game hacking scene. It draws from community consensus on UnknownCheats, GuidedHacking, and open-source cheat projects to reflect the current meta as of 2024–2025.

## README Coverage

- `Cheat > Render/Draw`
- `Cheat > Overlay`
- `Cheat > UI Interface`
- `Cheat > Triggerbot & Aimbot`
- `Cheat > WallHack`
- `Cheat > W2S`

---

## Standard Menu Tab Structure

### The Meta Layout (2024–2025)

Most competitive-game cheats (CS2, Valorant, Apex, EFT, etc.) converge on these top-level tabs. Order matters — users expect the most-used features first.

```
[RAGE] [LEGIT] [ANTI-AIM] [VISUALS] [MISC] [SKINS] [CONFIG]
```

For simpler cheats (single-mode, PvE games, trainers):
```
[AIMBOT] [ESP] [MOVEMENT] [MISC] [CONFIG]
```

Sidebar (vertical) navigation is the current dominant pattern over horizontal tabs. Left sidebar + right panel is the standard layout for desktop-style menus.

---

## Tab Definitions & Feature Groups

### 1. RAGE (Ragebot)

High-performance aim assistance. Named "Rage" because it's designed for aggressive "raging" — obvious, care-free play.

```
Rage
├── General
│   ├── Enable Ragebot          [toggle]
│   ├── Target Priority         [combo: Nearest / Lowest HP / Highest Threat]
│   └── Target Selection        [combo: Enemy / All]
├── Hitchance
│   ├── Hitchance %             [slider 0–100]
│   └── Min Damage              [slider]
├── Aiming
│   ├── Silent Aim              [toggle]        ← aim without camera movement
│   ├── Auto Wall               [toggle]        ← shoot through penetrable surfaces
│   ├── Auto Scope              [toggle]
│   ├── Auto Stop               [toggle]        ← stop moving to improve accuracy
│   └── Auto Crouch             [toggle]
├── Hitboxes
│   ├── Head                    [toggle]
│   ├── Neck                    [toggle]
│   ├── Body                    [toggle]
│   ├── Legs                    [toggle]
│   └── Multipoint              [toggle]        ← shoot multiple points on hitbox
└── Resolver
    ├── Enable Resolver         [toggle]        ← counter enemy anti-aim
    ├── Resolver Mode           [combo: Auto / Bruteforce / History]
    └── Override Delta          [toggle]
```

### 2. LEGIT (Legitbot)

Subtle aim assistance designed to look human. Prioritizes not getting overwatch/manual-banned.

```
Legit
├── Aimbot
│   ├── Enable                  [toggle]
│   ├── Aim Key                 [key bind]
│   ├── Aim Bone                [combo: Head / Neck / Nearest / Body]
│   ├── FOV                     [slider 0–180]      ← aim only within this cone
│   ├── Smooth                  [slider 1–20]       ← interpolation speed (higher = more human)
│   ├── RCS (Recoil Control)    [toggle + XY sliders]
│   └── Aim Step                [slider]            ← max degrees per frame
├── Triggerbot
│   ├── Enable                  [toggle]
│   ├── Trigger Key             [key bind]
│   ├── Delay (ms)              [slider 50–500]     ← humanization
│   ├── Shot Delay (ms)         [slider]
│   └── Fire in Scope Only      [toggle]
└── Backtrack
    ├── Enable                  [toggle]
    └── Time (ms)               [slider 0–200]
```

### 3. ANTI-AIM (AA)

Defensive angle manipulation — makes the player's hitbox hard to hit. Core of competitive CS2/Valve hack defense.

```
Anti-Aim
├── General
│   ├── Enable Anti-Aim         [toggle]
│   └── AA Key                  [key bind]      ← hold to activate
├── Pitch (up/down angle)
│   ├── Pitch Mode              [combo: Default / Down / Up / Zero / Custom]
│   └── Pitch Value             [slider -89 to 89]
├── Yaw (left/right spin)
│   ├── Yaw Mode                [combo: Off / Spin / Jitter / Backwards / Static]
│   ├── Yaw Base                [combo: Local View / At Target / Backwards]
│   ├── Yaw Add                 [slider -180 to 180]
│   └── Jitter Range            [slider]
├── Desync / Body
│   ├── Desync Mode             [combo: Off / Freestand / Jitter / Static]
│   ├── Desync Amount           [slider 0–60]   ← exploit LBY update gap
│   └── Fake Angle              [slider]
└── Fake Lag
    ├── Enable Fake Lag         [toggle]        ← choke packets to server
    ├── Fake Lag Amount         [slider 0–14]   ← choked ticks
    └── Adaptive Fake Lag       [toggle]        ← vary based on movement state
```

### 4. VISUALS (ESP)

Extra Sensory Perception — draws game information the engine wouldn't normally show.

```
Visuals
├── Players (Enemy)
│   ├── Enable                  [toggle]
│   ├── Box ESP                 [combo: Off / Corner / Full / 3D]
│   ├── Box Color               [color picker + alpha]
│   ├── Name                    [toggle]
│   ├── Health Bar              [combo: Off / Left / Top / Bottom]
│   ├── Armor Bar               [toggle]
│   ├── Skeleton                [toggle + color]
│   ├── Head Dot                [toggle]
│   ├── Distance                [toggle]
│   ├── Weapon Name             [toggle]
│   ├── Weapon Icon             [toggle]
│   ├── Flags                   [multi-select: Scoped / Defusing / Reloading / Flashed]
│   └── Visibility Check        [toggle]        ← color change if behind wall
├── Players (Team)
│   └── [same sub-options as Enemy]
├── Chams
│   ├── Enable Chams            [toggle]
│   ├── Visible Material        [combo: Flat / Wireframe / Glow / Glass / Metallic]
│   ├── Visible Color           [color picker]
│   ├── Occluded Material       [combo: same]
│   └── Occluded Color          [color picker]
├── World
│   ├── Dropped Weapons         [toggle]
│   ├── Grenades                [toggle]
│   ├── Bomb / Objective        [toggle]
│   ├── Defuse Kit              [toggle]
│   └── Planted Bomb Timer      [toggle]
├── Radar
│   ├── Enable Radar            [toggle]
│   ├── Show Teammates          [toggle]
│   └── Radar Scale             [slider]
└── Other
    ├── FOV Changer             [slider 60–120]
    ├── Viewmodel FOV           [slider]
    ├── Night Mode              [toggle + slider]
    ├── Remove Scope Overlay    [toggle]
    ├── Bullet Tracers          [toggle + color]
    └── Hit Marker              [toggle]
```

### 5. MISC (Miscellaneous / Utility)

Movement features, QoL, and anything that doesn't fit cleanly elsewhere.

```
Misc
├── Movement
│   ├── Bunny Hop               [toggle]        ← auto-jump at landing frame
│   ├── Auto-Strafe             [toggle]        ← optimal air strafing
│   ├── Slide Lock              [toggle]
│   └── Fast Stop               [toggle]
├── Inventory / Cosmetics
│   ├── Skin Changer            [toggle + item picker]
│   ├── Knife Changer           [combo]
│   └── Glove Changer           [combo]
├── Utility
│   ├── Spectator List          [toggle]        ← show who's watching
│   ├── Clantag Changer         [text input + scroll]
│   ├── Name Changer            [text input]
│   ├── Third Person            [toggle + key bind]
│   └── Unlock All              [toggle]
└── Menu
    ├── Menu Key                [key bind]
    └── Panic Key               [key bind]      ← instantly disable all features
```

### 6. SKINS

Skin/cosmetic features, often their own tab in major cheats:

```
Skins
├── Knife Model                 [combo: list of knife models]
├── Glove Model                 [combo]
├── Skin Picker                 [per-weapon, searchable list]
│   ├── Weapon Select           [combo]
│   ├── Skin                    [combo / search]
│   ├── Wear                    [slider 0.0–1.0]
│   ├── Seed                    [int input]
│   └── StatTrak Count          [int input]
└── Apply                       [button]
```

### 7. CONFIG

Profile management. Should always be in its own tab, never buried in Misc.

```
Config
├── Profile Name                [text input]
├── Save Config                 [button]
├── Load Config                 [button]
├── Delete Config               [button]
├── Import from Clipboard       [button]
├── Export to Clipboard         [button]
└── Reset to Default            [button]
```

---

## Feature Naming Standards

Industry-standard terms. These are the names users search for and expect. Don't rename them.

| Internal Concept              | Standard Name(s)                    | Notes                                      |
|-------------------------------|-------------------------------------|--------------------------------------------|
| Aim assistance (obvious)      | **Ragebot**, Rage                   | Never "aim hack" or "aimassist" in UI      |
| Aim assistance (subtle)       | **Legitbot**, Legit                 |                                            |
| Auto-fire on crosshair        | **Triggerbot**                      |                                            |
| Camera-independent aim        | **Silent Aim**                      | Also "Backtrack Aim"                       |
| Angle manipulation            | **Anti-Aim** (AA)                   |                                            |
| Anti-AA counter               | **Resolver**                        |                                            |
| Packet choking                | **Fake Lag** (FL)                   | Also "Lag Exploit"                         |
| LBY/view angle split          | **Desync**                          |                                            |
| Angle delta value             | **Delta**                           |                                            |
| Box around enemy              | **Box ESP**                         |                                            |
| See-through walls             | **Wallhack**, **Chams**             | Chams = colored model materials            |
| Enemy position on minimap     | **Radar Hack**                      |                                            |
| Hitbox visualization          | **Skeleton ESP**                    |                                            |
| Shoot through walls           | **Auto Wall**                       |                                            |
| Probability-based firing      | **Hitchance**                       |                                            |
| Auto-jump timing              | **Bunny Hop** (Bhop)                |                                            |
| Air movement optimization     | **Auto-Strafe**                     |                                            |
| No weapon spread              | **No Spread**                       |                                            |
| No weapon recoil              | **No Recoil**, **RCS**              | RCS = Recoil Control System                |
| Aim cone                      | **FOV** (on aimbot settings)        | Field of View — not to be confused w/ view FOV |
| Human-feel interpolation      | **Smooth**, **Smoothness**          |                                            |
| Shoot multiple hitbox points  | **Multipoint**                      |                                            |
| Minimum damage required       | **Min Damage**                      |                                            |
| Shoot from past positions     | **Backtrack**                       |                                            |

---

## Layout & UX Best Practices

### Navigation

- **Sidebar (vertical tabs)** is the dominant pattern. Horizontal tab bars feel outdated.
- Use icons alongside tab labels for quick scanning (FontAwesome works well with ImGui).
- Active tab should have a distinct accent color, not just bold text.

### Within a Tab

```
┌─────────────────────────────────────────────────┐
│  [Tab Name]                                     │
│  ─────────────────────────────────────────────  │
│  ┌─ Group Header ──────────────────────────┐   │
│  │  [✓] Feature Enable       [COLOR ████]  │   │
│  │  Sub-option slider  ══════════════ 75%  │   │
│  │  Dropdown           [Option A        ▼] │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
│  ┌─ Another Group ─────────────────────────┐   │
│  │  ...                                    │   │
│  └─────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

- Group related settings with `BeginChild` / `GroupBox` boundaries.
- Disable (grey out) sub-options when the parent toggle is off — don't hide them.
- Use **collapsing headers** for advanced sub-options (e.g., resolver settings, multipoint config).
- **Color pickers** should be inline, not in popups, for commonly-changed colors.
- **Key binders** must show the currently bound key at all times.

### Hierarchy Rule

```
Tab → Group → Feature Toggle → Sub-settings (indented or greyed)
```

Never put a sub-setting at the same visual level as a top-level toggle.

### Sizing

- Minimum menu width: ~500px (internal), ~600px (external).
- Line height: 22–26px. Cramping is a sign of amateur work.
- Group padding: 8px minimum inside, 4px between groups.
- Font: 13–14px. Anything smaller is unreadable mid-game.

### Color Scheme Meta

The dominant aesthetic in high-quality cheats (2024–2025):

```
Background:     #0D0D0F (near-black, slightly warm)
Panel:          #161618
Group border:   #252528
Accent:         Single accent color — deep blue (#4A90D9), red (#C0392B), or purple (#8E44AD)
Text primary:   #E8E8E8
Text secondary: #888888
Enabled toggle: Accent color
Disabled:       #3A3A3D
```

Avoid: rainbow RGB gradients (screams newbie), pure #FFFFFF backgrounds, Windows 98 gray.

---

## GUI Library Choices

### C++ Libraries

#### Dear ImGui (Immediate Mode GUI)
**The industry standard.** Every serious cheat uses it or forks from it.

- **Repo**: https://github.com/ocornut/imgui
- **Current stable**: 1.91.x (docking branch recommended for multi-panel layouts)
- **Mode**: Immediate mode (no retained state — call every frame)
- **Best for**: Internal cheats (hooked into game's D3D/OGL/Vulkan), external overlays
- **Rendering backends**: DirectX 9/10/11/12, OpenGL2/3, Vulkan, SDL, SFML, Metal
- **License**: MIT
- **Pros**: Mature, huge ecosystem, well-documented, massive example base
- **Cons**: Default styling is ugly — requires custom theme work, no layout engine
- **Meta usage**: >90% of public cheats use this
- **Breaking changes to watch**: `ImGui::GetIO().Fonts` API changed in 1.90+;
  `ImGuiKey_*` unified key enum replaced legacy `ImGuiKey_COUNT` in 1.87+

**Recommended forks / styled versions:**
- `ImGui-Standalone` (adamhlt) — standalone external window setup
- `RequestFX/ImGUI-Advanced-Cheat-Menu` — pre-styled cheat-specific template
- `ImGui_docking` branch — multi-window docking support (use this for complex menus)

**Styling in code:**
```cpp
// Centralize all style settings — never scatter SetStyleColor() calls
void ApplyTheme() {
    ImGuiStyle& style = ImGui::GetStyle();
    style.WindowRounding    = 8.0f;
    style.FrameRounding     = 4.0f;
    style.ScrollbarRounding = 4.0f;
    style.GrabRounding      = 4.0f;
    style.WindowBorderSize  = 1.0f;
    style.FramePadding      = ImVec2(8, 4);
    style.ItemSpacing       = ImVec2(8, 6);

    ImVec4* colors = style.Colors;
    colors[ImGuiCol_WindowBg]       = ImVec4(0.05f, 0.05f, 0.06f, 1.0f);
    colors[ImGuiCol_FrameBg]        = ImVec4(0.09f, 0.09f, 0.10f, 1.0f);
    colors[ImGuiCol_Header]         = ImVec4(0.29f, 0.56f, 0.85f, 0.35f);
    colors[ImGuiCol_HeaderHovered]  = ImVec4(0.29f, 0.56f, 0.85f, 0.55f);
    colors[ImGuiCol_CheckMark]      = ImVec4(0.29f, 0.56f, 0.85f, 1.0f);
    // ...
}
```

---

#### GDI / GDI+
- **Best for**: Simple external overlays (no DX hook), Windows-only
- **Pros**: No dependencies, always available
- **Cons**: No hardware acceleration, tearing issues, slow for complex UIs, easily detected
- **Verdict**: OK for basic boxes/text drawing. Don't build a full menu in GDI.

---

#### Direct2D / DirectWrite
- **Best for**: External overlays wanting hardware acceleration without DX hook
- **Pros**: Smooth rendering, anti-aliased text, good for overlay-only use
- **Cons**: More complex setup than GDI, no widget system (you build everything)
- **Verdict**: Good for ESP rendering layer in an external cheat. Pair with a config file UI instead of in-game menu.

---

#### Qt (C++)
- **Best for**: External "loader" / configuration applications (not in-game menus)
- **Pros**: Full-featured, professional-looking, great for standalone config tools
- **Cons**: Heavy, licensing issues (GPL/commercial), overkill for in-game
- **Verdict**: Use for the external config tool / injector GUI, not the in-game overlay.

---

#### wxWidgets / Dear ImGui (external window)
- Same verdict as Qt for external config panels.

---

### Rust Libraries

#### imgui-rs
- **Repo**: https://github.com/imgui-rs/imgui-rs
- **What it is**: Safe Rust bindings over Dear ImGui C++ via FFI
- **Best for**: Rust cheats that want the full ImGui ecosystem
- **Backends**: Works with dx11/dx12 via `imgui-dx11`/`imgui-dx12` crates
- **Pros**: Access to full ImGui feature set, active maintenance
- **Cons**: FFI overhead, C++ interop complexity, binding lag behind upstream

---

#### egui
- **Repo**: https://github.com/emilk/egui
- **What it is**: Pure Rust immediate-mode GUI — no C++ dependency
- **Best for**: External Rust cheats, config panels, loader UIs
- **Rendering backends**: `egui-d3d11` (see `gmh5225/egui-d3d11` in the README), `egui_winit_vulkano`, `wgpu`
- **Pros**: 100% safe Rust, no FFI, very active development, good documentation
- **Cons**: Different API feel from ImGui, less cheat-scene tooling around it
- **Meta status**: Growing — `egui-d3d11` specifically designed for D3D11 hook injection

**egui-d3d11 for internal cheat:**
```rust
// Hook into Present, get device, init egui renderer
// See: https://github.com/gmh5225/egui-d3d11
```

---

#### hudhook
- **Repo**: https://github.com/veeenu/hudhook
- **What it is**: Rust library for injecting ImGui-based overlays via rendering hook
- **Best for**: Internal Rust cheats — handles the D3D/OGL hook boilerplate
- **Hooks**: DirectX 11, DirectX 12, OpenGL3
- **Pros**: Abstracts away all the hook + ImGui init complexity, Rust-native
- **Verdict**: The cleanest choice for a Rust internal cheat with ImGui UI

---

#### Specter (iced-based)
- **iced**: https://github.com/iced-rs/iced
- Not commonly used in cheats yet — more of a general Rust GUI. No D3D injection story.
- **Verdict**: Skip for in-game overlays. Fine for standalone tool UIs.

---

## Internal vs External Overlay Architecture

### Internal Cheat (DLL injected into game)

```
Approach: Hook game's rendering API (Present/SwapChain)

C++:
  D3D11: Hook IDXGISwapChain::Present
  D3D12: Hook IDXGISwapChain3::Present
  Vulkan: Hook vkQueuePresentKHR
  OGL: Hook wglSwapBuffers / glXSwapBuffers

Rust:
  hudhook (handles all of the above automatically)
  imgui-rs + manual hook (more control)
  egui-d3d11 + manual hook

Pros:
  - Render within game's GPU context (no overlay window)
  - Access to game objects directly in memory
  - Harder to screenshot-detect via DWM
  - Lower latency

Cons:
  - Crash risk (hook must be stable)
  - Detected if AC scans for hooked vtables
  - Code runs in game process — AC has full visibility
```

### External Cheat (separate process)

```
Approach: Create transparent, click-through overlay window

C++:
  Win32: WS_EX_LAYERED | WS_EX_TRANSPARENT | WS_EX_TOPMOST
  Render: D3D11 swapchain on HWND, then ImGui on top
  Alternative: DWM-based overlay (see gmh5225/DWM-DwmDraw)

Rust:
  egui + winit transparent window
  imgui-rs + windows-rs for HWND creation
  Raw D2D1 rendering

Pros:
  - Separate process — harder for AC to tamper with
  - Clean separation of cheat logic from game
  - Won't crash the game on exception

Cons:
  - Window overlay detectable (GetWindowLong / DWM composition checks)
  - Mouse/keyboard passthrough needs SetWindowPos juggling
  - Screenshot capture sees overlay in some configs

Key technique — click-through external window (C++):
  SetWindowLong(hwnd, GWL_EXSTYLE,
      GetWindowLong(hwnd, GWL_EXSTYLE) | WS_EX_TRANSPARENT | WS_EX_LAYERED);
  SetLayeredWindowAttributes(hwnd, RGB(0,0,0), 0, LWA_COLORKEY);
```

---

## Menu Keybinding System Design

Every feature that can be toggled mid-game needs a keybind. Standard implementation:

```cpp
enum BindType { HOLD, TOGGLE, ALWAYS };

struct KeyBind {
    int      key;       // VK_* or scan code
    BindType type;
    bool     active;    // current state
};

// Poll in main loop, not in render:
void UpdateBinds() {
    for (auto& [name, bind] : g_binds) {
        bool pressed = GetAsyncKeyState(bind.key) & 0x8000;
        if (bind.type == HOLD)   bind.active = pressed;
        if (bind.type == TOGGLE && pressed && !bind.prev) bind.active ^= 1;
        bind.prev = pressed;
    }
}
```

**The panic key** (`INSERT` is traditional, but `DELETE` or `END` are also common) should disable everything instantly and optionally hide the menu.

---

## Config System Design

Configs should be JSON (human-readable, easy to share). Don't use binary formats — users share configs.

```cpp
// nlohmann/json is the meta choice for C++ config serialization
#include <nlohmann/json.hpp>

void SaveConfig(const std::string& path) {
    nlohmann::json j;
    j["aimbot"]["enabled"]  = g_cfg.aimbot.enabled;
    j["aimbot"]["fov"]      = g_cfg.aimbot.fov;
    j["aimbot"]["smooth"]   = g_cfg.aimbot.smooth;
    // ...
    std::ofstream(path) << j.dump(4);
}
```

**Config storage location**: `%APPDATA%\<cheat_name>\configs\` — never next to the game executable.

---

## Anti-Screenshot / Menu Visibility

For cheats that need menu invisibility from game screenshots:

- Use `SetWindowDisplayAffinity(hwnd, WDA_EXCLUDEFROMCAPTURE)` on Windows 10 2004+.
- See `Cheat > Anti Screenshot` in README for more techniques.
- The menu window (or DWM overlay) should be excluded from capture separately from the ESP overlay.

---

## Common Mistakes to Avoid

| Mistake | Fix |
|---|---|
| Doing heavy logic (AOB scans, memory reads) in the render loop | Move to separate thread; render loop only reads cached values |
| Using unique IDs from displayed text alone | Use `##suffix` in ImGui to decouple ID from label |
| One giant settings tab | Split into focused tabs; 15+ features per tab = bad UX |
| Showing ALL sub-options even when parent is disabled | `ImGui::BeginDisabled(!feature_enabled)` |
| Hard-coded hotkeys (no rebinding) | Always use a keybind struct + UI to change it |
| Configs in game directory | Use `%APPDATA%` |
| RGB rainbow accent color | One accent color — makes it look professional |
| Missing "panic key" | Every cheat needs a kill-switch |
| RWX memory for render data | Never allocate render buffers as PAGE_EXECUTE_READWRITE |
| Calling ImGui from multiple threads | ImGui is NOT thread-safe; all calls must be from the render thread |
| Holding game mutex during render | Deadlock risk; copy data into a render-safe struct each frame |

---

## Render / Logic Thread Model

This is the single most important architectural decision for a clean cheat:

```cpp
// THE PATTERN: three threads, two lock-free buffers
//
//  [Logic Thread]  →  writes to g_cache (atomic/mutex protected)
//  [Render Thread] →  reads from g_cache, calls ImGui each frame
//  [Input Thread]  →  polls keybinds, writes to g_config

// Shared render cache (POD only, no pointers into game memory)
struct PlayerRenderInfo {
    float    screen_x, screen_y;  // pre-computed W2S
    float    health;              // 0.0–1.0
    float    dist;
    char     name[32];
    bool     visible;
    bool     valid;
};

struct RenderCache {
    std::array<PlayerRenderInfo, 64> players;
    int     player_count;
    float   local_health;
    // ... other ESP data ...
};

// Double-buffer: logic writes to back, render reads from front
std::atomic<int>  g_front{0};
RenderCache       g_cache[2];

// Logic thread:
void LogicThread() {
    while (g_running) {
        int back = 1 - g_front.load(std::memory_order_relaxed);
        UpdateRenderCache(g_cache[back]);   // read game memory here
        g_front.store(back, std::memory_order_release);
        std::this_thread::sleep_for(std::chrono::milliseconds(1));
    }
}

// Render thread (called from hooked Present):
void OnPresent() {
    int front = g_front.load(std::memory_order_acquire);
    const auto& cache = g_cache[front];     // read-only, no locks

    ImGui_ImplDX11_NewFrame();
    ImGui_ImplWin32_NewFrame();
    ImGui::NewFrame();

    DrawESP(cache);      // draws boxes, names etc. from cached data
    if (g_menu_open) DrawMenu();

    ImGui::Render();
    ImGui_ImplDX11_RenderDrawData(ImGui::GetDrawData());
}
```

**Why this matters**: reading game memory inside Present/SwapChain hook
causes frame stutter and risks crashing inside driver code. Always decouple.
---

## Data Source

**Important**: This skill provides design guidance and community conventions. For implementation references and curated repos, use the following sources:

### 1. Project Overview & Resource Index

Fetch the main README for the full curated list of repositories, tools, and descriptions:

```
https://raw.githubusercontent.com/gmh5225/awesome-game-security/refs/heads/main/README.md
```

Relevant sections in the README:
- `Cheat > Render/Draw` — rendering libraries and overlay implementations
- `Cheat > UI Interface` — standalone ImGui setups
- `Cheat > Overlay` — DWM and external overlay techniques
- `Cheat > Anti Screenshot` — screenshot evasion

### 2. Repository Code Details (Archive)

For detailed repository information (file structure, source code, implementation details), the project maintains a local archive. If a repository has been archived, **always prefer fetching from the archive** over cloning or browsing GitHub directly.

**Archive URL format:**
```
https://raw.githubusercontent.com/gmh5225/awesome-game-security/refs/heads/main/archive/{owner}/{repo}.txt
```

**Examples:**
```
https://raw.githubusercontent.com/gmh5225/awesome-game-security/refs/heads/main/archive/adamhlt/ImGui-Standalone.txt
https://raw.githubusercontent.com/gmh5225/awesome-game-security/refs/heads/main/archive/RequestFX/ImGUI-Advanced-Cheat-Menu.txt
https://raw.githubusercontent.com/gmh5225/awesome-game-security/refs/heads/main/archive/veeenu/hudhook.txt
https://raw.githubusercontent.com/gmh5225/awesome-game-security/refs/heads/main/archive/gmh5225/egui-d3d11.txt
```

**How to use:**
1. Identify the GitHub repository the user is asking about (owner and repo name from the URL).
2. Construct the archive URL: replace `{owner}` with the GitHub username/org and `{repo}` with the repository name (no `.git` suffix).
3. Fetch the archive file — it contains a full code snapshot with file trees and source code generated by `code2prompt`.
4. If the fetch returns a 404, the repository has not been archived yet; fall back to the README or direct GitHub browsing.

### 3. Repository Descriptions

For a concise English summary of what a repository does, the project maintains auto-generated description files.

**Description URL format:**
```
https://raw.githubusercontent.com/gmh5225/awesome-game-security/refs/heads/main/description/{owner}/{repo}/description_en.txt
```

**How to use:**
1. Identify the GitHub repository the user is asking about (owner and repo name from the URL).
2. Construct the description URL: replace `{owner}` with the GitHub username/org and `{repo}` with the repository name.
3. Fetch the description file — it contains a short, human-readable summary of the repository's purpose and contents.
4. If the fetch returns a 404, the description has not been generated yet; fall back to the README entry or the archive.

**Priority order when answering questions about a specific repository:**
1. Description (quick summary) — fetch first for concise context
2. Archive (full code snapshot) — fetch when deeper implementation details are needed
3. README entry — fallback when neither description nor archive is available
