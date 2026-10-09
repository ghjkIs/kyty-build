KytyPS5 descriptor fix V6 (V5-B + upstream PR #1194) - EXPERIMENTAL

Base: KytyPS5/KytyPS5 7b9997baea0d385e9e8fc2bd3abeb02110bc4148
Applied in order by .github/workflows/build-windows-v6.yml (also build-windows.yml on this branch):
  1. ../descriptor-fix-v5/descriptor-v5.patch.part00..02 (cumulative V4 Layer A + V4B Layer B + V5-B)
     sha256 6212fc121db62685ca327d14503ef6297de8b949315c89339062cccf3757f00b  (unchanged from V5)
  2. upstream-pr1194-srt-ordinary-reader.diff  (KytyPS5 PR #1194, applied as-is, no path tweaks needed)
     sha256 777acbc6a80da0f39526d645b96114cfb71cdfbdabfe59a599044f7ae55a37a5
Cumulative result (V5 + #1194 vs 7b9997b) is emitted by CI as descriptor-v6.patch
  (Linux-generated sha256 e8cbcb9c0e8935abc147282b4f27de335f9581172d94aceaf4c43630ff34bded).

What V6 fixes (W0):
- Ace Combat 8 (PPSA24913) on V4B/V5 died with "runtimeLinker.cpp:720". That line is just the EXIT in
  KytyExceptionHandler; the fault is a HOST thread read of address 0 inside Kyty itself.
- Cause: SrtWalker::EvaluateRawRead specializing CS 0x57f3b48d4975aff3 evaluates LoadAddressU32 from
  s[4:5] (=0; the shader null-guards it). pipelineCache.cpp never set SrtRuntime::read_memory, so the
  walker fell back to a raw memcpy from guest address 0 -> access violation.
- PR #1194 ("graphics: read unmapped shader resource memory as zero") adds ReadShaderOrdinaryMemory +
  Memory::IsGpuMapped and sets .read_memory in pipelineCache.cpp; unmapped words read as 0
  (null descriptor), matching the shader's own null-guarded path. Same bug as upstream #1268 / #1070
  (Astro Bot). #1194 was closed upstream without merge; upstream main is still unfixed.

Not the cause: Usbd (sceUsbdGetDeviceList 8qB9Ar4P5nc / FreeDeviceList EQ6SCLMqzkM) and the
NpCppWebApi_v1.1 NIDs are optional unresolved imports. Their stubs return 0 and are not causal.

NOT included in V6 (deferred, see V6-LINKER-STUBS.md section 5): refuse-below-0x10000 / no-memcpy
fallback in EvaluateRawRead and CaptureOrdinaryRead, host-code crash attribution, Usbd cosmetic stubs.
Next likely walls (W1-W7): see V6-NEXT-WALLS.md.

Status: patches apply cleanly (git apply --check + diff --check) on 7b9997b; Linux tests not rerun for
#1194 (touches pipelineCache/renderContext/memory, outside resource_tracking_tests). No game run yet.
