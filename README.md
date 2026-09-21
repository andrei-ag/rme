# Russian_Mutant, Engine (RME) aka BAME II

**Metamorphic recompiler for x86 DOS code. Designed in 2000-2001.**

## What it is
- An engine that reassembles binary code while preserving semantics.
- Works at the x86 real-mode instruction level.

## What it does

Pipeline:

1. Disassembles the binary skeleton into instruction records.
2. Builds an IR (Intermediate Representation): `DNA_entry` → `mRNA_entry`.
3. Transforms instructions:
   - Replaces them with semantically equivalent variants.
   - Optionally injects Borland C++ 3.1-style junk code.
   - Eliminates the classic Get-IP sequence (`call $+5; pop ip_reg; sub ip_reg, offset`) by recompiling IP-relative memory references (`[ip_reg+offset]`) into absolute ones (`[offset]`). This shortens each such instruction by 1 byte.
4. Recalculates relative jumps and calls (`CALL`, `JNZ`, `LOOP`, `jmp`) whose offsets changed after mutation.
5. Emits the reassembled binary.
6. Optionally verifies the result via emulator and dumps debug info.

## Key features

- Full recompilation of the target code body, not just the decryptor.
- Mimicry of Borland C++ 3.1 code generation.
- Compiler-like modular architecture with dispatch tables for the disassembler and mutator, allowing easy extension with new instruction types and HLL-style junk templates (e.g., Borland C++ 3.1).
- Converts IP-relative references to absolute ones, removing the classic Get-IP trigger and saving 1 byte per instruction.
- Three modes:
  - **Polymorphic** — encrypts the body, mutates the decryptor.
  - **Metamorphic** — mutates the body without encryption.
  - **Full** — combines both.
- Integrated debug and validation subsystem (`DNA`/`mRNA` dumps, PASS/FAIL checks).
- Conceptually portable to 32/64-bit: the pipeline, IR model, and dispatch tables serve as a template, though the instruction layer (ModR/M, registers, opcodes) is 16-bit–specific and would need to be rewritten.

## Architecture
- `core/` — core, initialization, register selection.
- `build/` — code generation.
- `disasm/` — disassembler.
- `rna2dna/` — instruction translation.
- `rna2dna/borlandc/` — BC31 mimicry.
- `emul/` — emulator.
- `debug/` — debugging and dumps.

## The DNA/mRNA Metaphor

The naming in RME is not arbitrary — it mirrors the biological flow of genetic information and the process of metamorphic transformation.

In biology:

```
DNA → (transcription) → mRNA → (translation) → protein
```

In RME:

| Biology | RME | Role |
|---------|-----|------|
| **DNA** | `DNA_buffer` / `DNA_entry` | Stable representation of the original skeleton after disassembly |
| **Transcription** | `init_mRNA_entry` | Copies `DNA_entry` → `mRNA_entry`, adding mutation fields |
| **mRNA** | `mRNA_buffer` / `mRNA_entry` | Mutable working copy used for transformation |
| **Translation** | `rebuild_skeleton` | `mRNA` → new machine code in `temp_skeleton` |
| **Protein** | `temp_skeleton` | Functional output — the mutated skeleton |
| **Reverse transcription** | `rna2dna/` | Converts the working copy back into a stable "genome" |
| **Epigenetics** | `epigenetic_mutation` | Changes code *expression* without changing meaning |
| **Mutation** | `translate_*` handlers | Point substitutions of instructions |

### Why it matters

- **`DNA_entry`** describes an instruction abstractly (type, registers, target) — not as raw bytes. This is the invariant "genome" of the code.
- **`mRNA_entry`** is a transcript: it copies the DNA and adds fields like `@new_addr`, `@new_len`, and `@old_indx`, so the engine can mutate the working copy without touching the original.
- **`epigenetic_mutation`** is the most precise term: it changes the phenotype (machine code) while preserving the genotype (logic). This is the essence of metamorphism.
- **`rna2dna`** alludes to retroviruses, which use reverse transcription (RNA → DNA) to integrate into a host genome. For a metamorphic engine, this is both a technical and an ironic reference.

### Full cycle

```
┌─────────────────────────────────────────────────────────────────────┐
│  Input: binary skeleton                                             │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
            ┌───────────────────────────────────────┐
            │  1. Disassemble binary skeleton →     │
            │     instruction records (DNA)         │
            └───────────────────┬───────────────────┘
                                │
                                ▼
            ┌───────────────────────────────────────┐
            │  2. Build IR: NA_entry → mRNA_entry   │
            │     (transcription)                   │
            └───────────────────┬───────────────────┘
                                │
                                ▼
            ┌───────────────────────────────────────┐
            │  3. Transform instructions            │
            │                                       │
            │  ┌─────────────────────────────────┐  │
            │  │ 3a. Replace with semantically   │  │
            │  │     equivalent variants         │  │
            │  └─────────────────────────────────┘  │
            │                                       │
            │  ┌─────────────────────────────────┐  │
            │  │ 3b. Optionally inject           │  │
            |  │     HLL-style junk code         │  │
            │  └─────────────────────────────────┘  │
            │                                       │
            │  ┌─────────────────────────────────┐  │
            │  │ 3c. Eliminate Get-IP sequence   │  │
            │  └─────────────────────────────────┘  │
            └───────────────────┬───────────────────┘
                                │
                                ▼
            ┌───────────────────────────────────────┐
            │  4. Recalculate relative              │
            |     jumps/calls and offsets           │
            └───────────────────┬───────────────────┘
                                │
                                ▼
            ┌───────────────────────────────────────┐
            │  5. Emit reassembled binary           │
            └───────────────────┬───────────────────┘
                                │
                                ▼
            ┌───────────────────────────────────────┐
            │  6. Optionally verify                 │
            │     via emulator and dump debug info  │
            └───────────────────┬───────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────────┐
│  Output: mutated binary (decryptor + encrypted/mutated code + data) │
└─────────────────────────────────────────────────────────────────────┘
```

The engine never mutates the original. It transcribes, mutates, translates, and only then fixes the result via reverse transcription. The metaphor is not decoration — it is a precise model of the metamorphic process.

## Build and run
- Assembler: TASM, 16-bit, `.386`.
- `RME.ASM`, `RME2.ASM`, `RME3.ASM` — test wrappers.
- Assemble, run in DOS/DOSBox.
- Output: `rme_test.com`, `rmetest2.com`, `rmetest3.com`, `.dmp` dumps.
- Configuration: `RME.INC`, `RME2.INC`, `RME3.INC`.

## Main options
- `owner` — germ / test_program.
- `use_gen_decryptor` — generate decryptor.
- `mutate_code` — mutate body.
- `use_debug` — debug dump.
- `use_emul` — emulator.
- `use_borland_c_junk` — BC31-style junk.

## Status
Version 1.07a (alpha). Later versions are lost; a search is ongoing.

## Disclaimer
For educational and historical purposes only. Do not use for malicious purposes.

## Authors
© Copyleft 2000-2001. Andrei AG & MM.
