# Homework 4: Exploring Compilation Units in C

**Student:** Sargis Vardanyan

**Course:** Operating Systems

**GitHub repository:** https://github.com/SargisVardanian/aua-operating-systems-homework4

## 1. Environment

All compilation and inspection activities were performed on the course Ubuntu server in:

```text
/home/sargisv/homework4_compilation_units
```

Environment:

```text
Ubuntu 24.04, Linux 6.8.0-124-generic, x86-64
gcc (Ubuntu 13.3.0-6ubuntu2~24.04.1) 13.3.0
```

The complete command history captured on the server is available in [`outputs/command_history.txt`](outputs/command_history.txt).

## 2. Source Files

### `math_utils.h`

```c
#ifndef MATH_UTILS_H
#define MATH_UTILS_H

int square(int value);

#endif
```

The header declares the public interface of the utility compilation unit. The include guard prevents multiple inclusion in one translation unit.

### `math_utils.c`

```c
#include "math_utils.h"

int square(int value)
{
    return value * value;
}
```

This source file defines the `square` function.

### `main.c`

```c
#include <stdio.h>

#include "math_utils.h"

int main(void)
{
    int value = 7;
    printf("%d squared is %d\n", value, square(value));
    return 0;
}
```

This source file defines `main` and references `square` and the C library function `printf`.

## 3. Separate Compilation and Linking

Commands executed on the server:

```bash
gcc -c main.c
gcc -c math_utils.c
gcc main.o math_utils.o -o square_prog
./square_prog
```

Program output:

```text
7 squared is 49
```

`gcc -c` performs preprocessing, compilation, and assembly but stops before linking. It creates relocatable object files. The final `gcc` command invokes the linker, combines both object files with startup code and library references, resolves symbols, and creates the executable.

The `file` command identified the products as:

```text
main.o:       ELF 64-bit LSB relocatable, x86-64
math_utils.o: ELF 64-bit LSB relocatable, x86-64
square_prog:  ELF 64-bit LSB PIE executable, x86-64, dynamically linked
```

## 4. Symbol Analysis with `nm`

### `nm main.o`

```text
0000000000000000 T main
                 U printf
                 U square
```

`T main` means that `main` is defined in the text/code section of `main.o`. `U printf` and `U square` are undefined references that must be resolved later.

### `nm math_utils.o`

```text
0000000000000000 T square
```

`math_utils.o` defines `square` in its `.text` section and has no unresolved application-level function references.

### Important symbols from `nm square_prog`

```text
0000000000001060 T _start
0000000000001149 T main
0000000000001188 T square
                 U printf@GLIBC_2.2.5
```

The linker resolved the `square` reference by combining `main.o` and `math_utils.o`. Both functions now have final virtual addresses. `printf` remains an undefined dynamic symbol because it is provided at runtime by glibc. The executable also contains startup, initialization, finalization, GOT, and runtime-support symbols that were not present in the two source object files.

Complete outputs:

- [`nm main.o`](outputs/nm_main_o.txt)
- [`nm math_utils.o`](outputs/nm_math_utils_o.txt)
- [`nm square_prog`](outputs/nm_square_prog.txt)

## 5. Assembly Analysis with `objdump`

### `main.o`

The disassembly contains `main`, but the two call instructions have placeholder displacements:

```text
0000000000000000 <main>:
  18: e8 00 00 00 00        call 1d <main+0x1d>
  ...
  33: e8 00 00 00 00        call 38 <main+0x38>
```

The object file does not yet know the final addresses of `square` or `printf`. Its relocation table identifies what the linker must patch:

```text
OFFSET  TYPE            VALUE
19      R_X86_64_PLT32  square-4
27      R_X86_64_PC32   .rodata-4
34      R_X86_64_PLT32  printf-4
```

### `math_utils.o`

The implementation of `square` is already complete and uses `imul` to multiply the integer by itself:

```text
0000000000000000 <square>:
   0: f3 0f 1e fa           endbr64
   4: 55                    push   %rbp
   5: 48 89 e5              mov    %rsp,%rbp
   8: 89 7d fc              mov    %edi,-0x4(%rbp)
   b: 8b 45 fc              mov    -0x4(%rbp),%eax
   e: 0f af c0              imul   %eax,%eax
  11: 5d                    pop    %rbp
  12: c3                    ret
```

### `square_prog`

After linking, the call from `main` to `square` has been resolved to a real target:

```text
1161: e8 22 00 00 00        call 1188 <square>
...
117c: e8 cf fe ff ff        call 1050 <printf@plt>
```

The local `square` function is called directly. The external `printf` function is called through the Procedure Linkage Table (`.plt`), which supports dynamic linking. The executable also contains `_start`, initialization/finalization routines, PLT code, and compiler/runtime support code, so its disassembly is much larger than either object file.

Complete outputs:

- [`objdump -d main.o`](outputs/objdump_main_o.txt)
- [`objdump -d math_utils.o`](outputs/objdump_math_utils_o.txt)
- [`objdump -d square_prog`](outputs/objdump_square_prog.txt)
- [`main.o` relocation records](outputs/objdump_relocations_main_o.txt)

## 6. ELF Header Analysis with `readelf -h`

| Property | `main.o` | `math_utils.o` | `square_prog` |
|---|---:|---:|---:|
| ELF class | ELF64 | ELF64 | ELF64 |
| Architecture | x86-64 | x86-64 | x86-64 |
| Type | `REL` | `REL` | `DYN` (PIE executable) |
| Entry point | `0x0` | `0x0` | `0x1060` |
| Program headers | 0 | 0 | 13 |
| Section headers | 14 | 12 | 31 |

The object files are relocatable units, so they do not have an executable entry point or loadable program headers. Ubuntu's GCC creates a position-independent executable by default; therefore `square_prog` has ELF type `DYN` rather than the older non-PIE `EXEC` type. It has an entry point and program headers that tell the loader how to map it into memory.

Complete outputs:

- [`readelf -h main.o`](outputs/readelf_h_main_o.txt)
- [`readelf -h math_utils.o`](outputs/readelf_h_math_utils_o.txt)
- [`readelf -h square_prog`](outputs/readelf_h_square_prog.txt)

## 7. Section Analysis with `readelf -S`

### Sections in the object files

Both object files contain sections such as:

- `.text` — machine instructions;
- `.data` — initialized writable global/static data;
- `.bss` — zero-initialized or uninitialized global/static data;
- `.eh_frame` — stack-unwinding metadata;
- `.symtab` — full symbol table;
- `.strtab` and `.shstrtab` — symbol and section-name strings;
- `.comment` and GNU note sections — compiler/platform metadata.

`main.o` additionally has:

- `.rodata`, containing the `printf` format string;
- `.rela.text`, containing relocations for `square`, `printf`, and `.rodata`;
- `.rela.eh_frame`, containing unwind-information relocation data.

`math_utils.o` has no `.rodata` and no `.rela.text` because its function requires no string constant and no unresolved function/data address. It only has `.rela.eh_frame` for unwind metadata.

### Additional sections in the executable

The linker combines compatible input sections and adds the structures required for loading and dynamic linking. `square_prog` contains 31 sections, including:

- `.interp` — path to the dynamic loader;
- `.dynsym` and `.dynstr` — dynamic symbols and strings;
- `.gnu.hash` and `.gnu.version*` — dynamic symbol lookup/version metadata;
- `.rela.dyn` and `.rela.plt` — runtime relocations;
- `.init`, `.fini`, `.init_array`, and `.fini_array` — initialization/finalization support;
- `.plt`, `.plt.got`, and `.plt.sec` — dynamic function-call stubs;
- `.dynamic` and `.got` — dynamic-loader metadata and address tables;
- merged `.text`, `.rodata`, `.data`, and `.bss` sections;
- `.symtab`, `.strtab`, and `.shstrtab`.

Complete outputs:

- [`readelf -S main.o`](outputs/readelf_S_main_o.txt)
- [`readelf -S math_utils.o`](outputs/readelf_S_math_utils_o.txt)
- [`readelf -S square_prog`](outputs/readelf_S_square_prog.txt)

## 8. Role of the Linker

The linker performs the following work:

1. Reads symbols and sections from `main.o` and `math_utils.o`.
2. Finds that `main.o` needs `square` and resolves it using the definition in `math_utils.o`.
3. Merges input sections into the output ELF layout.
4. Assigns final virtual addresses to functions and data.
5. Applies relocations, replacing placeholder offsets in `main.o` with valid addresses.
6. Adds startup code and metadata needed by the runtime loader.
7. Records the dynamic dependency on `printf`/glibc and creates PLT/GOT entries.
8. Produces a runnable PIE executable with entry point `_start` at `0x1060`.

Without the linking step, `main.o` cannot run because `square` and `printf` are unresolved and no executable entry-point/loading structure exists.

## 9. Conclusion

The experiment demonstrates that each `.c` file is compiled independently into a relocatable object file. Headers provide declarations but do not define a separate runtime compilation unit. `nm` reveals which symbols each object defines or requires, `objdump` shows incomplete call sites and later resolved executable code, and `readelf` shows the transition from small relocatable ELF objects to a loadable dynamically linked PIE executable. The linker is responsible for combining the units, resolving local symbols, applying relocations, arranging sections, and preparing the final process image.

All required source, object, executable, and output files remain on the course server in `/home/sargisv/homework4_compilation_units`.
