KytyPS5 descriptor fix V4B (Layer A + Layer B) - EXPERIMENTAL
=============================================================

Base:          KytyPS5/KytyPS5 7b9997baea0d385e9e8fc2bd3abeb02110bc4148
Patch:         descriptor-v4b.patch (cumulative vs 7b9997b, LF-only)
Patch SHA256:  265aea350e932f2dcc754f7101eefde18be8898286c38efd651e9c04a861e313
Workflow:      .github/workflows/build-windows-v4b.yml (identical copy in build-windows.yml)
Artifact:      KytyPS5-Descriptor-Fix-V4B-Windows-x64

Status: builds and Linux resource-tracking tests only. No game was booted. Ace Combat 8
and Astro Bot compatibility are UNVERIFIED.

Layer A (unchanged from V4)
---------------------------
V2 indirect buffer reads + terminal StoreBufferU32 descriptor Phi, V3 host Select lowering
for multi-use uniform descriptor Phis (BufferAtomic*), V4 port of KytyPS5 PR #1224. All V4
hunks are carried verbatim. Layer B only runs where Layer A still fails.

Layer B: compute BDA writes (hand-ported idea, no Jetsku tree merge)
---------------------------------------------------------------------
Where ResourceTracking would stop with "buffer descriptor is not a valid runtime value"
for a write, it now accepts the write through the GPU-computed V# when all of these hold:
  * the shader is a compute shader,
  * KYTY_BDA_WRITES is enabled (it is by default),
  * the op is a raw (unformatted) BUFFER_STORE dword/x2/x3/x4, byte or short, or a 32-bit
    BufferAtomic* (Swap, CmpSwap, Add, Sub, S/U Min/Max, And, Or, Xor, FMin, FMax).
The memory op becomes ResourceKind::IndirectBuffer and info.bda_writes=1. The SPIR-V
backend resolves base+offset through the existing BDA page table (GetBdaPointer, 16 KiB
pages). It checks V# bounds per dword, stores through a PhysicalStorageBuffer pointer
(Aligned 4) or does the atomic there, and sets the page's bit in a "written pages" bitmap
in the second half of the fault buffer. Byte and short stores use an atomic
read-modify-write mask merge.
has_address_writes is NOT set, so compute control-flow specialization stays on and the
descriptors.cpp:1011 "cannot be proven disjoint from shader address writes" EXIT does not
trigger.

Post-dispatch sync: after every dispatch with bda_writes, FaultManager::SettleBdaWrites:
  1. inserts a barrier,
  2. compacts the bitmap with fault_buffer_process.comp (built with MAX_PAGE_FAULTS=65536),
  3. waits for the GPU,
  4. calls BufferCache::MarkBdaWritten for each page: GPU-modified tracking plus
     m_gpu_modified_ranges plus texture-cache invalidation.
Later CPU reads and specialization snapshots then download the GPU data first.

Environment:
  KYTY_BDA_WRITES   unset/1/on = ON (default); 0/off/false = OFF (upstream behaviour,
                    including the overlap EXIT); verify = ON, and dropped writes or
                    page-list overflow are fatal.
  KYTY_BDA_STORE_LOG  1/on = log BDA writes dropped because the page was unmapped
                    (count + last page, from a small tail area of the fault buffer).
                    Default OFF. Only meaningful with BDA writes on.
Why default ON: the only shaders affected are ones that would otherwise hit a fatal EXIT,
so ON cannot regress a shader that already worked. KYTY_BDA_WRITES=0 is the escape hatch.

descriptors.cpp overlap check (precise relaxation)
--------------------------------------------------
"scalar resource reads overlap a shader buffer write" is skipped only when
AllowComputeSnapshotOverlap() is true:
  * the pipeline has exactly one stage, that stage is Compute, and KYTY_BDA_WRITES is on.
Applied at:
  (a) the FindBuffers sentinel-range check,
  (b) the CommitBindings bound-buffer check (old line ~1034, the Ace Combat 8 failure on
      CS 0xe41c5e833516362b right after "GPU: using buffer device address"),
  (c) a new check that would EXIT for shaders with Layer B BDA writes.
Each bypass logs once per shader hash:
  descriptors: V4B allowing compute scalar-read/buffer-write overlap (<site>): shader=0x...
  read=[...) write=[...) bda_writes=N; ...
Unchanged: graphics pipelines, image/attachment overlaps, and the has_address_writes EXIT.
Rationale: a compute dispatch's scalar reads are captured right before it runs (after
GPU-dirty bytes are pulled back). On AMD hardware the scalar cache is not coherent with the
same wave's vector stores either. The overlapping write is GPU-dirty tracked, so the next
dispatch re-reads it. 0xe41c reads its table with s_load at offsets 64..172 and writes back
with BUFFER_STORE_DWORDX4/X2 at 152/168 at the end of the shader: a self-state write-back.

Jetsku U59 logs
---------------
The attached U59 logs are a Jetsku f9e1958 Ace Combat 8 run. U59 dies earlier, on the same
CS 0xe41c5e833516362b at ResourceTracking pc=0x130 (needs raw DWORD x2/x3/x4 loads), so it
never reaches the overlap check and gives no reference for it. That load case is planned
for V5 and is NOT in V4B.

Not included: Jetsku software RT, AMD wave32/LDS, skip-instead-of-exit, launcher settings,
and V5 x2/x3/x4 indirect loads.

Manual repro (Ace Combat 8, PPSA24913) - not performed by us
-------------------------------------------------------------
1. Download KytyPS5-Descriptor-Fix-V4B-Windows-x64 and extract it.
2. Edit GAME= in Run-PPSA24913-Diagnostic.cmd and run it.
   Logs: ace-combat-log-v4b.txt, ace-combat-detail-v4b.txt, _Shaders-V4B\
3. Look for, in this order:
     "GPU: using buffer device address"
     "GPU: V4B BDA writes enabled ..."
     "descriptors: V4B allowing compute scalar-read/buffer-write overlap (bound buffer):
      shader=0xe41c5e833516362b ..."
   and possibly a "BdaSettle" line if a shader uses Layer B writes.
4. Expected outcome: the 0xe41c descriptors.cpp:1034 EXIT is gone. The next blocker may be
   another shader or check; report the last ~200 lines. To compare with upstream behaviour,
   run with "set KYTY_BDA_WRITES=0" before the emulator line. For dropped-write
   diagnostics, use "set KYTY_BDA_STORE_LOG=1" (or KYTY_BDA_WRITES=verify).
