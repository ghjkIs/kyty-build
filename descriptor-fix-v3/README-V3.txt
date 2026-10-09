KytyPS5 descriptor fix V3 - EXPERIMENTAL
Target: Ace Combat 8 (PPSA24913), compute shader hash 0x1d81bb2b74faf456
        "unsupported GPU-selected access: opcode=BufferAtomicOr32"

This is a build kit, not a prebuilt EXE. GitHub Actions builds the EXE.

HOW TO BUILD AND RUN
1. Branch descriptor-fix-v3 of kyty-build holds the V3 workflow twice:
   .github/workflows/build-windows-v3.yml and, on this branch only,
   .github/workflows/build-windows.yml (same V3 content). main still has the
   V2 build-windows.yml, untouched, for comparison.
   Why both: GitHub can only dispatch workflows whose file exists on the
   default branch (main). build-windows-v3.yml is not on main yet, so it
   cannot be started directly; build-windows.yml is registered, and
   dispatching it with the branch selected runs the branch's (V3) contents.
2. Actions > "Build KytyPS5 descriptor fix V2 for Windows" (the registered
   name of build-windows.yml) > Run workflow > Branch: descriptor-fix-v3.
   The run itself is titled "Build KytyPS5 descriptor fix V3 for Windows".
   If you merge descriptor-fix-v3 into main later, build-windows-v3.yml shows
   up as its own entry and build-windows.yml on main becomes V3 as well.
3. When the run is green, download the artifact
   KytyPS5-Descriptor-Fix-V3-Windows-x64 (patched source is in
   KytyPS5-Descriptor-Fix-V3-Source).
4. Extract the whole ZIP into a NEW folder (keep the DLLs next to
   launcher.exe and kyty_emulator.exe). Do not overwrite V2.
5. Double-click Run-PPSA24913-Diagnostic.cmd. It uses
   F:\Games\PS5 Emulator and Games\PPSA24913-app0 (edit the GAME line if needed).
6. If it still exits, collect ace-combat-detail-v3.txt and ace-combat-log-v3.txt.

PATCH
descriptor-v3.patch is CUMULATIVE against official KytyPS5 revision
7b9997baea0d385e9e8fc2bd3abeb02110bc4148. It already contains everything in V2.
Do NOT apply V2 and V3 together; apply only descriptor-v3.patch to a clean tree.
SHA256: bbc01f37ce41f4a7c7f1e811ba769a987375402406d3aa07cbaae5d62f45e4bf

WHAT V3 CHANGES (ResourceTracking.cpp)
V2 only let the host evaluate a branch-merged (Phi) buffer address when the
shader had a single, final StoreBufferU32. The Ace Combat shader instead uses
the same merged address (lo Phi, hi Phi | 0x40000 flag) for four
BufferAtomicOr32 operations, so V2 still failed.
V3 scans GetBufferResource handles used by any BufferAtomic* op. A Phi in the
descriptor (directly, or under BitwiseOr32 with an immediate flag) qualifies
when: it is a 2-arm diamond/triangle on a conditional branch; the branch
condition and both incoming values are uniform runtime values (LaneId or other
per-lane values are rejected); the diamond is not inside a loop; and no shader
write can execute before the incoming values are produced. Such Phis get a host
SelectU32 (plus the cloned BitwiseOr32) for descriptor planning, any number of
uses is allowed, and the GPU Phi nodes are left unchanged. All atomic sites then
share one normal writable+atomic buffer binding.

LIMITS
Experimental. Not run on Windows, on a GPU, or with the game while this was
prepared. Ace Combat 8 may hit a different emulator limitation afterwards.
This does not claim the game boots.
