KytyPS5 descriptor fix V4 (Layer A) - EXPERIMENTAL

What this is
- descriptor-v4.patch: cumulative patch against official KytyPS5
  7b9997baea0d385e9e8fc2bd3abeb02110bc4148 = V2 + V3 + a hand port of
  KytyPS5 PR #1224 (TheCruZ) descriptor Phi / atomic V# handling.
  SHA256: 68c673346c67a1ab1bee46b4db89f0f663b1fccf571aadbc5a2735e4c253f4f2
- build-windows-v4.yml (also used as .github/workflows/build-windows.yml on
  branch descriptor-fix-v4): workflow_dispatch, checks out 7b9997b and this
  branch, applies descriptor-fix-v4/descriptor-v4.patch after a pinned SHA256
  check (not embedded, unlike V3), builds launcher +
  kyty_emulator + resource_tracking_tests with clang-cl, runs the
  resource_tracking ctest, uploads artifact KytyPS5-Descriptor-Fix-V4-Windows-x64
  (plus KytyPS5-Descriptor-Fix-V4-Source).

Layer A changes on top of V3
1. SrtWalker: IsRuntimeUniformOp() is exported (was file-static), as in #1224.
2. ResourceTracking LowerDescriptorPhi: with shader writes present, a uniform
   two-arm descriptor diamond is host-lowered (SelectU32 clone in value_storage,
   GPU Phi untouched) if ANY of:
     V2  the descriptor feeds the sole terminal StoreBufferU32,
     V3  the Phi feeds BufferAtomic* and is acyclic and write-free before both
         incoming values (FindUniformAtomicDescriptorPhis, still run first in
         Tracker::Run),
     V4  (#1224) the branch condition reads no memory (ReadConst,
         ReadConstBuffer, buffer/address/image ops) AND the diamond is not
         inside a cycle.
   LaneId / per-lane conditions still fail ValidateRuntimeValue -> rejected.
3. LowerDescriptorWord (#1224): any IsRuntimeUniformOp expression tree over the
   Phi (Phi|imm, (Phi+imm)|imm, Phi<<k, ...) is rebuilt over the host
   selection; shared per descriptor (all 4 dwords) and Phi selections are
   cached across handles, so V3's single shared atomic binding is kept.
4. V3's atomic Phi scan now finds Phis anywhere in such an expression (was only
   Phi or Phi|imm).
5. No Jetsku RT, no Layer B BDA writes, no skip-instead-of-exit.

Behaviour changes vs V3 (intended, from #1224)
- A user-data predicated descriptor Phi is now accepted even when a shader
  write precedes the diamond or the store is not terminal (V2/V3 rejected
  these). Memory-predicated selection after a write is still rejected.
- Deviation from #1224: #1224 rejects a memory-predicated atomic V# whenever
  the shader writes; V4 accepts it when V3's write-free/acyclic rule holds
  (required for the Ace Combat 8 shape and the V3 contract).

Status
- Linux/GCC 14.2 resource-tracking suite passes (see VALIDATION-NOTES.txt).
- Windows build: GitHub Actions run (see final report). Ace Combat 8 and Astro
  Bot are UNVERIFIED. Do not assume any game boots.
