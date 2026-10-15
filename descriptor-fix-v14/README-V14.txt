KytyPS5 descriptor fix V14 - EXPERIMENTAL DIAGNOSTIC BUILD (Ace Combat 8 / PPSA24913)

Layering (all on clean KytyPS5 7b9997baea0d385e9e8fc2bd3abeb02110bc4148):
  V5 parts -> PR1194 -> v7 -> v8 -> v9 -> v10 -> v11 -> v12 -> v13 -> v14-on-v13.patch
  v14-on-v13.patch sha256 0c8c631d1b65a9fb7c00ed426f1fca9856fd76bd6726d44a78949b7c09489a2e

V13 status: opening cutscene reached. R1 (no extra sync) dies with VK_EXT_device_fault
WriteInvalid @0x0 + DeviceLost; R2 (GLC atomic load), R3 (SYNC_COMPUTE=1), R4 (+SYNC_SUBMIT)
survive at 1-2 fps. Log: "pipeline cache: disabled (dirty build)". Black blobs on hair/foam/spray.

1. Synchronization (goal 1)
 Finding: every compute dispatch already ends with an upstream COMPUTE_SHADER -> ALL_COMMANDS
 memory barrier (ShaderAccessBarrier, SHADER_READ|WRITE -> MEMORY_READ|WRITE), and writing
 dispatches start with an ALL_COMMANDS -> COMPUTE barrier. So a dispatch->dispatch/draw race
 through compute-stage writes (bound buffers, storage images, BDA) is already ordered in R1.
 What KYTY_SYNC_COMPUTE=1 adds is an ALL_COMMANDS source scope, i.e. ordering of writes made by
 GRAPHICS stages (and draw->draw). Unbarriered producers found in the code:
   - graphics-stage storage IMAGE writes and graphics BDA writes (only bound storage BUFFER writes
     got the post-draw ShaderWriteBarrier);
   - the post-draw ShaderWriteBarrier did not include DRAW_INDIRECT / INDIRECT_COMMAND_READ.
 The V13 R1 log does not identify the faulting pair (checkpoint markers fell outside the ring;
 fault VA 0 matched no bind). The last op recorded in the failing command buffer was CS
 0x1640211e00 (hash 0x9807906298567ec4) at 187x106 groups -- previous frames ran it at 178x100,
 i.e. the game had just changed dynamic resolution, and at the new size the dispatch never reached
 the ring (consumed by the compute meta/image-clear fast path). DeviceLost is reported at the
 submit of 10551, so the faulting work is in an earlier, already submitted batch.
 V14 changes:
   - KYTY_SYNC_COMPUTE=3 (NEW DEFAULT): hazard tracker. After a draw, graphics-stage storage-image
     and BDA writes are recorded (buffer writes are already barriered). Before the next
     dispatch/draw that reads an image, uses BDA, or reads an overlapping buffer/indirect-arg range,
     ONE global ALL_COMMANDS barrier is emitted and the tracker cleared. Each such site is logged:
       [V14] hazard barrier #N: producer=<shader hash> -> dispatch|dispatch-indirect|draw
             consumer=<shader hash> (<reason>, addr=...)
     (first 64, then every 4096th). Those lines are the racing pairs. Global barriers from
     ACQUIRE_MEM / cache-flush events reset the tracker.
   - KYTY_SYNC_COMPUTE=4: mode 3 + mode 2. 0 = V13 behaviour. 1 and 2 unchanged.
   - ShaderWriteBarrier (after draws with storage-buffer writes) now also targets DRAW_INDIRECT
     with INDIRECT_COMMAND_READ.
   - CPU readback of indirect args: DrawIndirect / DrawIndirectMulti (args + count) and
     DispatchIndirect with USE_THREAD_DIMENSIONS read through TryReadBufferBacking: GPU-dirty bytes
     are downloaded first, which submits and waits for the producing work.
 If R1 (mode 3) still faults, run R6 (=V13 sync) vs R1 vs R3 and send the [V14] lines, and try
 KYTY_SYNC_COMPUTE=4.

2. Pipeline cache (goal 2)
 Cause: PipelineCache::InitializeDriverCache refuses KYTY_GIT_HASH ending in "-dirty"; the kit
 applies patches to the checkout, so every kit build is dirty. VkPipelineCache disk persistence
 already existed (_PipelineCache/<TITLE>.bin, saved only on clean shutdown).
 V14: new CMake cache var KYTY_PATCH_ID (the workflow passes the SHA256 of
 descriptor-v14-cumulative.patch). A dirty build with a patch id keeps the cache; the id is part
 of the file name (_PipelineCache/PPSA24913-<id16>.bin) and of the header signature, so a
 different patch stack never loads a stale cache. The cache is now also written every 32 new
 pipelines (at most every 20 s), so crashes / DeviceLost / closing the window keep it.
 Note: this caches the DRIVER compile (SPIR-V -> GPU code); guest-shader recompilation to
 SPIR-V still happens each run. Async/precompile was not added (pipelines are created on the main
 thread; making that async is not a small change).

3. f16 compares from V11 (goal 3)
 All 21 added VOPC opcodes were checked against the RDNA2 ISA encoding and NaN semantics:
   F/TRU (f16, cmpx f32) constant false/true; O = !(isnan a || isnan b); U = isnan a || isnan b;
   LG = FOrdNotEqual (false on NaN); NGE = FUnordLessThan; NLG = FUnordEqual;
   NLE = FUnordGreaterThan (all N* true on NaN); CMPX_{F,T}_{I32,U32}; CMPX_{LE,GE}_U16.
 Opcode numbers (0xc8/0xcf/0xe8/0xea/0xec/0xef, 0xd8/0xdd/0xdf/0xf8-0xfa/0xff, 0x10/0x1f, 0x90/0x97,
 0xd0/0xd7, 0xbb/0xbe) all match. The pre-existing f16 compares (EQ/LG/GT/GE/LT/LE ordered,
 NGE/NGT/NLE/NEQ/NLT unordered) are also correct. f16->f32 is GLSL UnpackHalf2x16 (exact, NaN and
 denormals kept); modules declare SignedZeroInfNanPreserve(32), so OpFUnord*/OpFOrd* keep NaN.
 SDWA: Read16LaneBits reads the raw register (DPP only), then extracts WORD_1 = bits [31:16],
 WORD_0 = [15:0], BYTE_n = 8 bits, then applies abs/neg on bit 15; correct for f16. No condition
 was changed in V14.
 Likely cause of black blobs: not the compares. Dithered / alpha-tested hair, foam and spray feed
 TAA / temporal upscaling; black blobs that grow or flicker there are typically NaN/Inf or
 uninitialized history written into a render target and then spread by temporal resolve/bloom.
 Candidates: (a) the unordered graphics storage-image writes that mode 3 now orders (stale or
 half-written UAV data); (b) V13 zero-filled image backing (all-zero texels give 0/0 -> NaN in
 normalisation); (c) MSAA/sample-mask or alpha-to-coverage for dithered materials. Compare R1(V14)
 with R6 and grep [V13] "zero-filled" lines for the affected images.

4. Other cheap perf wins
 - The V13 run scripts used --graphics-debug-dump true + shader logs + printf to file: the V13
   detail logs reached 0.44-3.9 GB per run (~570k "CP packet" lines per 200 MB). That alone keeps
   the game at 1-2 fps (R2 was just as slow with no extra barriers). V14 adds
   Run-PPSA24913-V14-R5-FAST.cmd (default sync, no detail/shader logs) for performance checks.
 - NV diagnostic checkpoints (vkCmdSetCheckpointNV on every dispatch) and NV diagnostics config
   (shader debug info, resource tracking, error reporting) are now OFF by default;
   KYTY_NV_DIAGNOSTICS=1 turns both back on. VK_EXT_device_fault, address-binding report and the
   256-dispatch ring stay on (cheap).
 - [watch-0x1640] per-dispatch console lines are OFF by default (KYTY_WATCH_1640=1).
 - Not changed (correctness): V4B SettleBdaWrites waits for the GPU after every BDA-writing
   dispatch (m_scheduler.Wait); that serialises CPU and GPU for those dispatches.

Run scripts: R1 = default (mode 3 hazard), R2 = KYTY_GLC_ATOMIC_LOAD=1, R3 = KYTY_SYNC_COMPUTE=1,
R4 = SYNC_COMPUTE=1 + SYNC_SUBMIT=1 (R1-R4 keep the detail logs), R5-FAST = default without detail
logs, R6 = KYTY_SYNC_COMPUTE=0 (V13 sync) with detail logs. Logs: ace-combat-log-v14-R*.txt.
Startup lines: "GPU: V14 sync: KYTY_SYNC_COMPUTE=3 (...)", "GPU: V14 NV diagnostics ...: off",
"Vulkan pipeline cache: dirty build keyed by patch id ...".

Risks: mode 3 only orders producers it can see (graphics storage images/BDA, buffer ranges); if
the fault is not a sync problem (dynamic-resolution image recreation, the meta/image-clear fast
path, use-after-free of a GC'd buffer), R1 can still fault -- then use mode 1/4 and send logs.
Indirect-arg readback can now stall (submit + wait) when the args are GPU-dirty. A bad driver
cache is rejected by signature/hash; delete _PipelineCache if in doubt. With NV diagnostics off,
a DeviceLost report has no checkpoint markers (set KYTY_NV_DIAGNOSTICS=1 to get them).
