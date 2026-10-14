KytyPS5 descriptor fix V7 - EXPERIMENTAL DIAGNOSTIC BUILD (Ace Combat 8 PPSA24913 prefs DeviceLost)
=====================================================================================================
Base: KytyPS5 7b9997b + V5 cumulative patch + upstream PR #1194 (= V6). No upstream-main rebase.
V7 patch: descriptor-fix-v7/v7-on-v6.patch (applies on top of V6; SHA256 pinned in build-windows-v7.yml).

CONTENTS
1. Kyty #780 device-fault reporter (itsmemac 1919d512, conflicts hand-merged)
   - VK_EXT_device_fault: queried only after ErrorDeviceLost, counts first then details.
     Logs description, address infos, vendor infos; vendor binary -> _device_fault.nv-gpudmp
   - VK_NV_device_diagnostic_checkpoints: a checkpoint on every compute dispatch
     (groups, mode, shader address, shader hash); last-seen markers dumped on DeviceLost.
   - VK_NV_device_diagnostics_config (shader debug info / resource tracking / error reporting).
   - VK_EXT_device_address_binding_report: BDA bind/unbind ranges recorded (last 16384);
     each fault VA is matched against them ("fault VA ... in BIND/UNBIND ..." lines).
   - Last-256 compute dispatch ring dumped on DeviceLost:
     seq, submit, shader address, shader hash, group counts, mode, indirect args, bda/bda_writes/glc.
   - Every dispatch whose shader address is 0x1640xxxxxx is ALWAYS logged as [watch-0x1640]
     (recurring culprits: args=192,108,1,65 @ 0x16401e1e00 and 189,106,1,65 @ 0x1640211e00).
   - Every extension is optional: enabled only if the driver reports it; otherwise a no-op.
     Startup log line "Vulkan diag: device_fault=.. nv_checkpoints=.. nv_diag_config=.. address_binding_report=.."
2. NeySlim ccfcb32a / #1294: BDA page table zero-filled at creation (always on).
   + de6966b1: fault buffer zero-filled at creation (always on). Test hunks dropped.
3. AnyPS5 #2259 port: coherent (glc/dlc) aligned BDA dword loads become Device-scope relaxed
   OpAtomicLoad when KYTY_GLC_ATOMIC_LOAD=1. DEFAULT OFF. No uvec2 bitcast.
   DEFERRED: #919 64-bit atomic for dwordx2 (our BDA path loads dword-by-dword, so there is no
   single x2 load to convert) and the bound-buffer coherent path (spirvEmitterMemoryHelpers.cpp).
NOT IN V7: #1403, #986 (loop limit), #1138 (atomic RecordBdaFault), #4279 (GDS coherent).

ENV KNOBS
  KYTY_GLC_ATOMIC_LOAD=1   A/B for #2259 (default off)
  KYTY_BDA_WRITES          leave default. NEVER set to 0 (boot EXIT on e41c5e833516362b / cff4569b849942cd)

TEST RUNS (one change per run, 3 repeats each)
  R1: Run-PPSA24913-Diagnostic-V7-R1.cmd       (V7 default)
  R2: Run-PPSA24913-Diagnostic-V7-R2-GLC.cmd   (KYTY_GLC_ATOMIC_LOAD=1)
  Success: prefs/options screen stays up >= 60 s, past VS19/PS57/CS188, no ErrorDeviceLost,
  no nvlddmkm event, no boot EXIT on e41c/cff4.

WHAT TO SEND BACK
  1. BUILD-INFO.txt from this build (kit commit + all patch SHA256s).
  2. Ace Combat 8 patch version: 01.000.000 / 01.001.000 / 01.001.001.
  3. Which run (R1/R2), env vars, freeze type (hard hang/TDR vs clean exit), timestamp.
  4. From the log: the "--- Device fault" block, the "--- Last N compute dispatches" ring,
     the "--- Diagnostic checkpoints" block, all [watch-0x1640] lines, the "Vulkan diag:" line,
     and the last ~200 lines. Plus _device_fault.nv-gpudmp if written.
  5. Windows Event Viewer -> System: nvlddmkm events (ID 13/153, Xid / page fault text) with timestamps.
  6. GPU + driver version; whether TdrDelay is modified.
