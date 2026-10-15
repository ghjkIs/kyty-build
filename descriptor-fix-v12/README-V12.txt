KytyPS5 descriptor fix V12 (on V11) - EXPERIMENTAL DIAGNOSTIC BUILD
====================================================================

Base: KytyPS5 7b9997baea0d385e9e8fc2bd3abeb02110bc4148 + V5 parts + PR1194 + V7 + V8 + V9 + V10 + V11
Patch: v12-on-v11.patch  SHA256 e0d1f34901eef03bb3d77f543d7adb40996dbd94d3b682fa5ea55ebb36d5cbfa

Why
---
V11 short logs (two runs) still end with VK_EXT_device_fault "WriteInvalid at 0x0" that
"matches no recorded bind/unbind", then vkQueueSubmit -> ErrorDeviceLost
(commandScheduler.cpp:385). The V10 null-write guard never logged, so no bound writable
descriptor had base 0: the zero address is computed on the GPU (BDA / pointer loaded from
memory). R2 (KYTY_GLC_ATOMIC_LOAD=1, coherent loads) survives much longer, which points at a
read of stale/not-yet-visible data -> a missing dependency between compute dispatches,
compute and draws, or host writes and the GPU.

Sync review (what Kyty does with guest sync packets)
----------------------------------------------------
* EVENT_WRITE CS_PARTIAL_FLUSH / VS(GS)_PARTIAL_FLUSH / PS_PARTIAL_FLUSH (0x07/0x0f/0x10)
  and CB/DB writeback-invalidate events -> EmitGlobalBarrier (all commands, memory write ->
  memory read|write). OK.
* EVENT_WRITE 0x0d,0x0e,0x12,0x17-0x1b,0x38,0x3a -> logged and ignored (pipeline stats /
  streamout / misc; not memory ordering).
* RELEASE_MEM -> global barrier unless (event 0x28 BOTTOM_OF_PIPE and no GCR cache bits). OK.
* ACQUIRE_MEM (IT_ACQUIRE_MEM 0x58 and the Agc custom R_ACQUIRE_MEM NOP) -> WAS A NO-OP.
  This is the packet games use (AgcDcbAcquireMem / AgcAcbAcquireMem) to make the results of a
  previous dispatch/draw/CP write visible to the next one (GL0/GL1/GL2 invalidate). Without a
  Vulkan barrier, a dispatch reading a pointer table written by the previous dispatch can see
  stale data (e.g. 0) -> write to VA 0.
  FIX (default ON): CpOpAcquireMem now emits EmitGlobalBarrier(). KYTY_ACQUIRE_MEM_BARRIER=0
  restores the old no-op (for A/B testing).

New debug switches (default OFF, printed once at startup as "GPU: V12 sync debug: ...")
-----------------------------------------------------------------------------------
KYTY_SYNC_COMPUTE=1  global barrier after every compute dispatch and before every draw
                     (srcAccess MemoryWrite -> dstAccess MemoryRead|MemoryWrite, all
                     commands; covers shader storage/BDA read/write and indirect-args reads).
KYTY_SYNC_COMPUTE=2  cheaper: barrier only after a dispatch that is followed by another
                     dispatch or a draw (no draw->draw barriers). Not filtered by
                     BDA/storage usage.
KYTY_SYNC_SUBMIT=1   wait for every vkQueueSubmit to finish (queue.waitIdle under the queue
                     lock) before continuing. Very slow; serializes host and GPU completely.

Run scripts
-----------
R1  plain                                  R2  KYTY_GLC_ATOMIC_LOAD=1
R3  KYTY_SYNC_COMPUTE=1                    R4  KYTY_SYNC_COMPUTE=1 + KYTY_SYNC_SUBMIT=1

How to read the results
-----------------------
R1 now survives      -> ACQUIRE_MEM gap was the bug.
R1 dies, R3 survives -> a remaining compute->compute/draw ordering gap (try =2 to narrow).
R3 dies, R4 survives -> host/GPU race (CPU reading GPU results or writing in-flight memory,
                        e.g. indirect draw args read on the host in DrawIndirectMulti).
R4 dies              -> not a sync problem; garbage address comes from elsewhere.

Risks
-----
ACQUIRE_MEM barrier is unconditional and ends the current render pass (EndRendering), so
games that issue many ACQUIRE_MEMs inside graphics passes may lose some performance.
R3/R4 are much slower by design. The barriers are executed only in the command buffer the
CP is recording; cross-queue (ACB vs DCB) ordering is not changed.
