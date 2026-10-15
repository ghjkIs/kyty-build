KytyPS5 descriptor fix V15 - EXPERIMENTAL DIAGNOSTIC BUILD (Ace Combat 8, PPSA24913)
=====================================================================================

Chain (clean KytyPS5 7b9997baea0d385e9e8fc2bd3abeb02110bc4148):
  V5 parts 00..02 (6212fc12) + PR1194 (777acbc6) + V7 (e822e3ff) + V8 (0d61586e) + V9 (56848a7e)
  + V10 (52ce5027) + V11 (8e10f255) + V12 (e0d1f349) + V13 (a582b6b4) + V14 (0c8c631d)
  + V15 descriptor-fix-v15/v15-on-v14.patch
    sha256 ef849d138e6962aba0f34184695298e8fca441a31d77ad0e90be63a9a062e33f
All V15 switches default OFF: with no environment variables V15 behaves like V14, except the log
text noted under "Other changes".

What the V14 logs showed (R1, R3, R4, R5-FAST, R6)
-------------------------------------------------
* Every crash is the same: VK_EXT_device_fault WriteInvalid at VA 0x0 (page 0x0..0xfff),
  which matches no bind/unbind range, and then ErrorDeviceLost. The faulting command is
  always the same post-process compute chain:
  0x32b3d126 / 0x811e9c0e / 0x77bbcdac / 0xeaded77b / 0xfebd6d5c / 0xbd0337a7 (indirect) /
  0x68672c34 / 0xdb04c377 (9,6,26) / 0x3fad8c36 x2 (9,6,26) / 6,4,7 dispatch (0x16c0.. / 0x1780..)
  / 0xc50b1c83 / 0xcfc7384c (1536) / 0x1fd0ae81 (1,1,1), followed by a 185..191 x 105..108
  dispatch of a 0x1640xxxxxx shader (the V7 "watch-0x1640" shader). Every dispatch in that chain
  has bda=0 and bda_writes=0.
* R3 (KYTY_SYNC_COMPUTE=1: a barrier after every dispatch and before every draw) and
  R4 (R3 + KYTY_SYNC_SUBMIT=1: queue waitIdle after every submit) crash in the same way.
  So the cause is not a missing GPU->GPU barrier inside the Vulkan command stream. There is only
  one VkQueue and one command buffer at a time (CommandScheduler), and every guest queue
  (graphics plus async compute) is recorded into it in parse order. A barrier at the end of every
  dispatch already orders compute->anything, and waitIdle removes any GPU overlap between submits.
* R1 survived with V14 defaults (KYTY_SYNC_COMPUTE=3) and heavy detail logging. R6 (mode 0) also
  had heavy logging and crashed. R5-FAST (mode 3, no logging) crashed. Heavy logging alone is
  therefore not enough to hide the bug. It only shifts timing, so one survival is not evidence.
* V14 hazard barriers: 64 graphics->draw/dispatch-indirect hits. Producers 0x4d86fa17 and
  0x424781d3, consumer draw 0x7768c721 ("BDA reader after pending graphics write") and
  dispatch-indirect 0xd923c80b ("image reader after graphics storage-image write"). addr=0x0 was
  not a real address. Those consumers (BDA reader / any image reader) have no single range, and
  V14 printed the consumer address field as 0 for them. V15 prints the producer's write ranges
  and the consumer range or "any".
* Note: the V14 ring dump printed the newest entry first, then the oldest (off-by-one); fixed.

Leading hypothesis for V15: a guest CPU <-> GPU ordering race, not a GPU <-> GPU race
--------------------------------------------------------------------------------
EOP / RELEASE_MEM label writes without an interrupt (graphicsRun.cpp WriteAtEndOfPipe write32 /
write64) are memcpy'd into guest memory by the emulator's GPU thread when the PM4 packet is
PARSED. On hardware the label only lands after all earlier work on that queue has finished. In
the emulator, that earlier work is at best recorded in the still-open command buffer: it is not
submitted, let alone executed. A game CPU thread polling the label (or a WAIT_REG_MEM on another
guest queue) then sees "GPU done" early. The game may then reuse, rewrite or read memory (constant
or descriptor rings, indirect args, GPU-produced tables) that the GPU work still needs or has not
yet written. The next frame could then build a V# or pointer from stale data (zero), and a later
compute shader writes through it to VA 0. Barriers and waitIdle cannot fix this because the label
is visible before the submit. Anything that slows the GPU thread relative to the game threads
changes the outcome: per-dispatch logging, NV checkpoints, driver diagnostics instrumentation.
KYTY_EOP_LABEL_SYNC=1 (R11) tests this directly.

V15 switches (environment variables, all default 0)
---------------------------------------------------
KYTY_NV_CHECKPOINTS=1   Enables VK_NV_device_diagnostic_checkpoints. Every compute dispatch and
                        every EOP/debug site records vkCmdSetCheckpointNV(marker).
                        GPU: each checkpoint is a marker write that the driver tracks at the
                        top and bottom of the pipe. On NVIDIA this adds per-command work and
                        can reduce overlap between back-to-back dispatches; the Vulkan spec
                        gives it no ordering guarantee. CPU: one ring-slot write per command.
                        It adds no barriers and does not split submits. On DeviceLost the last
                        executed checkpoints are printed.
KYTY_NV_DIAG_CONFIG=1   Enables VK_NV_device_diagnostics_config with ShaderDebugInfo,
                        ResourceTracking and ShaderErrorReporting at device creation. This is
                        driver-side: resource-tracking bookkeeping on every submit and
                        allocation, plus shader error reporting. It can slow submits and
                        shaders. It adds no barriers or submits of its own.
KYTY_NV_DIAGNOSTICS=1   The V14 umbrella. It only sets the default of the two switches above
                        to 1; an explicit value of either one wins.
KYTY_WATCH_1640=1       Prints and logs a [watch-0x1640] line for every dispatch of a shader at
                        0x1640xxxxxx. This is host-only work on the GPU thread (printf + LOGF,
                        no flush) and changes nothing on the GPU.
KYTY_DISPATCH_LOG_FLUSH=1  Logging-cost emulation: a [dispatch-flush] line plus fflush(stdout)
                        and Log::Flush() for EVERY compute dispatch. This blocks the GPU thread
                        on synchronous file I/O, which delays recording, submits and parse-time
                        label writes. It emits no GPU commands. If this alone avoids the crash,
                        the race is host-timing driven.
KYTY_EOP_LABEL_SYNC=1   Before a parse-time EOP/RELEASE_MEM label write becomes visible to the
                        guest: submit the open command buffer and wait for the GPU
                        (CommandScheduler::Finish, the same path that GDS EOP writes already
                        use). This restores hardware order at the cost of one extra submit
                        and a CPU stall per label.
                        =2: log the first 64 labels ([V15] EOP label lines, queue, submit,
                        address, value) with no waits.
KYTY_HAZARD_COMPUTE_BARRIER=1  When the V15 compute tracker reports a hit (below), also emit an
                        explicit ALL_COMMANDS global memory barrier. This is normally redundant.
Unchanged: KYTY_SYNC_COMPUTE (0..4, default 3), KYTY_SYNC_SUBMIT, KYTY_ACQUIRE_MEM_BARRIER,
KYTY_BDA_WRITES (never set it to 0), KYTY_GLC_ATOMIC_LOAD.

V15 compute hazard tracking (active in KYTY_SYNC_COMPUTE=3/4)
------------------------------------------------------------
* For the last 64 compute dispatches, V15 records the written storage-buffer ranges (guest
  [base, base+size), up to 16 per dispatch), whether storage images were written, and whether
  BDA writes happened, including writes through the BDA page table (no static range: any page).
* Before each dispatch, indirect dispatch (including its 16-byte argument range) and draw (all
  vertex stages + PS), V15 checks the consumer's buffer ranges, whether it reads images, and
  whether it is a BDA reader. Each new (producer, consumer) pair is logged once (up to 256 pairs)
  as:
    [V15] compute hazard #n: producer=<hash> (dispatch -k) -> <kind> consumer=<hash>
          (<why>; read=[a,b) write=[c,d) writer_ranges=N [+image] [+bda]; ordered by ...)
* These hazards are already ordered on the GPU: every dispatch ends with a COMPUTE_SHADER ->
  ALL_COMMANDS barrier on the single queue, and that holds across submits because Vulkan
  barriers apply in submission order on one queue. The report shows which compute->compute flows
  exist around the crash chain. KYTY_HAZARD_COMPUTE_BARRIER=1 adds explicit barriers on top.
* Cross-queue / ring: guest async compute queues and the graphics queue all record into the
  same VkCommandBuffer stream in parse order (the dispatch ring shows interleaved guest submit
  ids such as 6680, 6679, 6681). There is no host-side async compute and no VkQueue overlap.
  The remaining cross-queue edges are WAIT_REG_MEM on labels: CPU-polled, satisfied by the early
  labels above. GDS EOP reads already wait for the GPU. WRITE_DATA is applied at parse time,
  which matches the CP's ME-time write on hardware (no wait for earlier shaders).
* The graphics hazard lines (V14) now read "[V15] hazard barrier ...". They include
  consumer_range=[a,+n) or "any" and producer_writes: ranges=N [first 3 ranges] +storage-image
  +bda/unknown.

Other changes
-------------
* The device-lost dispatch ring is printed oldest -> newest (V14 printed the newest entry first).
* The startup line "GPU: V15 tracking: ..." shows every switch above.

Run scripts (copy next to kyty_emulator.exe, edit GAME= if needed)
------------------------------------------------------------------
| Script                                              | Logging        | Extra environment                       | Tests                                    |
|-----------------------------------------------------|----------------|-----------------------------------------|------------------------------------------|
| Run-PPSA24913-Diagnostic-V15-R1.cmd                 | heavy (detail) | none (V15 defaults)                     | baseline that survived in V14            |
| Run-PPSA24913-V15-R5-FAST.cmd                       | console only   | none                                    | fast baseline (crashed in V14)           |
| Run-PPSA24913-V15-R7-FAST-NV-CHECKPOINTS.cmd        | console only   | KYTY_NV_CHECKPOINTS=1                   | checkpoints alone hide the race?         |
| Run-PPSA24913-V15-R8-FAST-NV-DIAG-CONFIG.cmd        | console only   | KYTY_NV_DIAG_CONFIG=1                   | driver diagnostics alone hide it?        |
| Run-PPSA24913-V15-R9-FAST-WATCH-1640.cmd            | console only   | KYTY_WATCH_1640=1                       | watch lines alone hide it?               |
| Run-PPSA24913-V15-R10-FAST-SYNC-COMPUTE-SUBMIT.cmd  | console only   | KYTY_SYNC_COMPUTE=1 KYTY_SYNC_SUBMIT=1  | full GPU serialisation without logging   |
| Run-PPSA24913-V15-R11-FAST-EOP-LABEL-SYNC.cmd       | console only   | KYTY_EOP_LABEL_SYNC=1                   | early EOP label hypothesis (fix test)    |
| Run-PPSA24913-V15-R12-FAST-DISPATCH-LOG-FLUSH.cmd   | console only   | KYTY_DISPATCH_LOG_FLUSH=1               | logging-cost hypothesis                  |
Logs: ace-combat-log-v15-<R>.txt (R1 also ace-combat-detail-v15-R1.txt).

Result table to fill in (S = survived N min, C = crashed; note tick and debug_submit from the log):
| Run | V15 result | Notes |
|-----|------------|-------|
| R1  |            |       |
| R5  |            |       |
| R7  |            |       |
| R8  |            |       |
| R9  |            |       |
| R10 |            |       |
| R11 |            |       |
| R12 |            |       |
Run each fast script at least twice. A single survival is weak evidence because the crash time
varies a lot between runs (V14 ticks: 19911, 32484, 38564, 62962).

Risks
-----
* KYTY_EOP_LABEL_SYNC=1 adds a submit plus a CPU wait for every non-interrupt label. Expect a
  lower frame rate. If the game polls a label that a later packet in the same buffer must
  produce, the extra Finish only serialises; it cannot deadlock, because the GPU work before the
  label is complete once Finish returns. Mode 2 never waits.
* KYTY_DISPATCH_LOG_FLUSH=1 is very slow (thousands of flushes per frame), and the log files
  grow quickly.
* The compute tracker runs on the GPU thread for every dispatch and draw in modes 3/4: a 64-entry
  scan with small vectors. The cost is small but not zero. Logging is capped at 256 pairs.
* KYTY_HAZARD_COMPUTE_BARRIER=1 ends dynamic rendering before draws that hit, which costs
  render-pass splits.
* This build was syntax-checked locally and compiled only by CI (clang-cl). It has not been run.
