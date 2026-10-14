KytyPS5 descriptor fix V8 (experimental) = V7 + one change.

Crash fixed: Ace Combat 8 (PPSA24913) loading campaign under V7 aborted with
  unsupported queueEndOfPipeAction packet for address patch: 0x00000000
  in src/libs/agc.cpp:3960
The printed 0x00000000 is the packet HEADER (cmd[0]), not the label address:
the game called sceAgcQueueEndOfPipeActionPatchAddress on a command slot that
holds no packet yet (all-zero dword 0).

V8 change (v8-on-v7.patch, src/libs/agc.cpp only):
  AgcQueueEndOfPipeActionPatchAddress: if cmd[0] == 0, write nothing and return
  OK. Logs one "[V8] queueEndOfPipeAction address patch on zero packet ignored"
  line per unique cmd address (max 64 distinct sites; later hits are counted
  in "total=" but not logged), with cmd, address and dw0..dw3.
  RELEASE_MEM / EVENT_WRITE_EOP patching unchanged; any other NON-zero
  unknown header still EXITs as before. A patch function executes no GPU
  work, so no interrupt/fence/signal semantics are lost here; they come from
  whatever packet is later executed.
  No upstream/fork (KytyPS5, InoriRus/Kyty, NeySlim, Jetsku, MagnusPS5,
  SuperMedo, AnyPS5, sharpemu) handles a zero header; sharpemu returns
  INVALID_ARGUMENT, AnyPS5 validates and errors. Original code, not a port.

Risk: if the game expects the label write to be patched into a packet it
copies in afterwards, that EOP will write to the old/zero address and a
later wait on the label may hang. Look for [V8] lines in the log.

Patch SHA256 (LF): see build-windows-v8.yml v8_expected.
Apply order on 7b9997b: descriptor-v5.patch, upstream-pr1194, v7-on-v6.patch, v8-on-v7.patch.
