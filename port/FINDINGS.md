# Findings

Measurements from unattended trials of the GTA VI Xbox build under Proton with VIctory, on the
machine described in the README (2026-11-25, sections 6-8 added the same afternoon). Each one cost us trials to separate from noise;
they may save another port the same work.

## 1. The cell wall is missing at random per launch: stale copies of aliased pages

In the fixed shot after the opening cutscene (Lucia at her cell window, camera still), the wall
left of the window is drawn or absent, all or nothing per launch, with the same executable.
Base rate on our line: drawn in about 1 launch out of 4.

- Fork PR #571 (DirectStorage page-destination writes now notify the GPU side) did not change
  that rate here: 1 launch out of 4.
- Adding the upstream fix for aliased pages (Marvisak/VIctory #1122, `c75fdba`: a write through
  one mapping of title pages reaches every mapping of them) drew the wall in 3 valid launches
  out of 3. By chance alone that is 1 in 64.
- A diagnostic mode that compares every tracked host copy with title memory still reports about
  2 differing ranges per frame after both fixes, all on buffers. Whether they are missed writes or
  GPU results not yet merged back into title memory is still open.

Metric: `qualite.py`'s `mur()` reads the region left of the window in that shot. A drawn wall is
dark (mean luma under 80); without it the sky shows (150-200). It only works for this framing.

Lesson: one launch per side proves nothing for a per-launch bug. We count a fix as proven after
three good launches in a row and refuted after two bad ones.

## 2. Clothing without colour: packed textures marked stale

In the fixed shot, Lucia's jumpsuit came out black, as a wrong pattern or as coloured blocks in
every earlier launch we checked (over 20), never orange as in the cutscene.
fxceless/victory-fork `310c6eb6` changes two things: a newer texture view makes an older one stale only when it starts at the same address (packed small mips of a
neighbour no longer do), and a texture already on the card is refreshed even when the card is
full. With those two parts ported, the orange "INMATE" jumpsuit, the poster and the inmates in
the yard are drawn in 2 valid launches out of 2.

## 3. Speed: copy-back waits on GPU fences, not on PCIe

About 85 MB a frame come back from the card in about 200 ms: 3-4 % of what PCIe 4.0 x8 carries.
The time goes into about 7.5 fence round trips a frame (one copy list, one signal, one blocking
wait each), about 18 ms apiece behind queued work. A wider link would not help; fewer waits
would. Fork PR #648 and upstream PR #1124 attack exactly this.

## 4. Speed noise: compare on the still shot only

Two identical warm trials differed by +23 ms/frame [+3, +48] over the whole run: the cutscene
drifts along the clock from one launch to the next, so paired samples do not show the same
scene. On the fixed shot alone the same pair differs by +9 ms [-11, +33], which contains 0.
Compare speed on the still shot, with two independent pairs.

## 5. Xid 31 one second after our own kill is a teardown artifact

NVIDIA Xid 31 (GPU MMU fault) appeared about one second after our own `pkill -KILL` at the end
of a trial, never during one. `stats.py` splits kernel Xid lines into "during the trial" and
"within 5 s after our own stop"; only the first group counts.

## 6. Where a frame goes: the title is light, the layer is not

Profile of one trial on the fid9 line (369 ms a frame on the still shot), 25 s of samples, and a
replay of the per-second accounts of nine trials.

- The title's own code runs about 104 ms of CPU a frame over its 16 worker threads, the same at
  209 and at 369 ms a frame. No title thread runs more than 16 ms a frame. The submitting thread
  runs under 4 ms of title code a frame; the rest of its time is in the layer.
- The submitting thread binds: about 223 ms of serial work and 146 ms of waits a frame. The serial
  work: page guards of view memory 57 ms (about 470 faults a frame, about 120 us each, through
  Wine's signal path), other write tracking 13 ms, the asking and the look under flights 25 ms,
  bytes moved into host copies 21 ms, descriptor sync 18 ms, uploads 14 ms, textures 13 ms.
- One wait for flights is the card finishing what is in the air, not the fetch: wait = 5.5 x (card
  busy time / submissions) + 0.75 x one fetch wait, fitted on nine trials within 3 ms a wait.
- The card is busy about 212 ms a frame with work of ours queued; Series S resolution takes 55 ms
  of it off, so resolution explains about a quarter.
- The trial pays for its own proof: PNG captures cost about 9 ms a frame on the submitting thread.

## 7. Switches measured with no effect here

RTX 3090, Linux, Proton, still shot against the same family (310-350 ms for fid5, 369-377 ms for
fid9), margin about 20 ms:

- `VICTORY_ASYNC_SUBMIT=1 VICTORY_QUEUE2=1` (fxceless `f85101b6`): 359 ms. The copy queue
  received no work after the first second (0 jobs a frame, relay on). Compare fxceless issue #873,
  where the same switches gained 15 % on an RTX 4070 but lost the window's wire mesh.
- `VICTORY_INDIRECT_WINDOW=1`: 356 and 381 ms. Bounded indirect commands went from about 5,150 to
  about 2,550 a frame, with no change on the still shot.
- `VICTORY_CARD_FETCH_INFLIGHT=1`: about 935 pieces a frame merged inside flights, 367 ms.
- `VICTORY_LATE_COPY_AFTER=1` (port of `5851154f`) on fid5: 318 ms. The copy wait moves, it does
  not go away.
- RT-miss fallback light (felicienradio): no visible change on the still shot (mean luma 61.7
  against 61.4).
- Under Wine, `K32EmptyWorkingSet` is a stub and `FILE_FLAG_NO_BUFFERING` never reaches `open()`:
  a working-set relief valve and unbuffered DirectStorage reads measured on Windows do nothing here
  (wine `dlls/kernelbase/process.c`, `server/fd.c`, `dlls/ntdll/unix/file.c`).

## 8. Series S mode: 55-60 ms faster at 1080p

`CONSOLE=series-s` on a development branch (fid5 plus four commits for the second present plane
and two render target formats, not published yet): 265, 275 and 268 ms on the still shot in three launches, the
wall drawn in all three, against 310-350 ms in Series X mode. Output is 1080p instead of 4K. Two
defects so far: about one presented frame in sixteen before the world is an empty black layer
(fixed on our side by keeping the last image on screen, not published yet), and Lucia's skin
sometimes turns black in the world (open).

## Reproducing

- Wall and clothing: three or more launches with `VICTORY_CAPTURE_WORLD=1`, then
  `run/qualite.py --fin <trial>` and look at `plan-fixe.jpg`.
- Speed: two identical warm trials, then `run/stats.py <a> <b>` (it prints the still-shot pair).
- Xid: keep `kernel.txt` per trial; `run/stats.py` prints the two groups separately.
