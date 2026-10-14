KytyPS5 descriptor fix V9 (experimental) = V8 + v9-on-v8.patch
Base: KytyPS5 7b9997baea0d385e9e8fc2bd3abeb02110bc4148
Apply order: descriptor-fix-v5/descriptor-v5.patch.part00..02 (joined), descriptor-fix-v6/upstream-pr1194-srt-ordinary-reader.diff,
descriptor-fix-v7/v7-on-v6.patch, descriptor-fix-v8/v8-on-v7.patch, descriptor-fix-v9/v9-on-v8.patch.
v9-on-v8.patch SHA256: 56848a7e58a04e35ac4a2087a1c89abff2772d1a552d0e92299c0728c2b3cea2

Problem (Ace Combat 8, V8 logs):
  shader CFG build failed: unsupported decoded instruction in CFG at pc 0x0000054c:
  unsupported family=VOP2 opcode=0x11 raw=[0x22221cf9 0x03860611]
  reason=VOP2 SDWA modifier is not supported for opcode (surfaced via ShaderCFG.cpp:40;
  rejection originates in frontend/decode/VectorAluOps.cpp ValidateVop2Sdwa).

Decoding: word0 0x22221cf9 -> VOP2 op 0x11 = V_MIN_I32 (RDNA2), vdst v17, src0=0xf9 (SDWA), vsrc1 v14.
  SDWA word 0x03860611: src0 reg 0x11 with S0=1 (s17), src0_sel=DWORD, dst_sel=DWORD, dst_unused=PAD,
  src1_sel=BYTE_3, src1_sext=0, no neg/abs/clamp/omod.
  => v17 = smin(s17, zext(v14[31:24])).

Fix: V_MIN_I32 gets SDWA profile IntegerPartialDestination (same as V_MIN_U32/V_ADD_NC_U32);
  V_MAX_I32 and V_MAX_U32 upgraded from IntegerFullDestination to IntegerPartialDestination.
  Source extraction is done by the existing shared SDWA source reader in translate/Translate.cpp
  (BitFieldUExtract, or BitFieldSExtract when SEXT=1), so signed/unsigned byte/word semantics come
  from the encoded SEXT bit; SMin32/SMax32/UMin32/UMax32 then compare the extracted 32-bit values.
  Partial destinations use the shared WriteRawU32 insert path (dst_unused PAD/SEXT/PRESERVE).
  Neg/abs remain rejected for these integer ops (as on hardware they are float-only).
Tests: tests/shaderCfgTests.cpp gains the exact captured instruction plus v_max_i32 (sext byte0),
  v_min_u32 (word dst), v_max_u32 (byte1) decode checks. CI builds/runs shader_cfg_tests as a
  non-blocking final step.
