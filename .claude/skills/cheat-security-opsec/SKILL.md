---
name: cheat-security-opsec
description: Comprehensive guide for cheat developer and user security. Covers user OPSEC (avoiding bans, HWID hygiene, VM isolation), cheat self-protection (anti-crack, anti-RE, anti-dump), loader/distribution architecture (licensing, auth servers, per-user builds, watermarking), and secure development practices for game modification software.
---

# Cheat Security & OPSEC

## Overview

This skill covers the full security lifecycle of cheat software from both sides: protecting the **user** from detection/bans and protecting the **developer's** work from crackers, leakers, and reverse engineers. It also covers the distribution infrastructure (loaders, licensing, auth) that connects the two.

## README Coverage

- `Cheat > Anti Signature Scanning`
- `Cheat > Hide`
- `Cheat > Anti Forensics`
- `Cheat > HWID`
- `Cheat > Spoof Stack`
- `Anti Cheat > Binary Packer`
- `Anti Cheat > Obfuscation Engine`
- `Anti Cheat > Encrypt Variable`
- `Anti Cheat > Lazy Importer`
- `Anti Cheat > Compile Time`

---

## PART 1: KEEPING YOUR USERS UNDETECTED

This section is for **cheat developers**. The goal: architect your cheat so that users running it don't get banned. Each subsection maps to a detection vector and how to defeat it at the code/design level.

### Detection Vector Map (Developer Perspective)

```
Detection vector         What AC does                 What your cheat must do
─────────────────────────────────────────────────────────────────────────────
Signature scan           Byte pattern match in memory  Polymorphic code, encrypted
                         Entropy / packed-binary scan  sections, import obfuscation
                         Module hash verification      No static signatures

Image load callbacks     PsSetLoadImageNotifyRoutine   Manual map (no LdrLoadDll)
                         catches any PE loaded via     Shellcode (no PE header)
                         standard loader path          TLS-callback injection

Handle enumeration       Who has VM_READ on game?      Use kernel driver or syscall
                         NtQuerySystemInformation      directly; avoid OpenProcess
                         handle table walk             from your process

Thread detection         CreateRemoteThread source     Thread hijacking, APC
                         Start address outside module  injection, thread pool abuse
                         TEB.ThreadLocalStoragePointer Stack spoofing

Stack walk               RtlWalkFrameChain             Fake return addresses
                         .pdata UNWIND_INFO check      synthetic stack frames
                         Non-module RIP in stack       NtSetInformationThread

Screenshot               PrintWindow / DXGI dup        WDA_EXCLUDEFROMCAPTURE
                         ML classifier on captures     Kernel-level DWM bypass
                         FACEIT: periodic + random     Don't render in captures

Behavioral / server-side Aim histogram, HS rate         Build humanization into
                         Movement correlation           every aim feature; add
                         Fire rate, reaction time       noise and variance APIs

Driver trust             HVCI, PiDDBCache, blocklist   EFI loader, signed driver,
                         MmUnloadedDrivers             or no driver at all
```

---

### Safe Injection — What NOT to Do

```
NEVER use these injection methods in a cheat you ship to users:

  ✗ CreateRemoteThread        → trivially detected; logged by AC callbacks
  ✗ SetWindowsHookEx          → creates a registered hook visible to AC
  ✗ LoadLibrary (remote)      → triggers LdrLoadDll → PsSetLoadImageNotify
  ✗ WriteProcessMemory → RWX  → RWX private memory = instant red flag
  ✗ NtCreateThreadEx          → same as CRT, easily monitored
```

### Safe Injection — What to Use

```
Ranked by stealth (best first):

1. SHELLCODE via thread pool callback (no PE, no module, no LDR entry)
     NtAllocateVirtualMemory (RW) → copy shellcode → NtProtect (RX)
     TpAllocWork + TpPostWork → execute via legitimate thread pool thread
     Result: call stack looks like legitimate Windows work item

2. APC injection into game's own threads (early bird or existing thread)
     NtQueueApcThread on a suspended or alertable game thread
     No new threads created; execution context is the game itself
     Stack trace starts from the game's thread → looks normal

3. Manual mapper (PE without LoadLibrary)
     Parse PE, allocate sections, process relocs, resolve imports manually
     Never calls LdrLoadDll → never triggers image load notify callback
     Erase MZ/PE header after mapping → unrecognizable in memory dumps

4. Thread hijacking
     Suspend existing game thread → modify RIP → resume
     No new threads, no new handles; uses game's own thread
     Must restore cleanly; any crash = game crash

5. Reflective DLL injection
     DLL contains a custom reflective loader stub
     Loads itself into memory without Windows loader involvement
     No IAT-visible imports until runtime resolution
```

### Module Hiding (Post-Injection)

After injection, your code must disappear from every enumeration path:

```cpp
// 1. Unlink from PEB loader lists (InLoadOrderModuleList,
//    InMemoryOrderModuleList, InInitializationOrderModuleList)
void UnlinkFromPEB(HMODULE hModule) {
    auto peb = reinterpret_cast<PPEB>(__readgsqword(0x60));
    auto ldr = peb->Ldr;

    for (auto entry = ldr->InLoadOrderModuleList.Flink;
         entry != &ldr->InLoadOrderModuleList;
         entry = entry->Flink) {
        auto mod = CONTAINING_RECORD(entry, LDR_DATA_TABLE_ENTRY,
                                     InLoadOrderLinks);
        if (mod->DllBase == hModule) {
            // Unlink from all three lists
            RemoveEntryList(&mod->InLoadOrderLinks);
            RemoveEntryList(&mod->InMemoryOrderLinks);
            RemoveEntryList(&mod->InInitializationOrderLinks);
            break;
        }
    }
}

// 2. Erase PE header from memory (first page → zero or random bytes)
void ErasePEHeader(void* base) {
    DWORD old;
    VirtualProtect(base, 0x1000, PAGE_READWRITE, &old);
    SecureZeroMemory(base, 0x1000);
    VirtualProtect(base, 0x1000, PAGE_READONLY, &old);
}

// 3. Change memory region type from MEM_IMAGE to MEM_PRIVATE
//    (requires kernel driver — NtSetInformationVirtualMemory in Win10 1803+)
//    Without driver: scattered allocations that never look like an image
```

### Stack Spoofing (Critical for Thread Detection)

Every thread your cheat creates must have a believable call stack:

```cpp
// The problem: your thread starts at your_function() inside an
// unregistered memory region. AC walks the stack and finds:
//   RIP = 0x7FF600001234  ← your cheat code (not in any module)
//   → instant flag

// Solution: synthetic stack frame that looks like a legitimate callback

// Pattern 1: Thread pool fiber — start address is a thread pool function,
//            your code runs as a "work item" from it.
//            Stack looks like: ntdll!TppWorkerpProc → your callback

// Pattern 2: JMP gadget spoof
//   Allocate a "trampoline" inside a legitimate module's code cave
//   Thread entry = trampoline address (looks like it's in ntdll)
//   Trampoline JMPs to your real function
//   When AC walks stack, it sees: ntdll!xxx → your work

// Pattern 3: NtSetInformationThread HideFromDebugger
//   Hides thread from debugger attachment
//   Still visible to kernel callbacks — not sufficient alone
NtSetInformationThread(GetCurrentThread(),
    (THREADINFOCLASS)17, // ThreadHideFromDebugger
    nullptr, 0);

// Full stack spoof requires: valid .pdata UNWIND_INFO entries for
// every frame in the fake stack. Without this, RtlVirtualUnwind
// fails → AC detects invalid unwind chain.
// Reference: SpoofCallStack, Vulcan (see README: Cheat > Spoof Stack)
```

### Handle Concealment

Your cheat must not appear in handle enumeration:

```cpp
// Problem: any call to OpenProcess(PROCESS_VM_READ, game_pid) creates
// a handle that AC can find by walking all system handles.

// Solutions (ranked):

// 1. Kernel driver — no user-mode handle needed at all.
//    Read memory via MmCopyMemory from kernel. No handle, no trace.

// 2. Duplicate handle from an existing trusted process
//    (e.g., the game's own launcher which legitimately has a handle)
//    DuplicateHandle → you share the handle object, not a new one
//    Still visible but attributed to the trusted source process

// 3. NtQuerySystemInformation handle hijack
//    Find the game's CSRSS or services handle to itself
//    Duplicate from that process → looks like system process access

// 4. If you must use OpenProcess: do it via direct syscall
//    (not through ntdll — avoids usermode hook logging)
//    And close the handle as soon as you finish each RPM batch
//    Short-lived handles are harder to catch in periodic scans
```

### Screenshot Evasion (Built Into Cheat)

AC screenshots capture your overlay. This must be handled at the **render design** level:

```cpp
// External window approach (correct):
// The overlay window itself should be excluded from capture.
// This is a property set on the HWND, not on the screen region.

SetWindowDisplayAffinity(overlay_hwnd, WDA_EXCLUDEFROMCAPTURE);
// WDA_EXCLUDEFROMCAPTURE (0x11) — introduced Win10 2004 (build 19041)
// Window appears black or absent in any screenshot/capture API:
//   BitBlt, PrintWindow, DXGI OutputDuplication, DwmGetWindowAttribute

// Internal hook approach:
// You render inside the game's own device context.
// AC screenshots via BitBlt/PrintWindow capture the game window —
// they WILL see your overlay if it's rendered to the game's swapchain.
//
// Solution: hook the screenshot API and hide your draws before capture.
// Or: use a separate draw call that skips rendering during capture frames.
// Detection signal: if game screenshots suddenly have lower draw call count.

// Kernel-level: DWM composition bypass
// See README: Cheat > Anti Screenshot and Cheat > Render/Draw (DWM entries)
// DWM-based overlays can be excluded at composition level
// Reference: gmh5225/DWM-DwmDraw, Yukin02/Dwm-Overlay
```

### Behavioral Humanization APIs

Your aimbot's output is a detection vector. Build these into the feature itself:

```cpp
// Every aim assistance feature must expose:
struct HumanizationConfig {
    float smooth_factor;      // interpolation speed (1=instant, 20=very slow)
    float max_degrees_ps;     // max degrees/second the aim moves
    int   reaction_ms_min;    // minimum reaction time before aiming
    int   reaction_ms_max;    // maximum (adds random delay per target)
    float fov_degrees;        // only aim at targets within this cone
    float jitter_px;          // add sub-pixel noise to every aim step
    bool  stop_on_shoot;      // stop aiming mid-burst (human behavior)
};

// Aim step with built-in humanization:
Vector2 HumanizeAimStep(Vector2 current, Vector2 target,
                         const HumanizationConfig& cfg, float dt) {
    Vector2 delta = target - current;
    float dist = delta.Length();

    // Cap movement by degrees/second limit
    float max_step = cfg.max_degrees_ps * dt;
    if (dist > max_step) delta = delta.Normalized() * max_step;

    // Smooth interpolation (lerp toward target)
    delta *= (1.0f / cfg.smooth_factor);

    // Add Gaussian noise for sub-pixel jitter
    delta.x += GaussianNoise(0.0f, cfg.jitter_px);
    delta.y += GaussianNoise(0.0f, cfg.jitter_px);

    return current + delta;
}

// Trigger delay — never fire instantly on crosshair overlap:
void TriggerFire(const HumanizationConfig& cfg) {
    int delay = cfg.reaction_ms_min +
        rand() % (cfg.reaction_ms_max - cfg.reaction_ms_min);
    std::this_thread::sleep_for(std::chrono::milliseconds(delay));
    SendInput(/* fire */);
}
```

### Safe Unload

Unloading incorrectly is as dangerous as detection during operation:

```cpp
// Unload checklist (must happen in this order):
// 1. Stop all cheat threads gracefully (set flag, join threads)
// 2. Remove all hooks (restore original bytes before freeing memory)
// 3. Clear all allocated memory (SecureZeroMemory before VirtualFree)
// 4. Restore PEB LDR entries (if you unlinked yourself, re-link or
//    remove cleanly — dangling pointers crash the process)
// 5. Free the module (FreeLibraryAndExitThread from a different thread)
//    Never call FreeLibrary on yourself from DllMain

// Hook removal BEFORE memory free is critical:
// If AC calls the hooked function after you free your trampoline memory
// → access violation → crash → logged as suspicious crash

// Panic key implementation (user-triggered emergency unload):
void PanicUnload() {
    // Disable all features atomically first
    g_features_enabled.store(false, std::memory_order_seq_cst);
    // Wait one frame for render thread to finish using feature state
    Sleep(32);
    // Then begin full unload sequence
    BeginCleanUnload();
}
```

---

## PART 2: CHEAT SELF-PROTECTION (ANTI-CRACK)

### Threat Model for Cheat Developers

```
Who cracks your cheat:
  1. Competitors — rival cheat developers steal your features
  2. Resellers — crack and resell at lower price
  3. Script kiddies — crack for clout on UC/discord
  4. AC researchers — reverse to build signatures

Your goal: make the cost of cracking exceed the value gained.
Perfect protection doesn't exist — you're buying TIME.
```

### Binary Protection Layers

```
Layer 1: Compilation hardening
  ├── Strip symbols (MSVC: /DEBUG:NONE, GCC: -s)
  ├── Disable RTTI (/GR-)
  ├── Enable optimizations (/O2 or /Ox)
  ├── Use LTO/LTCG for cross-TU inlining
  ├── Randomize section names
  └── Use constexpr string encryption (compile-time)

Layer 2: Import obfuscation
  ├── Lazy importer: resolve API calls at runtime by hash
  │   xor_str("kernel32.dll") → GetModuleHandle → GetProcAddress
  ├── Syscall stubs: bypass user-mode hooks entirely
  │   Direct syscall via manual assembly (no ntdll dependency)
  ├── Dynamic IAT: build import table at runtime
  └── Indirect calls: function pointers resolved at init

Layer 3: Code protection (pick one or combine)
  ├── VMProtect — virtualizes selected functions into custom bytecode
  │   Use [VMProtect SDK] markers: VMProtectBegin / VMProtectEnd
  │   Virtualize: auth check, license validation, core features
  │   Mutate: everything else (lower overhead than VM)
  ├── Themida — similar to VMP, different VM architecture
  │   Winlicense variant includes licensing system
  └── Custom VM — hardest to crack but massive dev effort
      Roll your own bytecode interpreter for critical paths

Layer 4: Anti-tamper
  ├── CRC checks on own code sections at runtime
  ├── Debug register monitoring (DR0-DR7)
  ├── Timing checks (RDTSC between code blocks)
  ├── Parent process validation (ensure launched from loader)
  ├── INT 2D / INT 3 anti-debug tricks
  ├── NtQueryInformationProcess(ProcessDebugPort)
  ├── PEB.BeingDebugged + PEB.NtGlobalFlag checks
  ├── Heap flags verification
  └── Thread hiding: NtSetInformationThread(HideFromDebugger)

Layer 5: Anti-dump
  ├── Erase PE header from memory after loading
  ├── Scatter code across multiple allocations (no contiguous image)
  ├── Re-encrypt sections after use (decrypt on demand)
  ├── Guard pages on code sections (re-encrypt on access violation)
  └── VirtualProtect to PAGE_NOACCESS on unused code regions
```

### String Protection

```cpp
// NEVER leave plaintext strings in your binary.
// Every string is a gift to the reverser.

// WARNING: __TIME__[7] only has 10 possible values (0-9).
// That's trivially brute-forced. Use a proper per-build random seed.

// Better: inject a random 64-bit seed at build time via:
//   MSVC: /D BUILD_SEED=0x$(python -c "import random; print(hex(random.getrandbits(64))[2:])")
//   Or: generate a header with a random constant in your CI pipeline

// Production compile-time string encryption (C++17):
// Uses seed injected by build system, not __TIME__
#ifndef BUILD_SEED
#define BUILD_SEED 0xDEADBEEFCAFEBABEULL  // replace in CI
#endif

template<size_t N, uint64_t Seed = BUILD_SEED>
struct XStr {
    char buf[N]{};

    consteval XStr(const char (&s)[N]) {
        // Avalanche the seed per character position
        for (size_t i = 0; i < N; i++) {
            uint64_t k = Seed ^ (Seed >> 17) ^ (i * 0x9E3779B97F4A7C15ULL);
            buf[i] = s[i] ^ static_cast<char>(k ^ (k >> 32));
        }
    }

    // Decrypts into a local stack buffer — never stored as global
    [[nodiscard]] std::string str() const {
        std::string out(N - 1, '\0');
        for (size_t i = 0; i < N - 1; i++) {
            uint64_t k = Seed ^ (Seed >> 17) ^ (i * 0x9E3779B97F4A7C15ULL);
            out[i] = buf[i] ^ static_cast<char>(k ^ (k >> 32));
        }
        return out;
    }
};

// Macro: creates a consteval-encrypted literal, decrypts at call site
#define ENC(s) (XStr<sizeof(s)>(s).str())

// Usage:
auto url = ENC("https://auth.myserver.com/validate");
auto mod = ENC("kernel32.dll");

// For hot paths: decrypt into a fixed-size stack buffer instead of std::string
// to avoid heap allocation traces.
```

### AMSI / ETW Bypass (Stealth Injection)

Before injecting, disable telemetry that would flag your shellcode:

```cpp
// ETW patch: NtTraceEvent → RET (stops kernel-reported image loads)
// Must be done before injection, from a position of trust.
void PatchETW() {
    HMODULE ntdll = GetModuleHandleA("ntdll.dll");
    void* pEtwEvent = GetProcAddress(ntdll, "EtwEventWrite");
    // Patch first byte to C3 (RET x64) — kills ETW event emission
    DWORD old;
    VirtualProtect(pEtwEvent, 1, PAGE_EXECUTE_READWRITE, &old);
    *static_cast<uint8_t*>(pEtwEvent) = 0xC3;
    VirtualProtect(pEtwEvent, 1, old, &old);
}

// AMSI patch (if target uses .NET or script hosts):
void PatchAMSI() {
    HMODULE amsi = LoadLibraryA("amsi.dll");
    if (!amsi) return;
    void* pScan = GetProcAddress(amsi, "AmsiScanBuffer");
    // Patch to: XOR EAX,EAX; RET (return AMSI_RESULT_CLEAN=0)
    DWORD old;
    VirtualProtect(pScan, 6, PAGE_EXECUTE_READWRITE, &old);
    memcpy(pScan, "\x31\xC0\xC3", 3);  // xor eax,eax; ret
    VirtualProtect(pScan, 6, old, &old);
}

// NOTE: These patches are detectable. Do them from:
//   - A kernel driver (invisible to user-mode AC)
//   - A trusted process before cheat process starts
//   - Via syscall to avoid ntdll hook detection
```

### Anti-Debug Implementation

```cpp
// Multi-layered debug detection — check periodically in a thread

bool IsDebuggerAttached() {
    // Method 1: PEB check
    BOOL debugged = FALSE;
    CheckRemoteDebuggerPresent(GetCurrentProcess(), &debugged);
    if (debugged) return true;

    // Method 2: NtQueryInformationProcess
    DWORD debugPort = 0;
    auto NtQIP = (decltype(&NtQueryInformationProcess))
        GetProcAddress(GetModuleHandleA("ntdll"), "NtQueryInformationProcess");
    if (NtQIP) {
        NtQIP(GetCurrentProcess(), ProcessDebugPort,
              &debugPort, sizeof(debugPort), nullptr);
        if (debugPort) return true;
    }

    // Method 3: Hardware breakpoint detection
    CONTEXT ctx = {};
    ctx.ContextFlags = CONTEXT_DEBUG_REGISTERS;
    GetThreadContext(GetCurrentThread(), &ctx);
    if (ctx.Dr0 || ctx.Dr1 || ctx.Dr2 || ctx.Dr3) return true;

    // Method 4: Timing check
    auto t1 = __rdtsc();
    volatile int x = 0;
    for (int i = 0; i < 1000; i++) x += i;
    auto t2 = __rdtsc();
    if ((t2 - t1) > 500000) return true;  // single-stepping detected

    return false;
}

// Response: don't just exit — that tells them where the check is.
// Instead: corrupt internal state, return wrong offsets, crash later.
void OnDebugDetected() {
    g_offsets.player_health ^= 0xDEADBEEF;  // corrupt silently
    g_config.aimbot_fov = 0.0f;              // disable features quietly
    // Crash will come later, far from the detection point
}
```

---

## PART 3: LOADER & DISTRIBUTION ARCHITECTURE

### Loader Architecture

```
┌─────────────────────────────────────────────────┐
│                   USER'S PC                      │
│                                                  │
│  ┌──────────┐    ┌──────────────────────────┐   │
│  │  Loader   │───►│  Auth Server (HTTPS)      │   │
│  │  (.exe)   │    │  - Validate license key   │   │
│  │           │◄───│  - Check HWID binding     │   │
│  │           │    │  - Return encrypted DLL   │   │
│  │           │    │  - Log session metadata    │   │
│  │           │    └──────────────────────────┘   │
│  │           │                                   │
│  │  Decrypt  │                                   │
│  │  payload  │                                   │
│  │     │     │                                   │
│  │     ▼     │                                   │
│  │  Inject   │──────► Game Process               │
│  │  DLL/SC   │                                   │
│  └──────────┘                                    │
└─────────────────────────────────────────────────┘

Flow:
  1. User runs loader
  2. Loader collects HWID fingerprint
  3. Loader sends: { license_key, hwid_hash, loader_version }
  4. Server validates license, checks HWID matches stored binding
  5. Server returns: encrypted cheat payload (unique per user)
  6. Loader decrypts payload in memory (never touches disk)
  7. Loader injects into game process
  8. Loader wipes payload from own memory
  9. Loader exits (or stays for heartbeat)
```

### Licensing System

```
License types (standard in the scene):
  - Day pass:  24h from activation
  - Weekly:    7 days
  - Monthly:   30 days
  - Lifetime:  permanent (higher price, higher crack incentive)

HWID binding:
  - First activation: server stores HWID hash with license
  - Subsequent runs: HWID must match stored hash
  - HWID reset: allow 1-2 resets per month (hardware changes)
  - Multi-PC: some licenses allow 1-3 concurrent HWIDs

Server-side license checks:
  - Expiration timestamp (server time, not client time)
  - Active session count (prevent sharing)
  - IP geolocation (flag if login from different continent)
  - Rate limiting (max N logins per hour)
  - Concurrent session limit
```

### Per-User Build System (Anti-Leak Nuclear Option)

```
The most effective anti-leak technique: every user gets a unique binary.

Implementation:
  1. CI/CD pipeline builds the cheat per-request
  2. Each build has unique watermarks injected:
     - Unique constants in arithmetic operations
     - Unique NOP sled patterns between functions
     - Unique string encryption keys
     - Unique code ordering (function reordering via linker)
     - Unique junk code insertions
  3. Server stores mapping: user_id → build_hash → watermark_set
  4. If a build leaks: extract watermarks → identify leaker → ban

Watermark injection points:
  - Modify immediate values in non-critical comparisons
  - Insert unique byte sequences in padding between functions
  - Vary register allocation in non-critical paths
  - Add unique timing delays in initialization
  - Embed user ID in encrypted form in .rdata

Detection of leaked build:
  - Download cracked version from leak site
  - Extract watermark bytes from known locations
  - Query database: watermark_set → user_id
  - Revoke license, ban user, optionally pursue legally
```

### Payload Delivery Security

```
NEVER serve the cheat DLL as a file download. Never touch disk.

Secure delivery (SOTA 2025):
  1. Server encrypts payload with per-session AES-256-GCM key
     Key = HKDF(user_secret || hwid_hash || server_nonce || unix_ts/300)
     → Key rotates every 5 minutes; replay window is narrow
  2. Payload transmitted over TLS 1.3 (AES-256-GCM inside GCM = fine)
  3. Loader allocates RW memory (VirtualAlloc MEM_COMMIT|MEM_RESERVE)
  4. Decrypts payload directly into that buffer
  5. Performs manual map:
       a. Parse PE headers in memory
       b. Allocate sections at preferred base (or rebase)
       c. Process relocations
       d. Resolve imports via hash-walked PEB (no GetProcAddress)
       e. Call TLS callbacks + DllMain
  6. Mark executable sections PAGE_EXECUTE_READ (not RWX)
     Allocate a separate RW page for writable data
  7. Zero and VirtualFree the original decrypt buffer
  8. Erase PE header from mapped image (first 0x1000 bytes → 0)

SHELLCODE APPROACH (more stealthy than manual-mapped DLL):
  - Compile cheat as position-independent shellcode (PIC)
  - No PE structure → no MZ/PE header → won't show in VAD as an image
  - Allocate with NtAllocateVirtualMemory, write, NtProtectVirtualMemory
  - Execute via thread pool callback (NtQueueApcThread to pool thread)
    or via hijacked fiber/coroutine
  - Shellcode resolves its own imports by walking PEB loader at runtime
  - Result: no identifiable image in memory, no module in LDR tables

Anti-dump on delivered payload:
  - Scatter .text across non-contiguous allocations (function trampolines)
  - Re-encrypt individual functions after first call (VEH-based re-encryption)
  - Hot-decrypt only the function currently executing, re-encrypt on return
  - Guard pages trigger re-encryption VEH handler
  - PE header zeroed → reconstructed dumps will be missing relocations
```

### Auth Server Hardening

```
Your auth server is the #1 target for attackers:

Infrastructure:
  - Host behind Cloudflare/CDN (hide real IP)
  - Rate limit all endpoints aggressively
  - Use certificate pinning in the loader (prevent MITM)
  - TLS 1.3 only, strong cipher suites
  - No admin panel accessible from public internet

API security:
  - All requests signed with embedded client key
  - Replay protection: nonce + timestamp window (±30s)
  - Response signing: loader verifies server response integrity
  - Heartbeat: periodic check-in during active session
  - Kill switch: server can instruct loader to wipe + exit

Anti-emulation of auth server:
  - Challenge-response that requires knowledge of cheat internals
  - Server sends encrypted code snippets that loader must execute
  - Response depends on runtime state (loaded modules, OS version)
  - Rotate challenge logic with each update
```

---

## PART 4: KERNEL SELF-PROTECTION

For high-value cheats operating against kernel ACs (Vanguard, EAC, BE):

```
Problem: AC driver runs at Ring0 and can inspect/terminate your cheat.
Solution: Your cheat also needs kernel presence, or exploits one.

Approach A: BYOVD (Bring Your Own Vulnerable Driver)
  - Load a signed vulnerable driver (see README: Cheat > Vulnerable Driver)
  - Use it to gain kernel R/W primitives
  - From kernel: protect your cheat process
    * Remove your process from EPROCESS list (DKOM)
    * Elevate your process token to SYSTEM
    * Register a fake ObRegisterCallbacks entry to strip AC's handle access
    * Clear your image from PiDDBCacheTable and MmUnloadedDrivers
  - Risk: BYOVD driver may itself be detected (MS blocklist, HVCI)

Approach B: EFI/UEFI pre-boot loader
  - Load your kernel code during UEFI boot phase
  - Runs before Windows kernel, before any AC driver
  - Not visible to PsSetLoadImageNotifyRoutine (never triggers)
  - Can hook Windows kernel at boot time
  - Requires: Secure Boot disabled OR custom key enrolled
  - Hardest to detect, hardest to implement

Approach C: Hypervisor-based protection
  - Run your cheat logic in a Type-2 hypervisor layer
  - AC's kernel code runs as guest → cannot see host memory
  - EPT manipulation to hide your memory from AC's MmCopyMemory
  - Reference: SinaKarvandi/Hypervisor-From-Scratch
  - Extreme complexity; defeats all current kernel ACs

Approach D: Kernel protection without driver (kernel-less)
  - Use undocumented syscalls or data-only attacks
  - Modify kernel data structures via vulnerable driver briefly
  - Unload driver immediately after modification
  - Leave no driver trace in MmUnloadedDrivers

Kernel self-protection primitives:
  PiDDBCacheTable clearing:
    Walk PiDDBCacheTable (undocumented) and remove your driver entry
    Find via pattern scan in ntoskrnl or use known offset for Windows version

  MmUnloadedDrivers clearing:
    Kernel array of last 50 unloaded drivers
    Overwrite your entry with a benign name after unloading

  Handle stripping:
    ObRegisterCallbacks with OB_OPERATION_HANDLE_CREATE
    Strip PROCESS_VM_READ from any handle opened to your process
    → AC cannot read your cheat's memory even from kernel
```

## PART 5: SECURE DEVELOPMENT PRACTICES

### Development Environment

```
Separation of concerns:
  - Dev machine: NEVER has the game or AC installed
  - Test machine: runs the game, disposable VM or spare PC
  - Build server: CI/CD, isolated from both dev and test
  - Distribution server: auth API, separate from everything

Source code protection:
  - Private git repo (self-hosted Gitea/GitLab, NOT GitHub/GitLab.com)
  - GPG-signed commits (detect unauthorized pushes)
  - 2FA + hardware key (YubiKey) on all accounts
  - Full disk encryption on dev machine (BitLocker + PIN, or LUKS + TPM)
  - Separate KeePassXC vault for cheat-related credentials only
  - Consider air-gapped dev machine that never connects to internet

Communication:
  - Signal (individual) or Matrix/Element (team) — NOT Discord for anything sensitive
  - Session app for fully anonymous comms if needed
  - Never discuss implementation details in public channels
  - Use codenames for features in any shared/public-facing docs
  - Separate identity (handle, email, payment) for cheat dev vs personal life
  - Assume all Discord messages are logged and accessible to AC companies
```

### Build Pipeline Security

```
CI/CD (self-hosted Gitea Actions or GitLab CI on your own server):

  1. Pull source from private repo (SSH key auth only)
  2. Generate unique BUILD_SEED (64-bit random) for this build
  3. Compile with full optimizations + LTO + BUILD_SEED define
  4. String encryption baked in at compile time
  5. Import obfuscation post-compile (lazy importer, syscall stubs)
  6. [Per-user] Inject watermarks at known byte positions
  7. Run VMProtect SDK on virtualize-marked functions
  8. Encrypt final payload: AES-256-GCM with master key + build ID
  9. Sign payload with Ed25519 (loader verifies before decrypting)
  10. Upload to CDN/distribution server over authenticated channel
  11. Wipe all intermediate build artifacts (secure delete)
  12. Record: user_id → build_hash → watermark_positions in DB

Automated QA before release:
  - Run own binary through 15+ AV engines (VirusTotal API, private)
  - Entropy analysis: sections should not exceed 7.5 bits/byte
    (very high entropy is a packed-binary heuristic trigger)
  - String scan: grep for any plaintext API names, URLs, filenames
  - Test injection in isolated Win10 + Win11 VMs against target game
  - Verify anti-debug: attach OllyDbg, x64dbg, WinDbg → should detect
  - Verify heartbeat: kill network → cheat should disable itself
```

### Update Strategy

```
When AC updates signatures:
  1. Identify what was detected (signature, behavior, or timing)
  2. If signature: mutate the flagged byte pattern
  3. If behavior: change the technique (different hook, different comm)
  4. If timing: add jitter, change polling intervals
  5. Push update through auth server (loader auto-updates)
  6. Never reuse the exact same payload after a detection wave

Update delivery:
  - Loader checks for update on every launch
  - Server returns version number + update flag
  - New payload downloaded and injected (same flow as normal launch)
  - Old payload version is blacklisted on server (force update)
  - Emergency kill: server responds with KILL command → loader wipes
```

### Incident Response

```
When your cheat gets detected (ban wave):

Hour 0-1:
  - Disable downloads immediately (server-side kill switch)
  - Notify users to stop using current version
  - Begin collecting detection reports (what AC, what game, what features)

Hour 1-6:
  - Analyze detection vector (signature? behavior? new AC update?)
  - If you have a test environment: reproduce the detection
  - Begin developing patch

Hour 6-24:
  - Release patched version through update system
  - Re-enable downloads
  - Monitor for re-detection

Post-incident:
  - Document what was detected and why
  - Add regression test for the detection vector
  - Consider architectural changes if detection was fundamental
```

---

## PART 6: COMMON ANTI-CRACK BYPASSES (KNOW YOUR ENEMY)

### How Crackers Attack Your Cheat

```
Attack 1: Auth server emulation
  - Cracker reverse engineers the auth protocol
  - Sets up fake server that always returns "valid"
  - Patches loader to point to fake server
  DEFENSE: challenge-response tied to cheat internals, cert pinning,
           obfuscate auth protocol, rotate it frequently

Attack 2: Memory dump + reconstruct
  - Run loader legitimately with valid license
  - Dump decrypted payload from game process memory
  - Reconstruct PE from dump, patch out license checks
  DEFENSE: anti-dump (scattered sections, re-encryption),
           continuous server heartbeat (cheat dies without it)

Attack 3: Loader patching
  - Patch conditional jumps in license validation (JNZ → JZ)
  - NOP out HWID checks
  - Simple and effective if loader isn't protected
  DEFENSE: protect loader with VMProtect too, integrity checks,
           server-side validation (not just client-side)

Attack 4: Key sharing / account sharing
  - Users share license keys
  - Multiple people use same account
  DEFENSE: HWID binding, concurrent session limits,
           IP geolocation monitoring

Attack 5: Trial abuse
  - Create infinite free trial accounts
  - Automate account creation
  DEFENSE: phone verification, HWID tracking across accounts,
           manual approval for trials
```

### The "Cracker's Cost" Framework

```
Your protection doesn't need to be unbreakable.
It needs to make cracking MORE EXPENSIVE than buying a license.

Cost calculation for the cracker:
  Time to crack × hourly rate > license price × expected users

If your monthly license is $20 and cracking takes 40 hours:
  40h × $30/h = $1,200 cost to crack
  vs $20/month revenue if they just buy it

But if the crack serves 1000 users at $5 each:
  $5,000 revenue > $1,200 cost → profitable to crack

Conclusion: per-user builds defeat this model because
each crack only works for one user's HWID. The cracker
must repeat the full process for every user.
```

---

## Data Source

**Important**: This skill provides practical guidance. For curated tools and repositories, use the following sources:

### 1. Project Overview & Resource Index

```
https://raw.githubusercontent.com/gmh5225/awesome-game-security/refs/heads/main/README.md
```

Relevant sections:
- `Cheat > Anti Signature Scanning`
- `Cheat > Hide`
- `Cheat > Anti Forensics`
- `Cheat > HWID`
- `Cheat > Spoof Stack`
- `Anti Cheat > Binary Packer`
- `Anti Cheat > Obfuscation Engine`
- `Anti Cheat > Encrypt Variable`
- `Anti Cheat > Lazy Importer`
- `Anti Cheat > Compile Time`

### 2. Repository Code Details (Archive)

**Archive URL format:**
```
https://raw.githubusercontent.com/gmh5225/awesome-game-security/refs/heads/main/archive/{owner}/{repo}.txt
```

### 3. Repository Descriptions

**Description URL format:**
```
https://raw.githubusercontent.com/gmh5225/awesome-game-security/refs/heads/main/description/{owner}/{repo}/description_en.txt
```

**Priority order when answering questions about a specific repository:**
1. Description (quick summary) — fetch first for concise context
2. Archive (full code snapshot) — fetch when deeper implementation details are needed
3. README entry — fallback when neither description nor archive is available
