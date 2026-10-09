KytyPS5 descriptor fix V5 (V5-B) - EXPERIMENTAL

Base: KytyPS5/KytyPS5 7b9997baea0d385e9e8fc2bd3abeb02110bc4148
Recommended patch: descriptor-v5.patch (cumulative: V4B + V5-B)
  sha256 6212fc121db62685ca327d14503ef6297de8b949315c89339062cccf3757f00b
Alt: descriptor-v5.patch (V4 + V5-B, no BDA writes) was not pushed;
     v5b-incremental.patch (V5-B only) available locally.

V5-A (IndirectBuffer U8/U16 loads) was already part of V4/V4B - nothing new there.
V5-B: uniform, write-free descriptor Phis that feed raw buffer loads
(LoadBufferU8/U16/U32/x2/x3/x4) are host-lowered to SelectU32 like V3 atomics,
so they bind a normal buffer instead of a BDA IndirectBuffer read. Diamonds on
a cycle are accepted when loop-invariant (inputs defined off the cycle, memory
reads before any write). LaneId / loop-carried selections still fall back.

Status: Linux resource_tracking_tests pass (V4 and V4B trees). No Windows build,
no game booted. See V5-B-PATCH-PLAN.md. Run with KYTY_BDA_WRITES=1.
