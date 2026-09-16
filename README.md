# Agentic User Interface Engine (AUI)

**Documentation-only repository. The source code is not public.**
*[Versione italiana](./README.it.md)*

> **Wallpaper Engine, but for assistants.** Wallpaper Engine ships almost no
> wallpapers of its own: it is the thing that runs the ones other people make,
> plus a Workshop that moves them around. AUI is that layer for the desktop
> assistant — the engine underneath the character, not another character.

**User-made agents.** An agent — persona, voice, face, expressions, effects —
authored as a portable folder by the person who uses it, and installable by
anyone else, rather than compiled into an application.

Existing terms name what an agent *does*: the interfaces it drives, the cloud
environment that executes it, the enterprise avatar it wears. None of them names
*who authors it*. That category had no name, so **user-made agents** is the term
I use for it — proposed by **Alejandro Lopez**
([TheMackerel](https://github.com/TheMackerel)), September 2026. **AUI Engine**
is the runtime built to host them, and this repository documents it.

---

## What it is

The name is the specification, one word at a time.

- **Agentic** — what runs on top is an agent, not a picture: it hears you,
  answers in its own voice, moves its face while it speaks, and remembers what
  you just said.
- **User** — it runs on the machine of the person using it. Their GPU, their
  models, their files. No account, no cloud round trip, no subscription.
- **Interface** — the agent is something you see and talk to: a transparent
  overlay on the desktop, a face, a voice, a chat window. Not an API, not a
  terminal.
- **Engine** — the base architecture, and nothing above it. Who the character is,
  how it sounds, how it looks, which expressions and effects it has: that is
  content, and content belongs to whoever makes it.

So an agent is a folder, and the engine runs folders. The one shipped with the
app is **Mia** — the demo scene, the way a game engine ships one. Everything that
makes her *her* lives in that folder and none of it lives in the app: put another
folder next to hers and the desktop has someone else's character on it, with
their voice and their expressions, without recompiling anything.

**What it is not:** a collection of AI agents, and not an agent framework for
developers. It is the layer below both — the part that has to work before anyone
can drop in their favourite character.

Mia is the first agent running on it, and the product going to Steam.

> [DA INSERIRE] 20–30 s demo video
> [DA INSERIRE] screenshot lipsync
> [DA INSERIRE] clip swap characters

---

## The problem

Shipping one assistant is an app. Turning it into something other people can
build on is a different problem, and four hard constraints sit underneath it.

- **One application hardcodes one character.** Changing the voice, the
  expressions, the face or the model means recompiling — which is exactly what
  makes an assistant an app instead of a platform. This is the reason for an
  engine.
- **Three models, three lifecycles.** Speech-to-text, language model and
  synthesis are separate runtimes with incompatible dependency trees. Linked
  in-process, a model that dies takes the window with it.
- **VRAM is the budget, and it is shared.** A 4B model at 4 bits takes 3,396 MiB
  on an 8 GB card at 8k context (measured below). What is left has to be enough
  for the game in the other window.
- **The audio clock is a hard constraint.** Lip-sync is a timeline anchored to
  what the speakers actually played. Drift is visible on a face before it is
  measurable in a log.
- **The desktop is not a window.** A transparent always-on-top overlay decides,
  per pixel, whether a click belongs to the agent or to the desktop behind it.

The first point is why this is an engine. The others are why the engine has to
own the processes, the protocol, the timing and the security fence — and why
everything above that line is *declared* by the pack instead of coded into the
app.

---

## Architecture

```mermaid
flowchart LR
    subgraph desktop["Windows desktop"]
        UI["Transparent always-on-top overlay<br/>avatar · chat · per-pixel hit test"]
    end

    subgraph engine["AUI runtime"]
        CORE["Native core<br/>turn orchestration · audio ring · protocol filter"]
        SUP["Sidecar supervisor<br/>spawn · health · OS-assigned ports · kill-on-close"]
        CON["Contracts<br/>voice engine · avatar · character pack"]
    end

    subgraph models["Local model processes"]
        STT["Speech-to-text"]
        LLM["Language model"]
        TTS["Speech synthesis"]
    end

    PACK["Agent pack — a folder<br/>persona · voice binding · avatar · emotions"]
    MEM[("Local memory<br/>profile + rolling history")]

    UI <--> CORE
    CORE --> SUP
    SUP --> STT
    SUP --> LLM
    SUP --> TTS
    PACK --> CON
    CON --> CORE
    CORE <--> MEM
```

**Lifecycle of an agent.**

```mermaid
stateDiagram-v2
    [*] --> Discovered
    Discovered --> Validated: manifest parsed, gaps reported in the log
    Validated --> Bound: voice and avatar resolved against what exists
    Bound --> Active: prompt composed once, cached per character and language
    Active --> Bound: character switch
    Active --> [*]: shutdown
    Validated --> Listed: manifest broken
    Listed --> [*]: stays in the list with its reason, never silently dropped
```

**One turn.**

```mermaid
sequenceDiagram
    participant U as User
    participant E as Engine
    participant S as Speech-to-text
    participant L as Language model
    participant T as Synthesis
    participant A as Avatar

    U->>E: push to talk, microphone into a ring buffer
    E->>S: audio
    S-->>E: transcript and detected language
    E->>L: persona + scene + app instructions + memory + rolling history
    L-->>E: token stream
    Note over E: a state machine splits the stream — what is spoken, what is only shown
    E-->>U: text appears while it streams
    E->>T: the spoken part, as soon as it is complete
    T-->>E: audio
    E->>A: viseme timeline, spread over the real audio duration
    E-->>U: voice and lip-sync, anchored to the audio hardware clock
```

**Memory**, today: a persisted profile — name, preferences, projects, recent
topics — injected into the prompt, plus a rolling conversation window capped by a
character budget rather than a turn count, so laconic turns do not collapse
continuity and long ones do not blow the context. The long-term layer is designed
and unwritten (see Roadmap).

---

## The Workshop: how someone else's character gets on the desktop

```mermaid
flowchart LR
    subgraph creator["A creator"]
        MAN["A folder<br/>manifest + persona text"]
        DECL["Declares: character · voice<br/>expressions · motion · effects"]
    end

    subgraph dist["Distribution"]
        DIR["User packs directory<br/>works today"]
        WS["Steam Workshop<br/>subscribe and update — planned"]
    end

    subgraph engine["AUI runtime"]
        DISC["Discovery at startup<br/>bundled + user folder<br/>user pack wins on the same id"]
        VAL["Validation<br/>warns and keeps going<br/>path fence refuses"]
        RUN["Active agent"]
    end

    MAN --> DECL
    DECL --> DIR
    DECL -.-> WS
    DIR --> DISC
    WS -.-> DISC
    DISC --> VAL
    VAL --> RUN
```

*Dotted = planned, not implemented.*

A pack is a folder in the user's own directory, found at startup next to the
shipped one. Same id as a bundled pack, and the user's copy wins — so a shipped
character can be corrected without patching the app. A broken manifest does not
make a pack disappear: it stays in the list, marked invalid, with the reason,
because a character that vanishes without explanation is worse than one that
cannot smile. Validation warns rather than refuses, with one exception: a pack
that points **outside its own folder** is refused, since that is not an aesthetic
defect. Voice engines travel exactly the same way — a folder with a manifest, and
the proof is a working backend made of a manifest and a short script, with no
native code in it.

| Works today | Declared, no consumer yet | Planned |
|---|---|---|
| Persona text per language · voice binding per language · emotion vocabulary · expression map · user pack overrides bundled · path fence · invalid packs stay listed · voice engines as folders | `[effects]` (the slot is parsed, validated and logged — deliberately empty) · `[avatar.motion]` amplitudes (the motion layer is Roadmap 1) · the avatar model inside the pack (the schema accepts it; loading a rig from the pack folder is Roadmap 2) | Steam Workshop as the distribution channel · importing third-party character cards, with their prompt-bearing fields discarded |

---

## Key technical decisions

The section I would want to be interviewed on. Each one: what I chose, what I
rejected, what it cost.

**1. Rust and Godot — not Electron, not Unity, not a Python stack.**
Rejected: a webview with a Python backend (the category default: GC pauses and
IPC in the audio path), Unity (weight and licensing for a 2D overlay), Python
end-to-end (a 3–4 GB runtime in the installer, and a collector under the mouth).
*Cost:* a narrow FFI surface with its own traps, paid in code instead of latency.

**2. Models as local sidecars, not in-process bindings.**
The app owns the processes: spawned inside a Windows job object with
kill-on-close, so a crash cannot leave a multi-gigabyte model resident; ports
requested from the OS instead of fixed, so nothing collides; a CPU feature
preflight before each spawn, so an unsupported instruction set produces a message
instead of a silent crash.
*Cost:* serialization per call, and supervision becomes my problem.

**3. Threads and blocking calls, no async runtime — and cancellation by
generation id.**
The native layer lives inside the engine's frame loop, where an accidentally
initialized async runtime is a class of bug nobody wants to debug at 60 fps. Each
turn bumps a counter; the stream worker re-reads it per line and drops the
connection when superseded, so the model stops computing rather than being
ignored. Stopping drains three reservoirs at once: audio ring, playback buffer,
lip-sync state.
*Cost:* a call already in flight cannot be cancelled — a late synthesis is
discarded on return, so the user hears silence, not a stale line.

**4. The protocol belongs to the app, not to the pack.**
Plain text with a marker separating what is spoken from what is only shown; a
pure state machine splits the stream and strips markers before anything reaches
the user or the speakers. Imported third-party character cards have their
prompt-bearing fields discarded by design — an app that executes prompts shipped
inside content is injectable through content, and on a platform the content comes
from strangers. Rejected: a grammar-constrained JSON turn, which small models
degrade badly under.
*Cost:* the model will get the format wrong, so the degradation ladder is code —
16 golden cases, each replayed at several stream chunk sizes, because a marker
split across two tokens is not a bug you find by hand.

**5. A voice engine is a manifest, not an integration.**
A folder declares how to launch it, how to ask, how to read the answer, which
voices it offers. Two shapes are implemented: a resident local server, and one
process per line. The proof is a backend made of a manifest and a short script,
no native code: it appears in the list, speaks, and lip-syncs, with nothing
recompiled. Every persisted format carries a schema version from day one —
readers migrate the previous one forward with a backup, and treat an unknown
future version as read-only.
*Cost:* the manifest must describe what a hardcoded integration would just do.

**6. Dropped the better-sounding synthesis engine for the small one, on numbers.**
The candidate with voice cloning measured RTF 2.01x and 6.8 s to first audio
against a 3–4 s target. Replaced by an 82M ONNX model that runs faster than real
time on CPU.
*Cost, accepted and written down:* no voice cloning in 1.0. The gain is
structural — synthesis leaves the VRAM budget entirely, and a 3–4 GB Python/CUDA
dependency leaves the installer with it.

**7. Lip-sync computed by the app, not supplied by the engine.**
Text to phonemes via an external grapheme-to-phoneme binary run as a separate
process (also the clean answer to its licence), mapped to viseme classes, spread
over the real audio duration. Any engine that returns audio gets lip-sync.
Rejected: requiring engines to emit phoneme timestamps — exactly one candidate
did.
*Cost:* proportional distribution is an approximation, not forced alignment; and
the port left two implementations of one algorithm alive, so a golden parity test
with a 5 ms tolerance exists to catch them drifting apart.

**8. The avatar sits behind a capability contract.**
The app says "look there", "this mouth pose", "happy". The backend declares what
the rig can do and receives only what it can apply, through one written
degradation table tested with the engine switched off: 16 phoneme classes, to
five vowel morphs, to two axes, to open-only. This is what lets a pack bring a
face the app has never seen. Rejected: one implementation per rig format — three
answers to the same question are three different bugs.
*Cost:* a compromise for every rig instead of the best mapping for one. The 3D
backend is what will actually prove the contract.

**9. Speak when the speech is complete, not when generation is finished.**
The protocol marker guarantees nothing after it will be spoken, so the moment the
filter enters display-only, no further sentence can reach synthesis. Measured:
first word from ~24.5 s to ~9.0 s, same turn, same machine. Per-sentence
chunking, which would also mask latency *inside* the answer, is on indefinite
hold: it broke the sync anchor, reopened a class of stale-audio bug, and split
prosody across calls.
*Cost:* no latency masking within the spoken part.

---

## Constraints and measured numbers

Development machine: i9-9900KF, RTX 2080 SUPER 8 GB, Windows 10.
Declared target: the Steam hardware survey median — 6–8 cores, 16 GB RAM, 8 GB
VRAM.

| Measure | Value | Source |
|---|---|---|
| VRAM, 4B model at 4 bits, 8k context, quantized KV cache | **3,396 MiB** — 2,495 model + 384 context + 517 compute | runtime memory breakdown, recorded test |
| VRAM per agent | one language model; synthesis runs on CPU by design, speech-to-text is CPU-only | design constraint |
| First word of a spoken answer | **~24.5 s → ~9.0 s**, same turn | before/after logs of the streaming rework |
| Speech-to-text, 2 s clip, small model on CPU | **3.95 s → 1.47 s** (~2.7x) after sizing the analysis window to the clip | recorded measurement |
| Rejected synthesis engine | RTF **2.01x**, first audio **6.8 s** vs a 3–4 s target | the benchmark that closed the decision |
| Lip-sync parity with the reference implementation | 10 phrases, tolerance **5 ms** | golden test in the suite |
| Per-pixel hit test, 800x700 overlay | GPU readback median **1.822 ms/frame** against a 0.300 ms budget; the geometric test it would replace: **1.033 ms/call** | instrumented run |
| Agreement between the two hit tests | **96.87%** over 19,312 sampled points, every disagreement in one direction | golden comparison |
| Model sizes on disk | LLM 3.1 GB · speech-to-text 190 MB · synthesis 326 MB + 28 MB voices | files |
| Audio ring / microphone ring | 20 s at 24 kHz, ~1.9 MB, lock-free SPSC / 180 s, ~35 MB at 48 kHz, raised from 60 s after a 70 s utterance was truncated | code |
| Test suite | **100 passed, 0 failed, 3 ignored, 2.02 s** | `cargo test`, 2026-09-15 |

The per-pixel hit test is how decisions get made here: written, measured, and
switched off, because it costs six times its budget and the cheaper method it
replaces is not free either. It ships when its gate opens, not on a date.

---

## Mia, the first agent

The demo scene. Everything a single-purpose assistant would hold in code, she
declares:

```toml
schema_version = 1

[character]
id   = "mia"
name = "Mia"

[files]                       # persona, per language
it = "character.it.md"
en = "character.en.md"

[voice]                       # a binding, not an order: the user's choice wins
it = { backend = "kokoro", voice = "if_sara" }
en = { backend = "kokoro", voice = "af_heart" }

[emotions]                    # the single source of the emotion vocabulary
available = ["happy", "excited", "sad", "shy", "shocked", "angry", "neutral"]

[avatar]
type = "live2d"

[avatar.emotions]             # intent -> expression id, as the rig names it
happy   = "exp_02"
excited = "exp_04"
neutral = "exp_01"

[avatar.motion]
breath = 1.0
sway   = 0.6

[effects]                     # declared and empty, on purpose
```

- **The emotion vocabulary has one owner.** The same list feeds the prompt, the
  filter that recognizes the tag mid-stream, and the router that plays the
  expression. If the rig cannot render something the vocabulary promises, the
  parser says so in the log — in both directions.
- **Switching character rotates persona, voice and emotions together**, because
  they are three lines of one file.
- **The empty `[effects]` block is deliberate.** The slot is parsed and validated
  before anything consumes it, the same way the scene folder existed before there
  was a scene: an engine declares where content will go, then goes and builds the
  consumer.
- **A broken pack does not break the app**: it stays listed, marked invalid, with
  the reason.

> [DA INSERIRE] Steam page link.

---

## Current state

**Working, verified by tests or by a recorded run.** Sidecar supervision: three
local model processes owned by the app, kill-on-close, OS-assigned ports, CPU
preflight, local crash log. Speech-to-text on a resident CPU-only server with a
command-line fallback. Streaming language turn with real cancellation, and a
prompt composed from pack, scene and app instructions — cached, and ordered so
the stable prefix survives the model's own cache. Manifest-driven synthesis, two
engine shapes, voice precedence user over pack over engine default, path fence on
everything a pack declares. Client-side lip-sync with a golden parity test.
Avatar contract plus a 2D backend, expressions coming from the manifest,
transparent click-through overlay with a per-pixel silhouette. Persisted profile
and rolling history. Chat surface and status strip, on the working branch.

**In progress.** The avatar extraction is done — the app no longer names the rig
format outside its backend — with two items left: the per-frame cost of the
geometric hit test, and excluding non-drawing interaction areas from the
silhouette. The per-pixel hit test is written and disabled, waiting on its gate.
Screen-level acceptance of the rebuilt scene is pending: there is no headless
engine here, so that verdict is a human one.

**Known debts, tracked.** Twenty-one open items, each recorded with where it was
verified. The two worth stating in public: the build still carries absolute
development paths, so the repository does not start unmodified on another
machine; and the language selector exists in code with nothing calling it, which
pins the current build to English.

---

## Roadmap

Not implemented. In execution order, each gated on a result rather than a date.

1. Procedural motion layer — breathing, micro-saccades, nods on the audio
   envelope. Gate: a blind A/B on five people, four out of five.
2. 3D avatar as a second backend, and a pack that carries its own model. The real
   test of the avatar contract — and the point where a creator can bring a face,
   not only a personality.
3. Sidecar supervisor as its own subsystem: health sweeps, restart policy,
   suspend and resume.
4. GPU orchestrator and simultaneity tests — the remaining VRAM war is the
   language model against whatever game is running.
5. Long-term memory: a plain-text vault the user owns and can read outside the
   app, with a local index. Designed, no code.
6. Chat GUI as a specified subsystem; settings, multi-monitor and DPI.
7. Security, compliance and first run, including a hard licensing gate before any
   release build.
8. Workshop: packs distributed and updated through Steam, and third-party
   character card import. The in-app browser is deliberately deferred — the first
   months open the Steam page instead.

After 1.0: open microphone with cascaded voice detection, wake word, mid-sentence
emotion changes, contextual awareness, and optional cloud offload with the user's
own key — never silently, with three explicit modes.

---

## Method

Work started on 16 June 2026. Development is AI-assisted: I use Claude Code
as the implementation tool.

Mine is the architecture, the constraints, the specification and the direction —
what gets built, in what order, what counts as accepted, what is rejected and
why. Work is specified in numbered work orders with their own acceptance criteria
before it is written; deviations are recorded the day they are taken, with the
reason, and so are the options considered and refused, so they do not come back.

The part that matters is the verification discipline. Nothing is called done
without a test, a measurement or a named human verdict, and the things only a
person can judge — whether a mouth looks like speech, whether an interruption
sounds instant — stay open with a name on them instead of being quietly assumed.
Every number in this document comes from a test, a log or a recorded measurement
on the machine above.

> [DA INSERIRE] contact link.

---

© 2026 Alejandro Lopez. Documentation only: this repository distributes no source
code, no binaries and no assets. All rights reserved.
