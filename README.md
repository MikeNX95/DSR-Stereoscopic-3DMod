# DSR-Stereoscopic-3DMod
Dead Space Remake (2023) Stereoscopic 3D &amp; 6DOF Head Tracking Patch
# Dead Space (2023 Remake) — Stereoscopic 3D (AFR) + Head Tracking

This extends your `DeadSpaceRemake-HeadTracking` mod with a real stereoscopic
3D output, ported from **GBFR-Stereoscopic-3Dmod** (a DirectX 11 title) onto
Dead Space Remake's **DirectX 12** renderer (Motive/Frostbite-derived, DX12
is the only supported API on PC for this title).

## Fixes applied in this revision

You asked me to compare this package against `RDR2-3D-6DOF.zip` — a later,
more mature build from the same lineage — and port back anything DSR was
missing. That comparison turned up two genuine correctness issues in the
first DSR port, both fixed here by adopting RDR2's proven design. (RDR2 also
has a VR shared-surface bridge, FOV/Z-dolly camera layers, and depth-based
reprojection — those are a different scope than this AFR port was built for,
and weren't ported; see "What I didn't port" below.)

**1. The DX12 command-queue capture had a startup race.** The original
design hooked `IDXGIFactory::CreateSwapChain`/`CreateSwapChainForHwnd` from
`DllMain`, hoping to win the race against the game creating its own
device/queue/swapchain. RDR2's build replaced this with a cleaner technique
that has no race at all: create our **own** throwaway D3D12
device+queue+swapchain purely to read their vtables, hook
`IDXGISwapChain::Present` and `ID3D12CommandQueue::ExecuteCommandLists` on
those, then tear the dummies down. Because COM vtables are shared per class,
this patches the *game's* real objects too, however they were created and in
whatever order — `Stereo_Init()` can now run at its normal, unhurried
worker-thread timing instead of needing to beat `DllMain` to the punch.

**2. The eye was derived in two places.** The original design had the
camera hook derive an eye from present parity *and* push it into a queue
Present later popped from — two derivations of the same fact, kept in sync
only as long as camera writes and presents stayed perfectly 1:1. If a camera
frame was ever skipped or throttled, the queue could drift out of step with
which back buffer it was really describing. This revision makes Present the
**single owner**: it assigns the eye once per frame, unconditionally, from a
plain present-parity counter; the camera hook only *reads* that value. There
is exactly one place the eye is ever written.

Also carried over: a menu/cutscene guard, warmup, and eye-slot bookkeeping
that distinguish "camera wrote this frame" from "camera has ever written"
(fixes a black-screen-at-startup edge case), a **pipeline-depth pin**
(`AfrPipelineDepth`, default 1) that bounds how far the CPU can run ahead of
the GPU so L/R parity can't drift, and diagnostic counters — `eyeflips/s`,
`sameslot/s`, `dup/s` — logged once a second, plus a `[3D][FAULT]` log line
if the camera is writing but the eye is somehow not advancing (would mean
the output has silently gone mono; with the single-owner design this
shouldn't be reachable, but it's cheap insurance).

Separately, `opentrack_receiver.cpp` picked up a reliability fix RDR2's
build made to the same file: `SO_REUSEADDR` is no longer set on the UDP
socket (it let a stray leftover socket steal packets from the real
receiver — the "one packet then nothing" symptom), and the receive loop now
has a heartbeat log and a watchdog that rebuilds the socket if it hard-errors
for more than 3 seconds.

## What I didn't port

RDR2-3D-6DOF is a considerably larger project than this DSR port: a VR
full-resolution shared-surface bridge (`stereo_vr_dx12.h`, D3D11/D3D12
interop via a Katanga-style shared texture), independent FOV and
camera-dolly layers poked at per-engine struct offsets, and depth-based
reprojection groundwork. None of that was part of what this DSR port set out
to do, and FOV/Z-offset support in particular would need its own
reverse-engineering pass (a camera-struct FOV field offset specific to this
game, which nobody has found yet) — so I left those out rather than bolt on
unverified, unrelated functionality. If you want any of them, happy to do
that as a separate, scoped pass.

## How it works

Same architecture as the GotG stereo port:

1. The existing camera hook nudges the camera sideways by half the eye
   separation, alternating left/right every frame the hook fires (AFR) — no
   second render pass, no touched draw calls.
2. At Present time, the back buffer is copied into one of two persistent
   textures (`slot[0]`=LEFT, `slot[1]`=RIGHT).
3. A fullscreen pixel shader (unchanged from the DX11 mod — plain HLSL, not
   API-specific) composites both slots into your chosen layout and writes it
   back into the back buffer before the real `Present()`.

**What's different from the GotG port**, specific to this game's camera:

- Dead Space's hook gives you a **VIEW matrix** (world→camera), not a
  world/camera matrix — so translation is negated relative to the eye
  (`sgn = ViewMatrix ? -1 : 1`, already used by the existing head-lean code;
  the new eye-separation offset reuses the same sign).
- The convergence toe-in and eye-separation direction are computed from the
  **final head-tracked basis** (`nr`/`nu`/`nf` — right/up/forward *after*
  your head rotation is applied), not the game's raw clean camera, so the
  stereo axis follows wherever you're currently looking, consistent with how
  the GotG port did it.
- **Behavior change from the original head-tracking mod:** `Apply()` used to
  bail out entirely if no OpenTrack data was flowing, which would have
  silently disabled stereo 3D too whenever no tracker was connected (stereo
  and head tracking share this one hook). I changed that one `return` into a
  "hold a neutral head pose" fallback, so stereo 3D now works with or without
  a tracker plugged in. Head-tracking behavior itself when OpenTrack *is*
  connected is unchanged.

## What's genuinely unverified

**I still have no Windows/DirectX/GPU environment, so none of this has been
compiled or run** — this revision is, like the first, a careful structural
port rather than a verified build.

**The command-queue capture** no longer has the race-condition risk the
first revision carried (see "Fixes applied in this revision" above), but
capturing the queue by hooking `ExecuteCommandLists` still assumes the game
submits its scene through a `D3D12_COMMAND_LIST_TYPE_DIRECT` queue, which is
overwhelmingly the common case but not guaranteed for every engine.
`HeadTracking.log` will say `[3D] captured DIRECT command queue 0x...` if
this worked. If you don't see that line, or see
`[3D] dummy swapchain failed hr=...`, send the log.

**The engine's own hook fragility.** Your original README already flagged
this: *"Dead Space Remake uses a Frostbite-derived engine and may be
sensitive to hooking... if it's unstable, send the log."* That caveat applies
just as much here — the stereo compositor rides on the exact same camera
capture site, so anything that destabilizes head tracking destabilizes
stereo too.

**The world-unit scale.** GotG's stereo port had an empirically-documented
10-units-per-metre constant to build on (from that mod's own reverse
engineering notes). No equivalent was available for Dead Space. I set
`WorldUnitsPerMetre=1.0` as a starting guess, reasoned from this game's own
`LeanScale*` values (~1.5–2.5 applied directly to metres of head travel,
which only makes sense if world units are already roughly metre-scale) — but
this has not been checked against the running game. If 63mm separation looks
comically large or is imperceptible, this is the first thing to adjust.

**Everything else** — resource barriers, the composite shader, root
signature/PSO setup — carries over from the first revision, which was itself
a from-scratch, structurally faithful port of the proven DX11 GBFR
architecture.

## Build

**No compiled `.asi` is included in this package.** I have no Windows,
DirectX toolchain, or network access in this environment to build or fetch
one — you'll need to build it yourself. `build.sh` is included and mirrors
RDR2-3D-6DOF's own build line (mingw-w64), which is simpler than the MSVC
route I originally suggested:

```bash
./build.sh
# equivalent to:
x86_64-w64-mingw32-g++-win32 -O2 -std=c++17 -shared -static -static-libgcc -static-libstdc++ \
  -o DeadSpaceRemake-HeadTracking.asi src/dllmain.cpp src/opentrack_receiver.cpp src/stereo_dx12.cpp \
  -lws2_32 -lpsapi -ld3d12 -ldxgi -ld3dcompiler -luuid -lwinmm
```

This needs a mingw-w64 whose bundled headers include `d3d12.h`/`dxgi1_4.h`/
`d3dcompiler.h` (most reasonably current mingw-w64 builds do). If yours is
older and lacks them, or you just prefer MSVC, this also works from an
"x64 Native Tools" prompt:

```bat
cl /O2 /EHsc /std:c++17 /LD src\dllmain.cpp src\opentrack_receiver.cpp src\stereo_dx12.cpp ^
   /link d3d12.lib dxgi.lib d3dcompiler.lib ws2_32.lib psapi.lib winmm.lib ^
   /OUT:DeadSpaceRemake-HeadTracking.asi
```

Drop the resulting `.asi` next to the game's exe, alongside the `dinput8.dll`
(Ultimate ASI Loader) and `HeadTracking.ini` you already have installed —
those two are unchanged, only the `.asi` and `.ini` need replacing.

**Note on `logger.h` / `opentrack_receiver.h` / `opentrack_receiver.cpp`:**
your originally uploaded `DeadSpaceRemake-HeadTracking.zip` only contained
`dllmain.cpp` under `src/` — these three support files weren't included, so
I bundled versions from the same author's toolkit (identical `P5HT`
namespace and `OpenTrackReceiver`/`Pose` API is referenced by this
`dllmain.cpp` too). `opentrack_receiver.cpp` specifically now carries the
reliability fix described above (from RDR2's later build) — if you have your
own local copy of that one file, I'd suggest taking this version over it;
the other two are unmodified either way.

## Calibration order

1. Confirm the log shows the command queue captured
   (`[3D] captured DIRECT command queue ...`) and the pipeline built.
2. If you use a tracker, get head tracking feeling right first (unchanged
   from before). If you don't, stereo will run with a neutral head pose.
3. `Ctrl+F3`/`F4` — separation (3D strength). `Ctrl+F5`/`F6` — convergence
   distance.
4. `Ctrl+F11` cycles output mode; `Ctrl+F12` swaps eyes if depth looks
   inverted.
5. Ghosting on fast pans is normal AFR crosstalk (the two eyes are one frame
   apart in time) — try `AfrReproject=1` in `[Stereo3D]` if it bothers you.

## Files

- `src/dllmain.cpp` — your original head-tracking hook, extended with the
  stereo eye-offset/toe-in math and the neutral-pose fallback described
  above. `Stereo_Init()` is now called at ordinary worker-thread timing (no
  more early `DllMain`-time hook install — see "Fixes applied" above).
- `src/stereo_dx12.h` / `src/stereo_dx12.cpp` — the DX12 AFR compositor,
  rewritten in this revision around RDR2-3D-6DOF's dummy-device hook capture
  and single-owner present-parity eye assignment.
- `src/shaders_stereo3d.h` — the compositor's HLSL, unchanged (plain shader
  code, no D3D-version or game dependency).
- `src/stereo_config.h` — `[Stereo3D]` ini loading + hotkeys, with defaults
  retuned for this game's (unverified) unit scale, plus the new
  `AfrPipelineDepth` setting.
- `src/opentrack_receiver.*`, `src/logger.h` — bundled from the same
  author's other mods since they weren't in your zip (see the build-section
  note); `opentrack_receiver.cpp` carries the reliability fix from this
  revision.
- `HeadTracking.ini` — your existing settings plus the new `[Stereo3D]`
  section (now including `AfrPipelineDepth`).
- `build.sh` — mingw-w64 build command, matching RDR2-3D-6DOF's toolchain
  (see "Build" above for the MSVC alternative).
