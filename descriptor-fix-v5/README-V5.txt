KytyPS5 descriptor fix V5 — RESEARCH STUB (EXPERIMENTAL)

Status: research only. No patch yet. Wait for V4B Ace Combat log on
compute hash 0xe41c5e833516362b before implementing.

Plan (see V5-RESEARCH.md):
  V5-B (preferred): host-lower Phi for buffer loads (U16/U8/U32*),
    with loop-invariant exception so Ace Combat's 3-loop shader can pass.
  V5-A (fallback): broaden IndirectBuffer allow-list to U16/U8 only if
    ResourceMaterialization / SPIR-V emit already support those loads.

Base: descriptor-fix-v4b (BDA writes + compute scalar/write overlap allow).
Do NOT pull Jetsku software RT / Astro Bot path.

Bot2 wrote V5-RESEARCH.md; Bot1 will implement after V4B results.
