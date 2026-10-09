# V5 research: GPU-selected buffer loads (Ace Combat CS `0xe41c5e833516362b`)

Branch plan: `descriptor-fix-v5` off `descriptor-fix-v4b` (after V4B lands BDA writes + overlap allow).
Do **not** wait on V4B CI — this is research + draft patch plan only.

## Symptom

| Build | Where it dies | Message |
|-------|---------------|---------|
| Jetsku U59 | ResourceTracking.cpp:173, `pc=0x130`, hash `0xe41c…` | `GPU-selected access requires a raw DWORD x2/x3/x4 load` |
| Our V4 Layer A | descriptors.cpp:1034 after BDA on | `scalar resource reads overlap a shader buffer write` |
| Ace Combat Phi wall (V3) | hash `0x1d81…` | already cleared by V3/V4 |

U59 never reaches our V4B overlap — it dies earlier on the **same** Ace Combat CS hash. U59’s Astro win is unrelated (software RT).

## Root cause (upstream Kyty `Collect` in ResourceTracking.cpp)

When a buffer op’s `GetBufferResource` handle is GPU-selected (Phi / Phi\|imm) and host lowering fails:

```text
GetHandle(...) == false
  → if !SupportsIndirectBufferLoad(op):
        Fail("… GPU-selected access requires a scalar, raw DWORD x1/x2/x3/x4, or formatted X load")
  → else:
        demote memory.kind = IndirectBuffer; uses_dma = true
```

`SupportsIndirectBufferLoad` (ShaderIR.h) only allows:

- untyped 32-bit: `ReadConstBuffer`, `LoadBufferU32`, `LoadBufferU32x2/x3/x4`
- formatted: `LoadBufferU32` only

**Not** `LoadBufferU16` / `U8` / atomics / stores (those need host Phi lowering or BDA writes).

U59’s shorter “x2/x3/x4” text is the same gate with a narrower allow-list.

## IR for hash `0xe41c5e833516362b` (U59 log)

- CFG: **60 blocks, 3 loops** (Structurize → 66 blocks).
- Fail attributed to `pc=0x00000130` = start of **Block `$4`**, which is a Phi merge:

```text
Block $4 pc=0x00000130..0x00000138
  %137..%142 = Phi [$2]/[$3]   ← %141 uses=7
```

Later (Block `$11`):

```text
%239 = GetBufferResource %69, %72, %75, %78
%240 = LoadBufferU16 %239, %238, …, %228
```

So the live wall is almost certainly: **Phi’d V# → `LoadBufferU16`**, which cannot demote to IndirectBuffer. Host Phi lowering either never ran (loop / write-before / non-uniform gates from V3/V4/#1224) or only covers atomics/stores, not loads.

Native RDNA2 near that region also has `BUFFER_LOAD_DWORDX4` / `DWORDX2` (those *would* demote if GetHandle failed) — the U16 path is the Ace-specific killer.

## V5 goals (Ace Combat, not Astro)

1. Get past `0xe41c…` ResourceTracking on Phi’d buffer **loads** (esp. U16).
2. Keep V4B’s BDA-write + overlap allow (separate wall after BDA on).
3. Do **not** pull Jetsku RT / `KYTY_RT_SOFTWARE`.

## Draft patch plan

### Layer V5-A — IndirectBuffer for U16/U8 loads (fast, matches U59’s “allow list” idea)

In `MemoryInfo::SupportsIndirectBufferLoad`:

- Also allow `LoadBufferU16` / `LoadBufferU8` when `!typed && data_bits` matches (U16 → 16, U8 → 8), **or**
- Broaden Collect’s Fail gate: if opcode is any raw buffer load family and memory is Buffer/untyped, demote to IndirectBuffer instead of EXIT.

Prefer the Collect-side broaden so formatted/typed still Fail loudly.

**Risk:** IndirectBuffer materialization may not implement U16 loads yet — check `ResourceMaterialization` / SPIR-V emit. If missing, demote will pass tracking then die later → need V5-B.

### Layer V5-B — Host-lower Phi for buffer *loads* (preferred correctness)

Extend V3/V4/`#1224` Phi→host Select lowering so it applies when the Phi feeds:

- `LoadBufferU32` / `U32x2` / `U32x3` / `U32x4`
- `LoadBufferU16` / `U8` (after Convert)
- (already) `StoreBuffer*` / `BufferAtomic*`

Gates (reuse V4/#1224, tighten for loops):

1. Uniform 2-arm diamond; branch cond + arms are uniform runtime values (no LaneId).
2. **Loop policy:** if Phi is loop-invariant (defined outside loop / same value on all backedges), allow even when the *shader* has other loops. Reject only when the Phi itself is carried across a backedge that the load observes.
3. Writes-before: keep V4 “branch cond doesn’t read memory” / no shader write before incoming values.
4. Multi-use Phi OK (this shader’s `%141` has uses=7).

Emit host `SelectU32` (+ cloned `BitwiseOr32` flag) for descriptor planning; leave GPU Phi SSA unchanged (same as V3).

### Tests (`tests/ResourceTrackingTests.cpp`)

1. **Positive:** uniform Phi\|`0x40000` → `LoadBufferU16` / `LoadBufferU32x2` — PlanAndTrack succeeds; handle becomes host binding; no IndirectBuffer unless V5-A.
2. **Positive:** same Phi with shader-level other loops but Phi loop-invariant — must pass (Ace Combat shape).
3. **Negative:** LaneId-dependent Phi → still Fail with GPU-selected text.
4. **Negative:** Phi carried on backedge that the load sits in → still Fail.
5. Mutation: without V5-B, U16 case fails like U59’s message.

### Out of scope for V5

- Jetsku software RT / Astro Bot
- Overlap allow (V4B)
- Skipping EXIT entirely (`#1005` style) unless V5-A/B both fail

## Suggested branch layout

```text
descriptor-fix-v4b   ← Bot1 (BDA writes + overlap)
        │
        └── descriptor-fix-v5   ← V5-A and/or V5-B + README-V5 + VALIDATION-NOTES
```

Workflow: copy V4B’s `build-windows.yml` contents; artifact `KytyPS5-Descriptor-Fix-V5-Windows-x64`.

## Hand-off

Bot2 cannot `gh` push (no auth). @Bot1: create `descriptor-fix-v5` from `descriptor-fix-v4b` tip, add `descriptor-fix-v5/V5-RESEARCH.md` (this file) + stub `README-V5.txt` / empty patch until V4B log confirms the wall, then implement V5-B first (U16 load Phi), V5-A only if materialization lacks U16 IndirectBuffer.
