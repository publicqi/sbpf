# SBPF Version Differences (v0–v4)

This summarizes the behavioral/encoding differences between SBPF versions **v0, v1, v2, v3, v4** as implemented in this repository (see `src/program.rs` plus `doc/bytecode.md`, `doc/syscalls.md`, `doc/relocations.md`).

## Quick matrix

| Feature / behavior | v0 | v1 | v2 | v3 | v4 |
|---|---|---|---|---|---|
| Release note (in code) | legacy | SIMD-0166 | SIMD-0174, SIMD-0173 | SIMD-0178, SIMD-0189, SIMD-0377 | SIMD-0177 |
| ELF `e_flags` value | `0` | `1` | `2` | `3` | `4` |
| Manual stack bump (`add64 r10, imm` allowed) | no | yes | yes | yes | yes |
| Automatic stack bump on `call`/`callx` | yes | no | no | yes | yes |
| Initial `r10` (frame pointer) | `MM_STACK_START + stack_frame_size` | `MM_STACK_START + stack_len` | `MM_STACK_START + stack_len` | `MM_STACK_START + stack_frame_size` | `MM_STACK_START + stack_frame_size` |
| `callx` target register encoded in | `imm` | `imm` | `src` | `dst` | `dst` |
| `call imm` syscall vs internal call disambiguation | none (hash lookup wins) | none | none | `src=0` syscall, `src=1` internal | same |
| PQR instruction class (`lmul/udiv/urem/sdiv/srem/...`) | no | no | **yes** | no | no |
| JMP32 instruction class (`jeq32/jgt32/...`) | no | no | no | **yes** | yes |
| `lddw` (2-slot immediate load) | yes | yes | **no** | yes | yes |
| `hor64` (high-half immediate OR) | no | no | **yes** | no | no |
| `le` instruction | yes | yes | **no** | yes | yes |
| `neg32` / `neg64` | yes | yes | **no** | yes | yes |
| `sub{32,64} dst, imm` operand order | `dst - imm` | `dst - imm` | `imm - dst` | `dst - imm` | `dst - imm` |
| `mov32 dst, src` extension | zero-extend | zero-extend | **sign-extend** | zero-extend | zero-extend |
| `add32/sub32/...` result extension | sign-extend for `add32/sub32` family | same | **zero-extend for `add32/sub32` family** | sign-extend for `add32/sub32` family | same |
| Memory load/store opcodes | legacy `ldx*/st*/stx*` classes | same | moved to `LD_{1,2,4,8}B_*` + `ST_{1,2,4,8}B_*` | legacy classes | legacy classes |
| ELF header parsing | lenient | lenient | lenient | **strict** | **strict** |
| Runtime ELF relocations | supported/processed | supported/processed | supported/processed | expected absent | expected absent |
| rodata virtual address base | `MM_BYTECODE_START` (combined with code) | same | same | **`MM_RODATA_START` (separate region)** | same |
| Memory mapping mode | unaligned allowed | unaligned allowed | unaligned allowed | unaligned allowed | **aligned-only** |

## Version-by-version details

### v0 (legacy)

- Stack
  - No manual stack frames: `r10` is not writable by ALU ops (only by store instructions).
  - Automatic stack frame bump is enabled: `r10` is bumped when entering a callee (`call`/`callx`).
- ISA / encoding
  - No PQR class; no JMP32 class.
  - `lddw` is supported; `hor64` is not.
  - `le` and `neg{32,64}` are supported.
  - `callx` reads its target register from the instruction immediate.
  - `add32/sub32` results are sign-extended to 64-bit (implementation detail).
- Syscalls
  - “Traditional syscalls”: runtime relocation + murmur32 hash of symbol name, and no explicit internal/syscall disambiguation in the `call imm` encoding (see `doc/syscalls.md`).
- ELF / memory layout
  - Uses the lenient ELF loader; dynamic relocations are processed (see `doc/relocations.md`).
  - rodata shares the bytecode region base address (not mapped at `MM_RODATA_START`).

### v1 (SIMD-0166: dynamic stack frames)

- Stack
  - Enables manual stack frames: `add64 r10, imm` becomes legal, so programs can adjust the frame pointer explicitly.
  - Disables automatic stack frame bump on calls.
  - `r10` starts at the “top” of the stack region (`MM_STACK_START + stack_len`).
- ISA / encoding
  - Still no PQR class and no JMP32 class.
  - `callx` still reads its target register from the instruction immediate.

### v2 (SIMD-0174, SIMD-0173: arithmetic + encoding changes)

- ISA / encoding (major differences)
  - Enables PQR class (`lmul*`, `uhmul64`, `{u,s}{div,rem}*`, etc.) and repurposes opcode class `0x06` for PQR.
  - Disables `lddw` and enables `hor64` as the high-half immediate builder.
  - Disables `le` and `neg{32,64}`.
  - Changes `sub{32,64} dst, imm` to compute `imm - dst`.
  - Moves memory instructions into different opcode slots (via “moved memory instruction classes”), replacing legacy `ldx*/st*/stx*` opcodes with `LD_{1,2,4,8}B_*` and `ST_{1,2,4,8}B_*` / `STX_{1,2,4,8}B_*` style encodings.
  - Changes `callx` to read its target register from the instruction `src` field.
  - Changes result extension rules:
    - `mov32 dst, src` sign-extends (other versions zero-extend).
    - `add32/sub32` family uses a different 32→64 extension behavior than non-v2 (implementation detail).
- Syscalls / ELF
  - Still uses “traditional syscalls” and the lenient ELF loader with relocations.

### v3 (SIMD-0178, SIMD-0189, SIMD-0377: static syscalls + strict ELF + JMP32)

- Syscalls (static syscalls)
  - Adds explicit disambiguation in the `call imm` encoding:
    - `src=0` means “external syscall”
    - `src=1` means “internal call”
  - Enables ahead-of-time verification of syscall registration (see `doc/syscalls.md`).
- ELF / memory layout (strict)
  - Enables stricter ELF header validation.
  - rodata lives at `MM_RODATA_START` (0) and bytecode at `MM_BYTECODE_START` (region 1); ELF relocations are expected to be resolved ahead of time (see `doc/relocations.md`).
- ISA / encoding
  - Enables JMP32 (`jeq32/jgt32/...`) by using opcode class `0x06` for JMP32 (so PQR is not available).
  - `callx` reads its target register from the instruction `dst` field.
- Stack
  - Re-enables automatic stack frame bump on calls.
  - `r10` starts at `MM_STACK_START + stack_frame_size` (one frame allocated at entry).

### v4 (SIMD-0177)

- Inherits v3 behavior and additionally forces **aligned memory mapping**:
  - Unaligned memory mapping is unsupported for v4+, and `MemoryMapping::new*()` selects aligned mapping for v4 regardless of configuration.
