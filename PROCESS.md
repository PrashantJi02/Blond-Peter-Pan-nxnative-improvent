# The whole process — Ocarina of Time (Switch 2) on a Mac: from a 10 fps market to 120 shown fps

Everything below was done on a base M5 MacBook Pro (16 GB), macOS 27.2, running the Ocarina of Time Switch 2 remake through [seedds/nxnative](https://github.com/seedds/nxnative). Two separate problems, two separate fixes. Nothing here is upstream.

## Step 0 — Getting the game running

nxnative is not an emulator. The game's own ARM64 code runs natively on Apple silicon, with a host layer (dmhost) standing in for the Switch's system services and NVN graphics translated to Metal, shaders translated ahead of time. The game (title `0400486024DEC000`, update v131072) builds from a single 25.6 GB `.nup` with no keys needed:

```bash
python3 scripts/build_ipa_from_nsp.py "<folder with the .nup>" --platform mac-app
```

First build: ~2 hours, 42,848 shaders translated. The result plays at a locked 60 fps through the opening areas — except one place.

## Step 1 — Finding the market problem properly

Hyrule Castle market ran at a sustained 10–16 fps while everything around it held 60. Before changing anything, the frame-time logging was switched on (`DM_SLOWFRAME=1` in the game's `env.txt`) and a slow session was captured and read:

- Market frames ran 62–102 ms, bound by the present thread's CPU time — GPU times were modest, sync time ~0, and zero shaders compiled during the slow frames. Not a GPU limit, not shader stutter.

The last untested documented lever in the project's frame-rate notes was the present thread's real-time policy: the host declares a CPU-time budget for the thread to macOS, auto-sized from recent frames (starts at 2 ms, re-sizes every 120 frames toward the measured peak, cap 15 ms). On this machine it had learned ~4.6 ms from the light scenes. The market needs far more per frame — so macOS kept demoting the thread for overrunning its own declared budget, and the heavier the scene, the harder the throttling bit. A feedback loop, not a lack of power.

## Step 2 — The one-line fix, and proving it

Add to `env.txt` (`~/Library/Application Support/nxnative/0400486024DEC000/env.txt`):

```
DM_RT=0
```

That turns the declaration off entirely. Result: the market went from 10–16 fps to a flat 60. A full reference session afterwards: 11,390 judged frames, 50 slow (0.44%), market frames 16.6–18.3 ms.

This was written up as [issue #1985](https://github.com/seedds/nxnative/issues/1985). The maintainer (lunks) confirmed the auto-sizing behaviour from his side — on his M5 Pro the budget climbs to the cap and the market holds 60, so his read was that the auto-sizing reacts too slowly on the base M5 — and asked for an A/B with a fixed maximum declaration:

```
DM_RT_COMP=15
```

Result of the A/B: market mostly 60 with brief stutters, the market-entrance-to-fountain run locked at 55 fps, and once ~15 fps idling at the market entrance after loading — none of which happened under `DM_RT=0`. Conclusion: `DM_RT=0` is the setting. The A/B results were posted back to the issue.

## Step 3 — The frame generation idea

The display is 120 Hz ProMotion and the port already presents on a 120 Hz display link (`DM_PRESENT_LINK=2`), while the game steps at 60 fps — half the display ticks show nothing new. Frame generation fills those ticks on the present side, without touching the game:

- Every real frame is copied into a two-frame history as it is taken for presentation.
- On the tick a frame arrives, the screen shows a 50/50 blend of the previous frame and this one.
- The real frame itself is shown on the next tick.

The safety detail: real frames land one tick (8.3 ms) later than stock. The game paces itself from the *intervals* between presented times, and a constant offset cancels in the differences — so the patch subtracts its own delay back out of the presented times it reports, and generated frames take no present ticket and write no presented-time entry. From the game's side, nothing changed. It still renders and simulates at exactly 60; the display shows 120.

MetalFX frame interpolation can't be used at this point in the pipeline: the present path only holds the final colour texture, with no depth, motion vectors, or jitter data. So generation here is a colour blend — stage 1, deliberately, to prove the scheduling and accounting first.

## Step 4 — Four versions, four lessons

**v1 — crash.** Engaged immediately at boot. The performance HUD showed 80 fps with generation blinking on alternate frames, then the game crashed seconds in, on the first button press. The log (kept from before relaunch — the log rotates, so preserve it first) showed zero GPU errors from the patch and a SIGSEGV inside Apple's Metal shader-compiler cache (`fscache_find_and_retain`, `MTLCompilerFSCache::getElementImpl`) on a shader-prefetch worker — the boot window's compile storm. Suspected trigger, not proven: the blend's one new pipeline compiling inside that storm. With the switch off, the same build ran fine.

**v2 — the blink fix and the gate.** Two changes: engage only after 600 consecutive steady frames (~10 s of clean 60 fps), moving the blend's pipeline build out of the boot window; and fix the cadence bug behind the blink — the held-frame branch had been consuming the tick counter the next frame's engagement check needed, so every second frame failed the check (blend, real, real — averaging exactly the 80 fps the HUD showed). Result: no crash. But generation then behaved like an fps unlock — 120 while standing still, back to 60 the moment anything moved. Idle frames are metronomic; gameplay jitter (a 17 ms frame landing 3 ticks after the previous one) tripped the gate instantly, and recovery demanded 600 fresh perfect frames, which gameplay never produces. Generation lived only on a static picture, where it changes nothing.

**v3 — ride through jitter.** Takes landing 2 or 3 ticks apart both generate; only 4+ ticks or a frame period over 25 ms counts as a break. Re-engagement after 30 steady frames (~0.5 s); the 600-frame wait applies only to the first engagement of the process, the one that pays for the pipeline build near boot.

**v4 — forgiveness and a watchdog (current).** An isolated slow frame while engaged is now shown as-is, unblended, and generation stays engaged — two are forgiven, a third inside a decay window is treated as a real break. A per-frame cost budget on the link thread (5 ms) and a check for the GPU still being busy with the previous generated frame both trigger a backoff whose re-engage wait doubles when engagements keep dying young (30 → 60 → 120…, cap 600; a long run resets it). The wait for the blend allocator, unbounded in v1–v3, is bounded at 2 ms. Every disengagement logs its reason.

## Step 5 — Reproduce it

```bash
cd ~/nxnative-build
git checkout -b framegen-experiment
git apply framegen-stage1-v4.patch
make -C dm/host GAME=0400486024DEC000
python3 scripts/build_ipa_from_nsp.py "<folder with your game dump>" --platform mac-app
```

(The `make` step is a compile check — warnings are expected, errors are not. Save the patch file to the machine before applying; the build reuses the shader cache unless the translation chain changed upstream.)

Final `env.txt`:

```
DM_PRESENT_LINK=2
DM_SLOWFRAME=1
DM_RT=0
DM_FRAMEGEN=1
```

Run frame generation windowed, or add `DM_SCREEN_MATCH=0` — fullscreen otherwise pins the display to a multiple of the game's rate. Verify in the log: `frame generation on (DM_FRAMEGEN)` at start, `frame generation engaged` after ~10 s of steady play, and the HUD holding ~120 *while moving*. Kill switch: delete the `DM_FRAMEGEN` line and relaunch — stock behaviour, no rebuild.

## What it is, honestly

120 displayed, 60 rendered. Motion has twice the steps; input response stays at the game's 60 fps level plus one tick (8.3 ms). Fast camera spins ghost — warping with motion data is the stage 2 that would fix that. It does nothing for a scene that can't hold 60; the market fix above is what restored real 60 there. The two stack; they are not the same trick.

*Found, measured, built and tested on a base M5 by Prashant ([@PrashantJi02](https://github.com/PrashantJi02)); the base project is [seedds/nxnative](https://github.com/seedds/nxnative), and the market diagnosis was confirmed by lunks on issue #1985. The patch was iterated across four versions with AI assistance in drafting the code; every build, test and measurement was run on the real machine.*
