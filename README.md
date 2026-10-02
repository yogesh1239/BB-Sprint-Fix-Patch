# BB Sprint Fix Patch

A one-instruction patch for **Bloodborne v1.09** that stops **sprinting from dropping to half
speed at high frame rates**.

If you play with an unlocked frame rate (the community "Uncap FPS++" or "90 FPS++" patches),
you may have noticed that sprinting (holding Circle while moving) sometimes feels sluggish:
the character starts to sprint, then gets stuck at about half of the normal sprint speed.

- Above about **147 FPS**, it happens on almost every sprint.
- Between about **90 and 147 FPS**, it happens at random, more often on long sprints.
- Normal running (no Circle) is not affected.

This patch fixes the cause in the game code. It is not a workaround.

## Download

| File | For |
|---|---|
| [`patches/Bloodborne_Sprint_Fix.xml`](patches/Bloodborne_Sprint_Fix.xml) | Emulators and loaders that use the GoldHEN / shadPS4 XML patch format |
| [`bbport/bbport-sprint-fix.patch`](bbport/bbport-sprint-fix.patch) | The [bbport](https://github.com/deadinside28/bloodborne_pc) native Linux port (a `git` patch) |

## Install

### shadPS4 (or any tool that reads GoldHEN/shadPS4 XML patches)

1. Open your Bloodborne patch file in the emulator's patches folder.
2. Copy the `<Metadata Name="Sprint Fix (High FPS)" ...> ... </Metadata>` block from
   `patches/Bloodborne_Sprint_Fix.xml` into it, next to the other `<Metadata>` blocks.
3. Enable **Sprint Fix (High FPS)** together with **Uncap FPS++** (or **90 FPS++**).

Or load `Bloodborne_Sprint_Fix.xml` as its own patch file if your tool supports more than one.

### bbport

From the root of your `bloodborne_pc` checkout:

```sh
git am /path/to/bbport-sprint-fix.patch
```

This adds the fix to the "Uncap FPS++" and "90 FPS++" entries in `patches/Bloodborne.xml`.
The port compiles the patches at every start, so nothing else is needed.

## Compatibility

- Game version **01.09** only (`eboot.bin`). Other versions have different code addresses.
- The title IDs in the file are the same ones the community FPS patches use (these share the
  same 1.09 code layout). **Tested only on CUSA03173 (EU, GOTY), v1.09**, on bbport.
- Not tested on shadPS4 or on a real PS4. On a PS4 the frame rate never gets high enough for
  the bug to happen, so the patch is not needed there.
- No other entry in the community `Bloodborne.xml` patch file writes to this address, so it
  does not clash with them.

## Results

Measured on bbport, CUSA03173 v1.09, with a scripted controller (straight sprints on the same
spot, top speed in game units per second):

| Frame rate target | Without the fix | With the fix |
|---|---|---|
| 60 | about 6.2 | 6.87 |
| 120 | about 6.3 | 6.89 |
| 144 | about 6.3 | not measured |
| 200 | **about 3.2** (broken) | 6.95 |
| 240 | **about 3.2** (broken) | 6.73 |

Normal running stayed at 4.0 units/s with and without the fix. Pressing into a wall still
slows the character down, as in the original game.

Note: the test machine reached only about 170 real FPS with a 200 or 240 target.

## What was wrong

The game has a **wall detector** in its per-frame movement code. Its job: if you sprint into
a wall, slow the character down instead of letting the legs run in place at full speed.

It decides "I am blocked by a wall" with this rule:

> distance moved **in this frame** × 30 < 1.0

That is "moved less than 1/30 of a unit in one frame". At the original **30 FPS**, one frame is
1/30 s, so the rule means "slower than **1 unit/s**" — a sensible "I am stuck" test.

But the rule counts distance per frame, not speed. At higher frame rates each frame is shorter,
so the character moves less per frame:

| Frame rate | The same rule means "blocked when slower than" |
|---|---|
| 30 FPS | 1 unit/s |
| 60 FPS | 2 units/s |
| 150 FPS | 5 units/s |
| 240 FPS | 8 units/s |

A sprint starts at about 4.5 to 5 units/s and then speeds up to about 6.4. At high frame
rates, that start is below the limit, so the game thinks you ran into a wall:

1. It multiplies the movement by 0.8 each frame the "blocked" test is true, down to 0.5.
2. The character is now slower, so it moves even less per frame.
3. The test stays true, and the sprint **locks at half speed** until you stop.

Between 90 and 147 FPS, the sprint is usually fast enough to pass the test. But a single short
frame at the wrong moment, during the first moment of the sprint, can start the same lock.
That is why the slowdown looks random there.

This bug is in the original game. The community frame-rate patches do not cause it; they only
make it visible, because the game was never meant to run this fast.

## The fix

Divide the distance by the frame time instead of multiplying it by 30. The test becomes:

> distance moved in this frame ÷ frame time < 1.0, which means "slower than 1 unit/s"

This is exactly the original 30 FPS rule, now at every frame rate.

One small difference remains: at 60 FPS the original game used "slower than 2 units/s" as the
wall limit (see the table). With the fix it is 1 unit/s at every frame rate. In testing, walls
still slow the character down as before.

## Not tested yet

- Stairs, ladders, moving platforms and other moving objects.
- Sprinting while locked on to an enemy.
- The random slowdowns at 90–120 FPS were not reproduced on purpose. They fit the same cause,
  but this part is a likely explanation, not a measured one.

Reports are welcome in the [issues](../../issues).

---

## Technical details

All addresses are PS4 addresses with the `eboot.bin` base at `0x400000`, the same convention as
the community patch files.

**Function `0x1914500`**: the per-frame movement update. `xmm0` is the frame dt in seconds and is
stored once at `[rbp-0x74]` (`0x191454f`); nothing else writes that slot in the function.
`r13` is the movement controller and `r14` the physics object.

The wall detector:

```
; gate: planned speed r13+0x1e4 (= |move| / dt, written at 0x19146b3) > 5.0 and flag [r14+0x212]
; xmm0 = |[r14+0x1e0] - [r14+0x1f0]|            ; distance moved this frame
0x1914c24  vmulss   xmm0, xmm0, [0x4d2632c]     ; * 30.0      <-- patched
0x1914c2c  vmovss   xmm1, [0x4d26318]           ; 1.0
0x1914c34  vucomiss xmm1, xmm0
0x1914c38  jbe      0x1914c60                   ; not blocked: scale *= 1.2 (0x4d26338), max 1.0
0x1914c3a  ...                                  ; blocked:     scale *= 0.8 (0x4d26334), min 0.5 (0x4d26330)
0x1914c55  vmovss   [r13+0x1e0], xmm0           ; slowdown scale
```

The scale `r13+0x1e0` multiplies the whole movement vector at `0x19146c5`, when it is below 1.0.

**Patch** at `0x1914c24` (8 bytes):

| | Bytes | Instruction |
|---|---|---|
| Original | `c5 fa 59 05 00 17 41 03` | `vmulss xmm0, xmm0, dword ptr [rip+0x3411700]` (30.0) |
| Patched | `c5 fa 5e 45 8c 0f 1f 00` | `vdivss xmm0, xmm0, dword ptr [rbp-0x74]` ; `nop dword ptr [rax]` |

You can check the original bytes in your own `eboot.bin` before you apply the patch. The
constant 30.0 at `0x4d2632c` has no other user, so it stays untouched.

If dt is 0, the division gives +infinity, which means "not blocked". That is harmless.

**How it was found**: live memory diffs of the running game during scripted sprints at 120 and
200 FPS showed values that were 1.0 during a good sprint and 0.5 during a bad one. A hardware
watchpoint on one of them led to `0x1e4a610` (a copy of the movement block), then to its
caller `0x1914500`, and a second watchpoint on `r13+0x1e0` led to the only writer of the
scale (`0x1914c55`). Reverting each part of "Uncap FPS++" one at a time did not change the
bug, which confirmed that the cause is in the original game logic.

## Credits

- Lance McDonald (manfightdragon) and Kyo for the 60 FPS and unlocked frame-rate patches.
- The bbport authors for the native port this was found and tested on.

## License

[MIT](LICENSE)
