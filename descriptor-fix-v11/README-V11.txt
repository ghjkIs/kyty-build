KytyPS5 descriptor fix V11 (experimental) = V10 + v11-on-v10.patch
Base: KytyPS5 7b9997baea0d385e9e8fc2bd3abeb02110bc4148
Apply order: descriptor-fix-v5/descriptor-v5.patch.part00..02 (joined), descriptor-fix-v6/upstream-pr1194-srt-ordinary-reader.diff,
descriptor-fix-v7/v7-on-v6.patch, descriptor-fix-v8/v8-on-v7.patch, descriptor-fix-v9/v9-on-v8.patch,
descriptor-fix-v10/v10-on-v9.patch, descriptor-fix-v11/v11-on-v10.patch.
v11-on-v10.patch SHA256: 8e10f255674a3318f861d5a8498da4e3bb5441a045711f036bb4c66518fe630c

Problem (Ace Combat 8, V10 logs, campaign load):
  shader CFG build failed: unsupported decoded instruction in CFG at pc 0x00002400:
  unsupported family=VOPC opcode=0xf9 raw=[0x7df300f9 0x86060019]
  reason=VOPC SDWA modifier is not supported for opcode (ShaderCFG.cpp:40).
  This was the only "unsupported" decoder line in the V10 R1/R2 detail logs and the main log.

Decoding (RDNA2 / GFX10):
  word0 0x7df300f9: bits[31:25]=0x3e VOPC, OP bits[24:17]=0xf9, VSRC1 bits[16:9]=0x80, SRC0 bits[8:0]=0xf9 (SDWA).
  OP 0xf9 = V_CMPX_NGE_F16 (cmpx family 0xd8..0xdf / 0xf8..0xff). It is a CMPX, so on GFX10 it writes
  EXEC only. It does NOT write VCC: GFX10 CMPX no longer writes VCC/SDST.
  SDWA (SDWAB) word 0x86060019: SRC0=0x19 with S0(bit23)=0 -> v25; SDST[14:8]=0, SD(bit15)=0
  (would select VCC for a non-CMPX compare; ignored for CMPX); SRC0_SEL=6 DWORD, SEXT/NEG/ABS=0;
  SRC1_SEL=6 DWORD, SEXT/NEG/ABS=0; S1(bit31)=1 -> VSRC1 0x80 is a scalar operand code = inline constant 0.
  => EXEC &= !(f16(v25[15:0]) >= 0.0) (true for negative or NaN).
  Root cause: opcode 0xf9 was missing from the VOPC table, so it decoded as UNSUPPORTED and the SDWA
  path reported the generic SDWA rejection.

Fix:
  * VOPC table gains the remaining RDNA2 16/32-bit compares: V_CMP_{F,O,U,NLG,NLE,TRU}_F16,
    V_CMPX_{F,LG,O,U,NGE,NLG,TRU}_F16, V_CMPX_{F,TRU}_F32, V_CMPX_{F,T}_{I32,U32}, V_CMPX_{LE,GE}_U16.
    All are also reachable via VOP3 (LookupVop3Opcode falls back to the VOPC table) and DPP.
  * SupportsVopcSdwa: SDWA is accepted for every implemented 16/32-bit compare/cmpx/class opcode;
    only 64-bit (F64/I64/U64) compares are rejected (they have no SDWA form). An unimplemented
    opcode now reports "VOPC opcode is not implemented" instead of the misleading SDWA message.
  * Operand semantics (unchanged shared code, now reached by the new opcodes): src0/src1 byte/word
    select + SEXT are applied before the compare by ApplyBitSourceModifiers (32-bit) / Read16LaneBits
    (16-bit), including SGPR / inline-constant sources selected by S0/S1. SD=0 -> VCC, SD=1 -> SDST SGPR
    pair; CMPX -> EXEC (exec_lo/hi + thread mask).
  * Translator: new opcodes lowered (F/TRU constants, unordered/ordered F16 compares; O/U F16 now use
    the F16 lane reader via EmitFloatOrderedCompare(..., half=true)).
Tests: tests/shaderCfgTests.cpp TestNewShaderRecompilerVopcSdwaV11 (registered in main): exact captured
  instruction, v_cmpx_nge_f16 sext byte/word, v_cmp_u_f16 to SGPR pair, v_cmpx_le_u16 with scalar src0,
  v_cmp_lt_i32 sext byte2, v_cmp_nlg_f16 scalar src1, 64-bit rejection, and an end-to-end recompile.
  CI builds/runs shader_cfg_tests as a non-blocking final step.
