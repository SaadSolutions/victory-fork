# Duplicated memory between title memory, the PC's RAM and the card

A plan, not a change: no code in this tree is changed by this note. Every
number below is from this fork's own logs (the scrubbed diagnostics run in
`diagnostics/timing-run-2026-11-25.txt`, a short diagnostics build in the
first rooms, about 3.4 frames a second, and the README's runs). A
diagnostics build is slower than a release build, so its per-frame figures
are high. Treat them as proportions, not as release-build frame times.

## The premise, checked

"The console has one memory, so the PC keeps everything twice" is about
half right. On the console the title, the CPU and the GPU share one copy.
Here the title's own memory (RAM) is always the reference copy: the title's
x86 code reads and writes it directly and the layer cannot take that away.
Everything the GPU uses is a second copy, and some things have a third:

| Data | Title memory (RAM) | Host copy | Third copy | Alive at the log's last memory line |
|---|---|---|---|---|
| Memory outside buffer objects ("windows": vertices, indices, constants from the title's allocator) | yes | 4 MiB system-memory buffer per window (write-back, L0) | none (or a card buffer with `VICTORY_CARD_WINDOWS_MB`) | 690 windows, 2,892 MB |
| Title buffer objects | yes | card buffer (VRAM) | its CPU side, `Card::staging`, the same size in RAM | made so far: 3,567 on the card, 1,201 MB (the log does not count those alive) |
| Shader-written ranges | yes | | 16 bytes of hash per 64 KiB on the card (not a full twin, which was dropped already) | small |
| Textures | yes (tiled) | host texture (VRAM) | transient upload staging | 10,699 host textures, 4,917 MB; 16,707 title textures with no host copy yet (9,732 MB of title memory) |

At that line: 22,864 MB private (18,668 MB resident), 725 MB of RAM free,
card 8,537 of a 10,236 MB budget, 4,812 MB of RAM shared with the card.
(Task Manager's "GPU memory" adds dedicated and shared memory together. That
is how a 12 GB card can show "20 GB".)

So the duplication that costs real RAM is: windows (about 2.9 GB), the CPU
sides of card copies (up to about 1.2 GB), and the windows' zeroed tails
(below).
The duplication that costs time is **the traffic that keeps the copies in
step**, not the copies themselves:

- title memory → host copies: 6,825 copies, **201.5 MB a frame**
  (`plaincopy` line), work that falls mostly on the submit thread, one of
  the two walls;
- card → title memory: 24.8 fetch jobs, **65.7 MB a frame** (`queue2`
  line), merged byte by byte on the CPU. In run 277 (49 min) that was
  497 GB, and of the stores sampled **at most 0.19 % could have been read
  by the title's own code** (60 % untouched until the next store, 36 %
  touched only by the layer's own refreshes, 3.8 % written by the title).

The ideas below are ranked by expected gain against risk. Several of them
already exist in this tree behind switches that have never been measured in
a game. Measuring those costs nothing to build and comes first.

---

## 1. Copy-back on demand (the largest new idea)

**What:** stop writing what shaders wrote back into title memory after
every submission. Leave it on the card (or in the fetched landing buffer)
and bring it back only when something on the CPU actually touches those
pages.

**Mechanism:**

1. After a submission, the card's hash compare already tells which 64 KiB
   pieces shaders changed (`Host::card_back`). Today those pieces are
   fetched and merged at once (`Shadow::card_merge_relay`).
2. Instead, mark each changed piece **card-newer**: protect its title pages
   `PAGE_NOACCESS` at every view that shows them. `watch.rs` already guards
   every view of a physical page (`protect_alias`, with the views from
   `inplace::pages_of`) and installs a vectored handler. Here it would
   be a no-access guard instead of a read-only one.
3. On a fault in a card-newer page (title read or write): fetch that piece
   alone (wait for the submission that wrote it, copy it on the copy queue),
   merge it as today (`merge_piece`, same byte rules), restore the page's
   normal guard, and return `EXCEPTION_CONTINUE_EXECUTION`.
4. The layer's own readers check the card-newer set first:
   - a refresh of the **same** copy skips card-newer pieces, because the
     copy already holds the newest bytes. That removes the 36 % "touched
     only by our refreshes" case outright.
   - a refresh of **another** copy over the same memory (overlapping
     buffers, a window over a buffer, a texture made from that memory,
     `VICTORY_TEXTURE_BACK`) forces the fetch first.
5. A piece stored again while still card-newer costs nothing. That is the
   60 % case.

**Hazards (most have a starting point in this tree already):**

- A kernel call that reads or writes a no-access page fails instead of
  faulting into the handler. Title file reads into guarded memory already
  go through a "read beside" path (`victory-hle` `files.rs`), which is
  where to fetch first. Kernel calls that *read from* such a page (writing
  a file from it, for instance) never needed a hook under read-only
  guards. They would now, so every title import that hands a buffer to
  the host has to be checked.
- DirectStorage and `xmem::write_physical` write through views of the
  layer's own, which a guard on title views does not see. They already
  report through `watch::writing`, which must fetch-and-merge (or drop the
  card-newer mark when the whole piece is overwritten) before writing.
- Private memory (`XMemVirtualAlloc`) is write-watched, not guarded.
  `VirtualProtect` works there too, but a protected page cannot be in a
  heap opened in place (idea 2: error 87). Keep the two apart: card-newer
  only for copies that are not in place.
- Ordering with the relay (`VICTORY_RELAY`, `Flights`): a fetch on fault
  must see every send queued before it. The relay's send numbers already
  give that order.
- The title's frame pacing reads a fence value with `mwaitx`. That memory
  is never shader-written, so it never becomes card-newer. Keep an
  allow-list check anyway.

**Expected:** most of the 65.7 MB a frame of fetches and merges gone, and
with them the refreshes they set off. That work lands on the copy queue
and the copy-back thread, and through refreshes on the submit thread. The
price is one fault and one small synchronous fetch for the rare piece the
title does read (≤ 0.19 % of stores in run 277).

**How to land it safely:** a switch (`VICTORY_BACK_ON_DEMAND`), first in an
`alt` form (twenty seconds on, twenty off, as the other switches do), with
a checker mode that still fetches eagerly and compares the eager bytes with
what the on-demand path would have produced at each fault. The same
pattern proved `VICTORY_INDIRECT_WINDOW_CHECK` and `VICTORY_BACK_OWN_CHECK`.

## 2. Windows in place: measure `VICTORY_INPLACE=1` (exists, never measured)

**What:** a window of private title memory is not copied at all. The
title's own pages are opened as a host heap (`OpenExistingHeapFromAddress1`)
and a buffer is placed over them (`inplace.rs`, `shadow::loose`).

**Why the README's objection does not settle it for windows:** the README
holds that opening pins title pages "in a PC with 1 to 5 GB free". But a
window's host copy is itself a resident system-memory buffer of the same
4 MiB, so it is locked too while resident. For a window in place, pinning
the title's pages replaces pinning a second copy. RAM use falls, it does
not rise. The one real difference: pages the title committed but never
touched become resident. With 1,645 of 2,892 MB of window pages ever
copied, that is bounded by what the copies already hold.

**Expected:** up to 2.9 GB less memory shared with the card, and the
window part of the 201.5 MB a frame of refresh copies gone. Only windows
in private memory qualify: view memory (the page pool) is turned down, and
the `title memory in place` line every ten seconds says how much was
taken.

**Risk to watch:** while a window is in place the title cannot
`VirtualFree`/`VirtualProtect` those pages (error 87). None has been seen
(the title keeps its heaps), but a run must grep for it. Do not combine
level 2 with `VICTORY_LATE_WAIT` (see `inplace::level`).

## 3. Card copies that no shader writes, in GPU upload heaps (new code, small)

The test PC has GPU upload heaps (resizable BAR): `supported=true` at line
55 of the diagnostics log. Windows can already live there
(`VICTORY_WINDOW_UPLOAD=1`, run 273: memory shared with the card 5.7 → 3.8
GB, free RAM 1.1–1.9 → 3.2–3.6 GB, same frame rate in one run).

Title buffer copies on the card still have two sides: a VRAM buffer and a
same-sized `Card::staging` in RAM (1,201 MB of card copies made by that
line). For a copy **no
shader has declared a write to**, the staging is only a stop on the way,
exactly as it was for windows. Make such copies one GPU-upload-heap
buffer: the CPU writes title bytes straight into VRAM, the send copies
leave the "send-and-hash" lists, and the RAM side is gone.

The catch: a write declared later (`Shadow::gpu_writes`) needs the byte
merge's "held" bytes, and reading a write-combined BAR mapping back is
slow. So on the first declared write, convert the copy once: make the
staging and a card buffer, copy over, and switch the `Shadow` to the
two-sided form. Count conversions in the log. If buffers convert in the
first frames anyway, the idea is not worth it and the count shows it.

## 4. Windows only as long as what is read (existing switches)

Of 2,892 MB of windows, **1,645 MB were ever copied** and 1,453 MB lie
between the lowest and highest address ever asked for in each (log lines
18626–18631). The rest is host memory that was zeroed and never read,
about 1.2 GB. `VICTORY_WINDOW_KB=1024` (a finer grid for asks with a
length) and `VICTORY_WINDOW_TAILS=1` (asks without a length start at their
256 KiB step instead of the 4 MiB line) were written for exactly this and
are off because no run has measured them. Measure them with the memory
line's own counters: "of their pages ever copied", "asked for without a
length", and "lie over N MB of title memory".

## 5. One heap for the whole page pool (larger, for view memory)

View memory cannot be opened in place today because
`XMemAllocatePhysicalPages` makes its section with `SEC_RESERVE`, and
Direct3D only opens `SEC_COMMIT` sections. Per `inplace.rs`'s own probe,
`OpenExistingHeapFromFileMapping` **locks nothing** (256 MiB: 0.2 ms, no
working set) and survives views being unmapped and remapped, because the
heap is the section, not a view.

- Make the section `SEC_COMMIT` first, falling back to `SEC_RESERVE` on
  failure. (The comment there says the title's 6.1 GiB ask "often cannot
  commit" on a busy host. The log shows about 50 GB left to commit on the
  test PC, so try it and keep the fallback.) Under Wine the section is
  already `SEC_COMMIT` (and `commit()` returns at once there), so the
  runtime has run on committed sections before, though not on Windows.
- Open each pool section once. A title buffer whose pages are one run of
  the section (`xmem::runs` already computes runs) is a buffer placed at
  that run's offset, with no copy, no refresh and no copy-back.
  Scattered buffers keep today's copies.
- Do **not** use tile mappings into such a heap. `inplace.rs` records that
  this removes the device.
- First step, no risk: run the census (`VICTORY_INPLACE_PROBE=1`) to learn
  how many bytes of title buffers are one run, and whether the 174 MB heap
  buffer at `0xeee0000` (the one that carries the most traffic) is.

## 6. Textures: fewer remakes, not fewer copies

A texture must be in VRAM and the title keeps its own bytes, so two copies
are unavoidable. What can shrink is the churn: 16,145 host copies let go
(3,882 MB) and 1,215 made again (326 MB) by that memory line. In longer
runs, 5.1 to 6.9 GB were made again in runs without card windows. Ideas:

- `VICTORY_LEVELS_ON_DEMAND=1` (exists): 3.1 GB not allocated for levels
  the title does not hold.
- Make the 1.2 GB of window and staging space freed by ideas 2 to 4 visible
  to `resource::room`, so it goes to textures. The README's runs 271 and
  272 already show that fewer textures let go means fewer long frames (21–26 %
  → 4–6 % of the last stretch's time).
- Not recommended here: `Evict`/`SetResidencyPriority` instead of
  destroying. That moves pages into RAM, and RAM is the scarcer of the two
  on this PC (725 MB free).

## What will not help (already measured)

- Comparing before copying to send less (`VICTORY_COPY_CHANGED=1`): the
  card's sync lists fell 12 → 8 ms but the submit thread paid 3.6 ms, so
  no net gain.
- Refreshing on more threads (`VICTORY_WIDE_REFRESH`): no gain in runs 268
  and 269.
- A full "before" twin on the card for shader-write detection: already
  replaced by 16 bytes of hash per 64 KiB piece (`card_hash_bytes`).

## Order of work

1. **Measurements only, no code:** one run each, scene held still and then
   moving, with the memory line and the frame line, against the run 253
   switch set:
   - `VICTORY_INPLACE=1`
   - `VICTORY_WINDOW_UPLOAD=1 VICTORY_CARD_WINDOWS_MB=2048`
   - `VICTORY_WINDOW_KB=1024 VICTORY_WINDOW_TAILS=1`
   - `VICTORY_INPLACE_PROBE=1` (census only)

   Report: private and resident MB, free RAM, shared-with-card MB, card in
   use, the `plaincopy` MB a frame, the submit thread's "copies" ms,
   textures made again, and any error 87.
2. Idea 3 (small code, ReBAR PCs only, off by default).
3. Idea 1 behind `VICTORY_BACK_ON_DEMAND`, with its checker mode first.
4. Idea 5 if the census shows enough one-run buffers.
