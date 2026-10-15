KytyPS5 descriptor fix V13 - EXPERIMENTAL DIAGNOSTIC BUILD (Ace Combat 8 / PPSA24913)

Layering (all on clean KytyPS5 7b9997baea0d385e9e8fc2bd3abeb02110bc4148):
  V5 parts -> PR1194 -> v7 -> v8 -> v9 -> v10 -> v11 -> v12 -> v13-on-v12.patch
  v13-on-v12.patch sha256 a582b6b4cbd452b131b29d9771ef5789a3183055bb69299ddefd846e8f4eafd3

Problem (V12 R1/R2/R3): after ~430 CS / 858 frames the emulator stops with
  "BufferCache: failed to read mapped guest image backing" (bufferCache.cpp:545).
ObtainBufferForImage read the image through TryReadSparseBacking, which fails when the range
is outside the guest address space, overlaps a private (flexible / private-committed) mapping,
or (before upstream 9bcf80bf) when the staging stream buffer cannot hold the image.

V13 contents:
 1. Port of upstream KytyPS5 9bcf80bf "graphics: share temporary staging for oversized image
    uploads" (bufferCache.cpp part): images larger than the staging ring use a one-time upload
    buffer instead of failing.
 2. Port of upstream 40692dfb "graphics: copy sparse buffer uploads through resident mappings"
    (bufferCache.cpp + kernel memory parts; tests not ported): new
    LibKernel::Memory::CopySparseMemory copies backing pages and resident private mappings,
    zero for reserved gaps. Buffer uploads (UploadCopies, stream path) use it.
    (Upstream main still EXITs at the image-backing site; no fork fix found to port for it.)
 3. V13 tolerance (KYTY_ZERO_UNMAPPED_IMAGE, default ON; 0/off/false restores the EXITs):
    - ObtainBufferForImage and both buffer-upload reads: if the full read fails, the range is
      re-read in 16 KiB pages (TryReadSparseBacking, then CopySparseMemory); unreadable pages are
      zero-filled and emulation continues, mirroring PR1194's "read unmapped as zero".
    - Logged once per unique (address, size, kind), max 512 sites:
        [V13] unreadable guest image|buffer backing zero-filled: caller=... addr= size=
              unreadable= (whole range|partial|page reads succeeded) runs= first_bad= last_bad_end=
              pages(0x4000)=[+off,+off) ... gpu_mapped= total_events=
        [V13]   image: guest_format= vk_format= type= tile_mode= extent=WxHxD pitch=
              bytes_per_block= samples= stencil= metadata_kind=
      Startup line: "[V13] zero-fill unreadable guest image/buffer backing ON|OFF".
    - TextureCache::MaterializeColorClear color-metadata reads (TryReadBacking EXITs) now log
      "[V13] unreadable color metadata ... fast-clear materialization skipped" and skip that slice.

Run scripts R1-R4 are unchanged from V12 except names/log files (v13-R*).
Please send ace-combat-log-v13-R*.txt; grep "[V13]" for the unreadable addresses.

Risks: zero-filled texels may render black/garbage; if the image is genuinely in memory the
emulator does not track, the real fix is elsewhere (memory mapping/aliasing). A crash may simply
move further along. CopySparseMemory memcpy's resident private mappings from host memory.
