# Pinned host memory on Windows, with two cards

Measured 2026-11-12 on the system in [`../README.md`](../README.md): RTX 5090 (PCIe 5.0 x8) + RTX 3090 (4.0 x8), 192 GB
of RAM, Windows 11 (WDDM), CUDA 13.4, a 12 GB system-managed page file, so a commit limit of 201.6 GiB. Three small
CUDA programs did the measuring; the numbers are this box's.

Engines that keep most of a MoE's experts in RAM copy them to a card while reading a prompt and when their expert
cache changes. Whether the source is pinned decides the rate, and on Windows how much can be pinned is not a matter of
free RAM. This is what strata-mimo's prompt path ran into ([two-gpu.md](two-gpu.md)).

## Copy rates

13.4 MB copies (one MiMo expert), `cudaMemcpyAsync`, GB/s:

| Source | To the 5090 | To the 3090 |
|---|---|---|
| `cudaHostAlloc` memory | 28.8 | 13.3 |
| A read-only file mapping, registered with `cudaHostRegisterReadOnly` | 28.8 | 13.3 |
| The same mapping, not registered | 15.9 alone; 9-15 inside an engine | |
| `malloc` memory | 15.2 | 12.9 |

## What caps it

- **`cudaHostAlloc` stops at 146.6 GiB** with nothing else loaded: the 47th 3.2 GiB block is refused with 25 GiB of RAM
  still available. A second card's context doesn't raise it.
- **Pinned memory and VRAM are both charged to the commit limit** (RAM + page file). Each card's VRAM in use counts in
  full. With 52 GiB of VRAM in use and ~24 GiB committed at idle, 201.6 GiB leaves about 125 GiB to pin.
- **At the commit limit `cudaHostAlloc` doesn't fail, it waits.** With 51 GiB of VRAM held it pinned 127.5 GiB, then
  made no progress for ten minutes at 208.2 of 210.3 GiB committed while Windows grew the page file. An engine has to
  stop short of the limit itself (`GlobalMemoryStatusEx`, `ullAvailPageFile`); strata-mimo's local build leaves 14 GiB
  and pins 34 of MiMo's 47 expert layers, 108 GiB.
- **Pinned memory lowers the free VRAM both cards report**: 6.1-6.3 of 31.8 GiB "free" with 147 GiB pinned and nothing
  in VRAM. Anything sized from `cudaMemGetInfo` has to take its VRAM before the host pins.
- The page file's size therefore sets how much can be pinned. Nothing is written to it; the limit is bookkeeping.

## Pinning a file mapping in place

A GGUF is already mapped and, once read, sits in the file cache. `cudaHostRegister(p, n, cudaHostRegisterReadOnly |
cudaHostRegisterPortable)` on the read-only view (`PAGE_READONLY`, `FILE_MAP_READ`) lets the driver lock those pages:
nothing is copied, the model is in RAM once instead of twice, and copies run at the pinned rates above.

- **The registration is lazy.** 4 GiB register in under a millisecond; a page is locked when a copy first needs it.
- **It is charged to the commit limit all the same**, at that first copy: 147.2 GiB registered and copied once took
  145 GiB of commit.
- **One 150 GiB registration is refused** ("out of memory"). 46 registrations of 3.2 GiB pass and the 47th is refused,
  the same ~147 GiB as `cudaHostAlloc`.
- **With both cards holding VRAM (30 + 22 GiB), the driver locks 73.6 GiB and no more.** The copy that would take it
  to 76.8 GiB: `cudaMemcpyAsync` returns success, `cudaStreamSynchronize` returns "out of memory", with 48 GiB of commit
  and 98 GiB of RAM still free. After it the process can pin nothing: `cudaHostAlloc` is refused too. 73.6 is exactly
  half of the 147.2 GiB one context locked, which suggests that with the Portable flag each card's context locks the
  pages for itself. Registering for one context only wasn't tried.
- **The other card reads pages the first one locked** at its own rate (12.6-13.3 GB/s to the 3090), with no further
  commit.
- **A copy whose source crosses the border of a registered range fails** with "invalid argument": across two adjacent
  registrations, or from registered into unregistered pages. A GGUF's tensors lie back to back at an offset that isn't
  a multiple of a page, and registrations are whole pages that can't overlap. So register one tile per tensor (from the
  page its first byte is in, when the tensor before it ends there, to its last whole page) and cut any copy at the
  tile borders.

Outcome here: with two cards the pinned copy (108 GiB under the commit rule) covers more than pinning in place (73.6
GiB), so strata-mimo uses the copy and pays for it at the start (about a minute). In-place pinning stays in its build
as an opt-in, capped at 60 GiB.

## Related measurements

- **RAM traffic is shared.** CPU expert reads and DMA copies to a card run on the same memory bus: together about
  35 GB/s on this DDR5-3600, where the CPU lane alone sees 37-50 and the 5090's link alone 28.8. A prompt path that has
  the CPU compute some experts while the card copies others gains less than the sum.
- **Soft faults are slow in bulk.** Touching 150 GiB of a mapped file that is already in the file cache, on 16 threads,
  ran at 2.4-2.8 GB/s. `PrefetchVirtualMemory` in 256 MiB pieces brought 41 GiB back from the SSD in 8-11 s.
- **A pinned copy pushes the rest of the file out of the cache.** After strata-mimo copied 34 layers, the 13 it reads
  through the mapping were gone from RAM, and the first turn after a start took 20 s instead of 3.6. It reads them
  back in on a thread at the start now.
- **Tensor-core FP16 products aren't the limit for MiMo's prompts.** `cublasGemmEx` (FP16 x FP16 -> FP32) at the
  experts' shapes on the 5090: 95 TFLOPS at 64 rows, 200 at 512, 237 from 4,096 rows up, with a floor of about 20 us a
  call. The engine's int8 products keep up with the copies either way; the copies set the pace.
- **A kernel launch costs the host about 4.85 us on WDDM.** A CUDA graph of 1,000 small kernels launches in 3 us and
  runs in 0.61 ms. A decoded DeepSeek token on Maya's engine spends about 12 ms in launches against 4.8 ms in its CPU
  lane, which is why multi-token drafting pays there.
