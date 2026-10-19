# `libAppleEXR.dylib` — unbounded `dataWindow` extents

## Where the bug lives

```
/usr/lib/libAppleEXR.dylib
  DecompressedLocations::CreateDecompressedLocations(
      const TileSize&, axr_size_t, axr_size_t, axr_size_t)
  link-time address (x86_64 slice): 0x7ffa15917b34
  allocation site:                  +0xa8b80
```

Reached during `axr_data_create` → `AXRData::AXRData` →
`Part::InitOffsets` → `ChannelDescription::CreateMipLevels` →
`CreateDecompressedLocations`, which in turn is reached from
ImageIO's format dispatcher whenever a caller invokes
`CGImageSourceCreateWithData` plus any properties / image /
thumbnail accessor on EXR-magic bytes. A per-step AppKit reproducer (see `evidence/nsimage/`) on the
same file isolates the commit to `NSBitmapImageRep initWithData:`,
which allocates roughly +1.52 GiB by itself. `NSImage initWithData:`
on its own allocates only ~3 MiB. So the bug is reached through at
least two independent API surfaces: CGImageSource (with caching/
thumbnail flags) and NSBitmapImageRep.

## The arithmetic

The allocation size is computed in registers as:

```
alloc_size  =  arg2 * (arg3 >> 29 & ~7)
             + arg2 * 8
             + 24
```

`arg3` ultimately derives from the file's `dataWindow` via
intermediate frames in `Part::InitOffsets` and `CreateMipLevels`.

## What the PoC actually contains

The `dataWindow` box2i attribute in `pocs/exr.exr` is:

```
xMin = 0
yMin = 0x89ffd8ff      // signed int32 = -1,979,721,473
xMax = 0x000001ff      // 511
yMax = 0
```

So the file describes:

```
Width  = xMax - xMin + 1  =  512
Height = yMax - yMin + 1  =  0 - (-1,979,721,473) + 1
                          =  1,979,721,474 pixels
```

A nominal 512-pixel width and a derived height of about 1.98
billion pixels. This extent propagates through the intermediate
frames and reaches `CreateDecompressedLocations` as a large
`arg3`, which drives the right-shift product into the multi-GiB
range. In the captured runs the underlying request is approximately
15.8 GiB.

## What the existing checks catch, and what they miss

The disassembly excerpt in `exr-disassembly.txt` shows `jo` and
`jb` after each `mul`/`add`, and the caller `CreateMipLevels`
NULL-checks the returned pointer. Together these handle:

- complete `uint64_t` overflow of the multiplication
- complete carry of the `add`
- allocation returning NULL

What they do not handle: a value that is extremely large but
remains arithmetically valid (no `uint64_t` overflow). The
missing check is an earlier bound on attacker-controlled image
extents.

## Suggested fix

In `Part::InitOffsets` or earlier in `axr_data_create`, validate
the derived width/height and the final allocation size against
reasonable implementation-defined limits before the allocation
arithmetic — e.g. clamping derived per-dimension extents, or
capping the final `alloc_size`. The specific bounds are an
implementation decision; the relevant point is that there is
currently no such bound in this path.

## Measurement

From `evidence/measurements.txt`, three back-to-back runs in a
native arm64 process (`sysctl.proc_translated = 0`) with no
sanitizers:

```
Run 1/3:  resident +1,816,674,304  (1.69 GiB)
Run 2/3:  resident +1,982,070,784  (1.85 GiB)
Run 3/3:  resident +1,982,103,552  (1.85 GiB)
```

The first run's delta is lower than the next two — likely warm-cache
effects in the allocator and image-source plugin loader.
`/usr/bin/time -l` on a freshly forked process in the captured run
reports:

```
/usr/bin/time -l peak memory footprint:  1,983,367,160 B  (1.85 GiB)
```

Comparing this against the stable second/third `task_info` runs,
the two methods agree to within about 0.06%.

## qlmanage on the same input

`qlmanage -t -s 1024 exr.exr` does **not** complete on this PoC.
The process hangs with no progress for at least 20 seconds and
produces no thumbnail; the wrapper kills it after the budget.
The `/usr/bin/time -l` line for that hung process shows only
`peak memory footprint = 9,192,072` (≈ 8.8 MiB) — qlmanage never
reaches the inner decode path that drives the larger commit.

So for the qlmanage path, only the hang is claimed, not the memory
commit. The 1.85 GiB commit is observed through the direct
`CGImageSource` path, and the AppKit per-step evidence isolates
another high-commit site to `NSBitmapImageRep initWithData:`
(≈ +1.52 GiB).

The same `qlmanage` invocation on the control EXR completes in
about 1 s and produces a thumbnail with `peak memory footprint =
69,812,872` (~67 MiB).

## Legitimate EXR control

Same APIs, legitimate file from Xcode's `CoreRE.framework`
resources (`twinkle.exr`, 246,981 bytes):

```
task_info resident delta:        +5,177,344    (4.94 MiB)
/usr/bin/time -l peak:           6,652,432     (6.35 MiB)
qlmanage peak memory footprint:  69,812,872    (67 MiB, completes in 1 s)
```

A legitimate EXR file ~640× larger than the PoC uses ~400× less
resident memory through the same code path.

## PoC layout (384 bytes)

```
offset  bytes                       meaning
0x00    76 2f 31 01                 OpenEXR magic
0x04    02 00 00 00                 version 2
0x08+   "channels" \0 "chlist" \0   R, G, B, A as FLOAT
...     "compression" \0 …          compression = 0
0x82    "dataWindow" \0 "box2i" \0
0x93    10 00 00 00                 box2i attribute size = 16
0x97    00 00 00 00                 xMin = 0
0x9b    ff d8 ff 89                 yMin = 0x89ffd8ff  ← malicious
0x9f    ff 01 00 00                 xMax = 0x000001ff = 511
0xa3    00 00 00 00                 yMax = 0
...     rest of header is structurally well-formed
```

`yMin = 0x89ffd8ff` is a signed int32 of -1,979,721,473. The
derived image height `yMax - yMin + 1` becomes 1,979,721,474
pixels — about 1.98 billion rows of pixels — while width remains
a normal 512.

## Note on the disassembly excerpt

The disassembly in `exr-disassembly.txt` is from the x86_64 slice
of the shared cache, included as supplementary static evidence.
The primary evidence for this report is the runtime measurement
on native arm64; no arm64 disassembly excerpt is included.
