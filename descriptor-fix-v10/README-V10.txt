KytyPS5 descriptor fix V10 - EXPERIMENTAL DIAGNOSTIC BUILD (V9 + host-side null-write guard)

Base: KytyPS5 7b9997baea0d385e9e8fc2bd3abeb02110bc4148. Apply order (all SHA256-pinned in
.github/workflows/build-windows-v10.yml): descriptor-v5 parts 00..02, upstream PR 1194,
v7-on-v6, v8-on-v7, v9-on-v8, v10-on-v9.

Why: V9 Ace Combat 8 (PPSA24913) dies with VK_EXT_device_fault "WriteInvalid at 0x0"
(precision 0x1000) -> ErrorDeviceLost. In both R1-type crashes the last recorded dispatch is
CS hash 0x68672c344445c550 (201x113 / 206x116 groups, 8x8 local) in a submit whose ring ends with
0xbd0337a734e0873d. NOTE: the ring entry is only the LAST dispatch recorded in the lost submit;
the fault can come from any dispatch/draw in that submit.

What V10 changes (KYTY_NULL_WRITE_GUARD, default ON; set KYTY_NULL_WRITE_GUARD=0 to disable):
 1. Written buffer V# (storage/formatted/atomic, compute AND draw stages, same binding path)
    whose guest base is 0 (or size 0) previously bound the 16-byte Kyty.NullBuffer. It now binds
    a dedicated 16 MiB device-local buffer "Kyty.NullWriteGuard" with range
    clamp(num_records*stride, 64 KiB, 16 MiB). Read-only null V#s are unchanged.
 2. Written storage image T# that is not the canonical null descriptor but has Base40()==0 is
    bound to the 1x1 null storage image (robust image access drops the stores) instead of being
    resolved at guest address 0.
 3. BDA: the shader-side get_bda_pointer already returns 0 for unmapped pages and every BDA
    store/atomic is predicated on a non-zero pointer (stores to page-table-zero pages are dropped
    and recorded in the fault buffer). V10 adds the host side: a cache buffer whose
    VkBuffer device address is 0 never publishes non-zero-looking entries (base+offset) into the
    BDA page table; it publishes 0 (page treated as missing) and logs once.
 Log lines: "[V10] null-write guard ON/OFF", "[V10] null-write guard: shader hash=... writable
 buffer slot=N has base 0 (stride records format size ...)", "... storage image source=N has
 base 0 ...", "... cache buffer ... has device address 0 ...". Once per (shader hash, kind,
 slot), max 256 sites.

Investigation result (honest): in the V9 R1 detail log the faulting dispatch
(0x68672c344445c550) binds 2 read-only V#s (non-zero bases) and 4 storage images with non-zero
bases; no zero-base writable descriptor was seen there, and BDA writes are already null-guarded
in the shader. If V10 still faults with no [V10] guard line printed, the VA-0 write is NOT a
zero V#/T#/BDA base; next suspects: freed/evicted image or buffer still referenced by an
in-flight submit (stale descriptor), shared-memory/LDS emulation buffer, or a draw-side write
(render target / streamout) in the same submit.

Run scripts: Run-PPSA24913-Diagnostic-V10-R1.cmd (default) and -R2-GLC.cmd
(KYTY_GLC_ATOMIC_LOAD=1). Never set KYTY_BDA_WRITES=0.
