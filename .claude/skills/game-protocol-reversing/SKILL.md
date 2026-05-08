---
name: game-protocol-reversing
description: Guide for reversing game network protocols, packet analysis, traffic interception, server emulation, and network-based cheat development. Use this skill when analyzing encrypted game traffic, building MITM proxies, reconstructing protocol structures, developing private servers, or understanding server-side anti-cheat validation.
---

# Game Protocol Reversing & Network Security

## Overview

This skill covers the discipline of reversing game network protocols — from capturing raw traffic to fully reconstructing serialization formats, building proxy tools, emulating game servers, and exploiting network-layer weaknesses. This is a fundamentally different skillset from memory hacking: you're working with serialized data over the wire rather than live process memory.

## README Coverage

- `Cheat > Packet Sniffer&Filter`
- `Cheat > Packet Capture&Parse`
- `Cheat > SpeedHack`
- `Game Network > Guide`
- `Game Network > Source`

---

## Protocol Reversing Methodology

### Phase 1: Reconnaissance

Before touching a single packet, understand the target.

```
1. Identify the transport layer
   - TCP: reliable, ordered — MMOs, turn-based, chat, inventory
   - UDP: unreliable, fast — most competitive FPS (CS2, Valorant, Apex)
   - WebSocket: browser/mobile games, HTML5 wrappers
   - QUIC (UDP/443): increasingly common (2023+)
       Valorant: uses custom DTLS over UDP
       Some mobile games: QUIC for reliability without TCP head-of-line blocking
       QUIC encrypts headers too — harder to passively inspect than TCP/TLS
   - DTLS: encrypted UDP (Valorant, some VoIP games)
   - HTTP/S REST: mobile F2P, login/matchmaking, economy endpoints

2. Identify encryption / obfuscation
   - TLS/SSL? → look for certificate pinning in binary
   - QUIC/DTLS? → custom DTLS handshake, often with ECDH key exchange
   - Custom encryption? → static analysis of send/recv functions
   - Protobuf / FlatBuffers? → .proto artifacts, varint patterns in traffic
   - XOR / RC4 / custom stream cipher? → common in older/lazy implementations
   - Obfuscated opcodes? → some games XOR opcodes with a rotating key
   - No encryption? → congratulations, skip to Phase 3

3. Map the network stack in the binary
   - Find send() / recv() / WSASend() / WSARecv() for TCP games
   - For UDP: recvfrom() / sendto() or custom socket wrappers
   - For QUIC: find the QUIC library (msquic, quiche, ngtcp2, custom)
   - Trace back from socket calls to find serialization functions
   - Identify the packet dispatch table (opcode → handler mapping)
   - Find the encryption/decryption wrapper — your golden target

4. Tool selection by transport:
   - TCP/UDP plaintext: Wireshark
   - TLS (with keys): Wireshark + SSLKEYLOGFILE
   - QUIC: qvis, wireshark with QUIC dissector + key log
   - All: Frida hooks (intercept post-decryption regardless of transport)
```

### Phase 2: Traffic Capture & Key Extraction

```
Tools:
  - Wireshark + game-specific Lua dissectors
  - Fiddler / mitmproxy (HTTP/S games)
  - Scapy (custom packet crafting and replay)
  - npcap / WinDivert (Windows packet capture/modification)
  - PcapPlusPlus (C++ packet parsing library)
  - Frida (hook send/recv at runtime — the most universal approach)
  - qvis.quic.tools (QUIC traffic visualization)

For TLS-encrypted games (most common 2024+):
  Method 1: SSLKEYLOGFILE (if game uses OpenSSL/BoringSSL)
    Set env var before launch, Wireshark reads the session keys
    Works on: many Unity games, Node.js backends, Electron launchers
    Does NOT work on: games with custom TLS stacks or static linking

  Method 2: Hook SSL_write / SSL_read with Frida
    Captures plaintext before encryption regardless of implementation
    // Frida script (JavaScript):
    const SSL_write = Module.findExportByName(null, 'SSL_write');
    Interceptor.attach(SSL_write, {
      onEnter(args) {
        const buf = args[1];
        const len = args[2].toInt32();
        console.log('[SSL_write]', hexdump(buf.readByteArray(len)));
      }
    });

  Method 3: Hook at game's own network layer (above crypto)
    Find the function called AFTER decrypt / BEFORE encrypt
    Hook it — you get plaintext + game context (player ID, etc.)
    This is more stable than hooking SSL directly

  Method 4: Patch certificate validation
    Find and NOP the pinning check (compare public key hash)
    Install your own CA cert → full MITM proxy

For DTLS / QUIC games (Valorant, some mobile):
  - DTLS handshake is similar to TLS, hook at application layer
  - Frida hook on the game's own decrypt function is easier
    than dealing with DTLS internals
  - Key extraction: hook the function that receives DTLS app data
  - QUIC: hook quic_stream_read or equivalent in embedded QUIC lib

For custom encryption (older MMOs, some mobile games):
  1. Set breakpoint on send() / WSASend() / sendto()
  2. Walk the call stack back to find the encryption function
  3. Identify the cipher (XOR, RC4, AES-CTR, ChaCha20, custom)
  4. Find the key derivation — usually from handshake or hardcoded seed
  5. Reimplement in your proxy tool or hook post-decrypt instead

Frida all-in-one network intercept (game-agnostic):
  // Hook send/recv at WinSock level for any unencrypted UDP game
  const sendto = Module.findExportByName('ws2_32.dll', 'sendto');
  Interceptor.attach(sendto, {
    onEnter(args) {
      const buf = args[1]; const len = args[2].toInt32();
      console.log('[OUT]', hexdump(buf.readByteArray(Math.min(len, 256))));
    }
  });
```

### Phase 3: Protocol Structure Reconstruction

```
Packet structure (typical pattern):
┌──────────┬──────────┬──────────────────────┐
│ Header   │ Opcode   │ Payload              │
│ (2-4 B)  │ (1-2 B)  │ (variable)           │
└──────────┴──────────┴──────────────────────┘

Header usually contains:
  - Total packet length (2 bytes, little-endian or big-endian)
  - Sequence number (for UDP reliability)
  - Flags (compressed, encrypted, fragmented)

Opcode is the message type ID:
  - Maps to a handler function in the client
  - Finding the dispatch table is the single most valuable RE target
  - Usually a switch/case or function pointer array

Common serialization formats:
  - Protobuf (Google Protocol Buffers) — very common in modern games
  - FlatBuffers — used by some mobile games, zero-copy
  - Custom binary — field-by-field with known types
  - JSON / MessagePack — mobile/web games, easy to reverse
  - Unreal Engine: FArchive serialization, replicated properties
  - Unity: Mirror/MLAPI NetworkVariable serialization
```

### Phase 4: Building the Dispatch Map

This is the core of protocol RE — mapping every opcode to its meaning.

```cpp
// What you're looking for in the disassembly:
// A function that reads the opcode and dispatches to handlers

void PacketDispatcher(uint8_t* data, size_t len) {
    uint16_t opcode = *(uint16_t*)(data + HEADER_SIZE);
    
    switch (opcode) {
        case 0x0001: HandleLogin(data, len); break;
        case 0x0002: HandleMovement(data, len); break;
        case 0x0003: HandleChat(data, len); break;
        case 0x0004: HandleCombat(data, len); break;
        case 0x0010: HandleSpawnEntity(data, len); break;
        case 0x0011: HandleDespawnEntity(data, len); break;
        // ...
    }
}

// Reconstruct this table by:
// 1. Finding the dispatch function (trace from recv)
// 2. Dumping the switch table or function pointer array
// 3. Naming each handler based on the data it reads/writes
// 4. Documenting field offsets within each message
```

---

## Protobuf Protocol Reversing

Many modern games (especially Mihoyo/Hoyoverse, mobile titles, some PC games) use Protocol Buffers.

### Detecting Protobuf

```
Indicators in binary:
  - Strings like "google.protobuf", ".proto", "DESCRIPTOR"
  - Varint encoding patterns in packet payloads
  - Wire type tags (field_number << 3 | wire_type)
  - Embedded .proto descriptors in the binary

Indicators in traffic:
  - Payloads that don't align to fixed-size fields
  - Variable-length integers (MSB continuation bit)
  - Nested length-delimited fields
```

### Extracting .proto Definitions

```
Method 1: From binary descriptors
  - Games compiled with protobuf often embed FileDescriptorProto
  - Use protod, pbtk, or protobuf-inspector to extract
  - Search binary for serialized FileDescriptorSet

Method 2: From decompilation
  - Find generated message classes (C++: ::google::protobuf::Message subclasses)
  - Read field numbers and types from MergePartialFromCodedStream
  - Reconstruct .proto manually

Method 3: From traffic analysis
  - Use protobuf-inspector or blackboxprotobuf
  - Feed raw payloads → get field numbers + wire types
  - Infer field meanings from values and context

Tools:
  - protobuf-inspector (Python) — interactive protobuf decoding
  - blackboxprotobuf (Burp extension) — decode without .proto
  - pbtk — Protobuf Toolkit for extracting descriptors
  - protod — decompile protobuf descriptors from binaries
```

---

## MITM Proxy Architecture

### Passive Proxy (sniff only)

```
                  ┌──────────┐
Client ──────────►│  Proxy   │──────────► Server
                  │ (capture │
                  │  & log)  │
                  └──────────┘

Implementation:
  - WinDivert to redirect traffic at kernel level
  - Transparent proxy: no client modification needed
  - Log all packets with timestamps for replay analysis
  - Parse known structures, dump unknown as hex
```

### Active Proxy (modify packets)

```
                  ┌──────────┐
Client ◄─────────┤  Proxy   ├──────────► Server
       modified   │ (decode  │  original
       response   │  modify  │  or modified
                  │  reencode)│  request
                  └──────────┘

Implementation:
  - Must handle encryption: decrypt → modify → re-encrypt
  - Must maintain sequence numbers and checksums
  - Must handle packet fragmentation/reassembly
  - Can inject entirely new packets (spoofing)
  - Can drop packets (filtering)
  - Can delay packets (lag simulation)

Use cases:
  - Speed hacks (modify timestamp/tick fields)
  - Teleportation (modify position in outgoing packets)
  - Item duplication (replay transaction packets)
  - Unlock content (modify server response data)
  - Economy manipulation (modify trade/purchase packets)
```

### Hook-Based Proxy (internal)

```
Instead of network-level interception, hook the game's own
send/recv wrappers after decryption:

  - Hook the function AFTER the game decrypts incoming packets
  - Hook the function BEFORE the game encrypts outgoing packets
  - You get plaintext without needing to break crypto
  - Can modify data in-place
  - Most stealthy — no external network artifacts

Typical hook targets:
  - ProcessPacket() / HandleMessage() / OnRecv()
  - SendPacket() / SendMessage() / OnSend()
  - The serialization layer (pre-encryption)
```

---

## Server Emulation

### Private Server Architecture

```
┌────────────┐     ┌─────────────────────────────────┐
│ Game Client │────►│ Custom Server                    │
│ (unmodified │     │                                  │
│  or patched)│     │ ┌─────────┐  ┌───────────────┐  │
│             │◄────│ │ Network │  │ Game Logic     │  │
│             │     │ │ Layer   │  │ (state, AI,    │  │
│             │     │ │         │  │  physics,      │  │
│             │     │ │ decode/ │  │  spawning)     │  │
│             │     │ │ encode  │  │                │  │
│             │     │ └────┬────┘  └───────┬───────┘  │
│             │     │      │               │          │
│             │     │ ┌────┴───────────────┴───────┐  │
│             │     │ │ Database (accounts, world, │  │
│             │     │ │ inventory, progression)    │  │
│             │     │ └────────────────────────────┘  │
└────────────┘     └─────────────────────────────────┘

Steps to build:
  1. Fully reverse the client→server protocol
  2. Reverse the server→client protocol (login, world state, etc.)
  3. Implement packet encoding/decoding matching the client's expectations
  4. Redirect client to your server (hosts file, DNS, or binary patch)
  5. Implement enough game logic to get past login
  6. Incrementally implement features: spawning, movement, combat, etc.
```

### Client Patching for Private Servers

```
Common patches needed:
  - Server IP/hostname redirect (string patch or hosts file)
  - Certificate pinning removal (for TLS games)
  - Version check bypass
  - Login token validation bypass
  - Anti-cheat disable (won't connect to AC backend)

Methods:
  - Binary patching: direct hex edit of the executable
  - DLL injection with hooks on connect() / getaddrinfo()
  - DNS spoofing (local hosts file or custom DNS server)
  - Proxy with SNI/hostname rewrite
```

---

## Network-Based Cheat Types

### Position Manipulation

```
Outgoing movement packets usually contain:
  - Position (X, Y, Z float32)
  - Velocity (X, Y, Z float32)
  - View angles (pitch, yaw)
  - Movement flags (jumping, crouching, sprinting)
  - Timestamp / tick number

Attacks:
  - Teleportation: modify position to arbitrary coordinates
  - Speed hack: scale velocity or reduce tick intervals
  - No-clip: modify movement flags to ignore collision
  - Fly hack: zero gravity flag or constant upward velocity

Server-side validation common for:
  - Maximum speed checks (flag if distance/time > threshold)
  - Position delta checks (reject impossible movement)
  - Anti-teleport: server authoritative position
```

### Combat Exploitation

```
Damage packets / hit registration:
  - Some games: client reports hits → server validates
  - Others: client sends intent → server does hit detection
  - Worst case (client-authoritative): client sends damage directly

Attacks on client-reported hits:
  - Force headshot flag on every hit report
  - Modify damage values in outgoing packets
  - Ignore distance/LOS checks client-side

Attacks on server-authoritative systems:
  - Still possible via timing manipulation
  - Input injection to achieve "perfect" aim timing
  - Lag compensation abuse (backtrack)
```

### Economy Manipulation

```
Trade / purchase / crafting packets:
  - Item ID, quantity, source slot, destination
  - Currency amounts
  - Recipe IDs

Race condition attacks:
  - Send multiple purchase packets before server deducts currency
  - Cancel transaction after receiving item but before deduction
  - Replay valid transaction packets

Server-side defenses:
  - Atomic transactions with locking
  - Idempotency keys (unique per-transaction)
  - Rate limiting on economic actions
  - Server-authoritative inventory
```

---

## Tools & Libraries

### Packet Capture & Analysis

| Tool | Language | Purpose |
|------|----------|---------|
| **Wireshark** | C | Full protocol analyzer, custom dissectors via Lua, QUIC dissector built-in |
| **npcap** | C | Windows packet capture library (WinPcap successor) |
| **PcapPlusPlus** | C++ | Packet parsing/crafting library |
| **Scapy** | Python | Packet crafting, sending, sniffing, dissecting |
| **WinDivert** | C | Windows kernel-level packet capture/modify/drop/inject |
| **mitmproxy** | Python | HTTP/S + QUIC MITM proxy with scripting API |
| **Fiddler Everywhere** | C# | HTTP/S + WebSocket debugging proxy |
| **Charles Proxy** | Java | HTTP/S proxy with SSL proxying |
| **Frida** | Python/JS | Runtime instrumentation — hooks any function post-decryption |
| **qvis** | Web | QUIC traffic visualization and analysis |

### Protocol Reconstruction

| Tool | Language | Purpose |
|------|----------|---------|
| **protobuf-inspector** | Python | Decode protobuf without .proto files |
| **blackboxprotobuf** | Python | Decode unknown protobuf, Burp extension |
| **pbtk** | Python | Extract .proto from binaries |
| **protod** | Go | Decompile protobuf descriptors from binaries |
| **Kaitai Struct** | Multi | Binary format parser generator with web IDE |
| **010 Editor** | — | Hex editor with binary templates marketplace |
| **ImHex** | C++ | Open-source hex editor with pattern language |
| **binwalk** | Python | Firmware/binary analysis, find embedded structures |

### Proxy Development

| Library | Language | Purpose |
|---------|----------|---------|
| **WinDivert** | C | Kernel-level packet diversion (transparent proxy) |
| **ndisapi** | C++ | NDIS-level packet filtering (win-shaper in README) |
| **libuv** | C | Async I/O for proxy event loops |
| **tokio** | Rust | Async runtime for Rust proxy tools |
| **Netfilter (iptables/nftables)** | C | Linux packet filtering/NAT |
| **nfqueue** | C/Python | Linux userspace packet queue |
| **boringssl / mbedtls** | C | Embed TLS in custom proxy for game MITM |

---

## Game-Specific Protocol Notes

### Valorant (DTLS + custom)
```
Transport: Custom DTLS over UDP port 7086
Encryption: DTLS 1.2, Riot's own TLS stack (linked statically)
Anti-RE: UE4 binary, packed with custom obfuscation
Approach: Hook at UE4 networking layer (UNetDriver::SendRawBunch)
          or at UTcpNetDriver::Shutdown level
Server-auth: Full server-side hit detection, Fog of War
             Client only reports input — almost no packet manipulation possible
```

### CS2 (Valve's custom UDP)
```
Transport: UDP with Valve's custom reliability layer
Protocol: Source Engine 2 network protocol
Encryption: Lightweight XOR-based obfuscation + Valve's net_channel
Approach: Hook CNetChan::ProcessPacket or CNetMessage handlers
          Abundant open-source RE work on this protocol
Server-auth: Partial — some client-side simulation + server correction
```

### GenshinImpact / Hoyoverse games
```
Transport: UDP with custom protocol ("KCP" reliability layer)
Encryption: XOR with per-session key + MT19937 PRNG
Obfuscation: Protobuf with obfuscated field numbers (rotated per version)
Approach: Hook post-decryption in client, extract session key from memory
          Community tools: GenshinProxy, Cultivation (private server)
Protobuf: Field numbers change each update — requires re-mapping per patch
```

### EFT (Escape from Tarkov)
```
Transport: UDP with custom protocol
Engine: Unity, Photon PUN networking
Approach: Hook Photon SDK callbacks (PhotonPeer::ReceiveIncomingCommands)
          OR hook at Unity NetworkManager level
Packets: Photon's binary format (not protobuf)
Risk: BSG's server-side anti-cheat is aggressive; loot/item manipulation
      is quickly detected via server-side validation
```

### Mobile Games (general)
```
HTTP/S API games:
  - Use mitmproxy + SSL kill switch (Android) or SSL Unpinning (iOS Frida)
  - API endpoints are usually REST/JSON or REST/protobuf
  - Replay attacks on purchase/reward endpoints are common attack vector

WebSocket games:
  - mitmproxy websocket addon for live interception/modification
  - Charles Proxy for initial mapping

QUIC mobile games:
  - Frida hook on QUIC library (quiche, ngtcp2, or custom)
  - Intercept after QUIC decryption, before app logic
```
---

## Encryption Breaking Workflow

### Common Game Encryption Patterns

```
Tier 0 — No encryption:
  Plaintext over TCP/UDP. Rare in 2024+ but still found in:
  - Indie games
  - LAN-only games
  - Some older MMO emulators

Tier 1 — XOR / Simple stream cipher:
  - Static XOR key (found in binary)
  - Rolling XOR (key derived from packet counter)
  - Custom substitution ciphers
  → Break: find key in binary or derive from known-plaintext attack

Tier 2 — RC4 / Custom symmetric:
  - Session key exchanged during handshake
  - RC4 still common due to simplicity
  - Custom stream ciphers (game-specific)
  → Break: hook key exchange, extract session key

Tier 3 — AES / ChaCha20 + Diffie-Hellman:
  - Proper key exchange (DH, ECDH)
  - AES-GCM or ChaCha20-Poly1305
  - Per-packet nonces
  → Break: hook post-decryption, or extract DH private key from memory

Tier 4 — TLS with certificate pinning:
  - Standard TLS 1.2/1.3
  - Certificate pinning (public key or cert hash check)
  → Break: patch pinning check, install custom CA, hook SSL_read/write

Tier 5 — TLS + token-based auth + server-side validation:
  - Full TLS with mutual auth
  - JWT/OAuth tokens
  - Server-authoritative game state
  → Hardest to exploit; focus shifts to client-side hooks
```

### Known-Plaintext Attack on Custom Crypto

```
Many custom game ciphers are vulnerable to known-plaintext:

1. Identify a packet with predictable content
   - Login packet (contains username you control)
   - Heartbeat/ping packet (fixed structure)
   - Version check (known version string)

2. XOR the ciphertext with the known plaintext
   → For XOR cipher: gives you the key directly
   → For stream cipher: gives you the keystream at those offsets

3. If keystream repeats (short key or key reuse):
   → Full decryption capability with enough samples

4. For rolling keys:
   → Capture multiple packets, correlate sequence numbers with keystream
   → Derive the PRNG state or key schedule
```

---

## Server-Side Anti-Cheat (Network Perspective)

### What Servers Validate

```
Modern server-side validation typically checks:

Movement:
  - Speed (distance / time between position updates)
  - Teleportation (delta > physically possible)
  - Clipping (position inside geometry — requires server-side navmesh)
  - Fly detection (sustained Y-axis without ground contact)

Combat:
  - Hit rate (statistical improbability)
  - Damage values (within weapon parameters)
  - Fire rate (faster than weapon allows)
  - Line of sight (server-side raycast on hit reports)
  - Range (target within weapon effective range)

Economy:
  - Transaction atomicity
  - Rate limiting
  - Value bounds checking
  - Cross-referencing inventory state

Timing:
  - Client tick rate deviation from expected
  - Packet timing regularity (too perfect = bot)
  - Latency spike correlation with advantageous actions
```

### Bypassing Server-Side Validation

```
Strategies:
  1. Stay within bounds: keep speed 1.1x instead of 10x
  2. Gradual escalation: slowly increase advantage to avoid threshold triggers
  3. Noise injection: add randomization to avoid statistical detection
  4. Timing variance: don't send packets at perfect intervals
  5. Client-side only: focus on information cheats (ESP) that don't modify outgoing data
  6. Prediction abuse: use latency compensation to your advantage (legal backtrack)
```

---

## Data Source

**Important**: This skill provides methodology and conceptual guidance. For curated tools and repositories, use the following sources:

### 1. Project Overview & Resource Index

Fetch the main README for the full curated list of repositories, tools, and descriptions:

```
https://raw.githubusercontent.com/gmh5225/awesome-game-security/refs/heads/main/README.md
```

Relevant sections:
- `Cheat > Packet Sniffer&Filter`
- `Cheat > Packet Capture&Parse`
- `Cheat > SpeedHack`
- `Game Network > Guide`
- `Game Network > Source`

### 2. Repository Code Details (Archive)

For detailed repository information (file structure, source code, implementation details), the project maintains a local archive. If a repository has been archived, **always prefer fetching from the archive** over cloning or browsing GitHub directly.

**Archive URL format:**
```
https://raw.githubusercontent.com/gmh5225/awesome-game-security/refs/heads/main/archive/{owner}/{repo}.txt
```

**Examples:**
```
https://raw.githubusercontent.com/gmh5225/awesome-game-security/refs/heads/main/archive/basil00/Divert.txt
https://raw.githubusercontent.com/gmh5225/awesome-game-security/refs/heads/main/archive/ValveSoftware/GameNetworkingSockets.txt
https://raw.githubusercontent.com/gmh5225/awesome-game-security/refs/heads/main/archive/seladb/PcapPlusPlus.txt
```

**How to use:**
1. Identify the GitHub repository the user is asking about (owner and repo name from the URL).
2. Construct the archive URL: replace `{owner}` with the GitHub username/org and `{repo}` with the repository name (no `.git` suffix).
3. Fetch the archive file — it contains a full code snapshot with file trees and source code generated by `code2prompt`.
4. If the fetch returns a 404, the repository has not been archived yet; fall back to the README or direct GitHub browsing.

### 3. Repository Descriptions

**Description URL format:**
```
https://raw.githubusercontent.com/gmh5225/awesome-game-security/refs/heads/main/description/{owner}/{repo}/description_en.txt
```

**Priority order when answering questions about a specific repository:**
1. Description (quick summary) — fetch first for concise context
2. Archive (full code snapshot) — fetch when deeper implementation details are needed
3. README entry — fallback when neither description nor archive is available
