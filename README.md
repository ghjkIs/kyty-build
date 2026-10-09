# kyty-build

Windows GitHub Actions kits for experimental KytyPS5 descriptor-fix builds.

## Current: descriptor fix V3 (experimental)

Branch: [`descriptor-fix-v3`](https://github.com/ghjkIs/kyty-build/tree/descriptor-fix-v3)

Target crash (Ace Combat 8 / `PPSA24913`): compute shader
`hash=0x1d81bb2b74faf456` — `unsupported GPU-selected access: opcode=BufferAtomicOr32`.

V3 extends V2 so a **uniform** branch-merged (Phi) buffer address used by
**multiple** `BufferAtomic*` ops can be lowered to a host binding. Lane-dependent
Phis, writes before the diamond, and Phi diamonds inside loops are still rejected.
Not proven to boot Ace Combat 8.

### Build

1. Open [Actions](https://github.com/ghjkIs/kyty-build/actions) →
   **Build KytyPS5 descriptor fix V2 for Windows** → **Run workflow**.
2. Select branch **`descriptor-fix-v3`** (on this branch only,
   `build-windows.yml` runs the V3 kit; `main` still has V2).
3. When green, download artifact **`KytyPS5-Descriptor-Fix-V3-Windows-x64`**.

Details and patch SHA256: [`descriptor-fix-v3/README-V3.txt`](descriptor-fix-v3/README-V3.txt).

### Run (diagnostic)

1. Extract the ZIP into a **new** folder (do not overwrite a V2 install).
2. Run `Run-PPSA24913-Diagnostic.cmd` (edit the `GAME` path if needed).
3. If it still exits, keep `ace-combat-detail-v3.txt` and `ace-combat-log-v3.txt`.

### Patch

Apply **only** `descriptor-fix-v3/descriptor-v3.patch` to official KytyPS5
`7b9997baea0d385e9e8fc2bd3abeb02110bc4148`. It is cumulative (includes V2).
Do not stack V2 + V3.

```
SHA256: bbc01f37ce41f4a7c7f1e811ba769a987375402406d3aa07cbaae5d62f45e4bf
```

### Branches

| Branch | Build kit |
|--------|-----------|
| `main` | V2 |
| `descriptor-fix-v3` | V3 |

Validation notes: [`descriptor-fix-v3/VALIDATION-NOTES.txt`](descriptor-fix-v3/VALIDATION-NOTES.txt).
