# Assembly Coding Skills — x86/x64 & ARM

## Reading Assembly
Registers → Instructions → Memory → Stack → Calling conventions

## x86-64 Registers
```
RAX RBX RCX RDX — general purpose (A=accumulator, B=base, C=counter, D=data)
RSI RDI          — source/dest index (also args 2 & 1 in Linux)
RSP RBP          — stack pointer, base pointer
R8–R15           — extended general purpose

Argument registers (Linux x64 calling convention):
RDI, RSI, RDX, RCX, R8, R9 — first 6 args
RAX — return value

Sub-registers:
RAX (64-bit) → EAX (32) → AX (16) → AH/AL (8)
```

## Core Instructions
```nasm
; Data movement
MOV rax, 42        ; rax = 42
MOV [rbp-8], rax   ; memory[rbp-8] = rax
LEA rax, [rbp-8]   ; rax = address of rbp-8 (load effective address)
PUSH rax           ; push to stack
POP rbx            ; pop from stack
XCHG rax, rbx      ; swap

; Arithmetic
ADD rax, rbx       ; rax += rbx
SUB rax, 1         ; rax -= 1
IMUL rax, rbx      ; rax *= rbx (signed)
IDIV rbx           ; rax = rdx:rax / rbx, rdx = remainder
INC rax            ; rax++
DEC rax            ; rax--

; Bitwise
AND rax, rbx       ; bitwise AND
OR  rax, rbx       ; bitwise OR
XOR rax, rax       ; rax = 0 (fastest way to zero a register)
SHL rax, 3         ; rax <<= 3 (multiply by 8)
SHR rax, 1         ; rax >>= 1 (divide by 2)
NOT rax            ; bitwise complement

; Comparison & jumps
CMP rax, rbx       ; sets flags (does not store result)
TEST rax, rax      ; AND without storing (checks if zero)
JZ  label          ; jump if zero flag set
JNZ label          ; jump if not zero
JL  label          ; jump if less (signed)
JG  label          ; jump if greater (signed)
JMP label          ; unconditional jump
CALL fn            ; push RIP, jump to fn
RET                ; pop RIP, return
```

## Hello World (Linux x64 NASM)
```nasm
; hello.asm — assemble: nasm -f elf64 hello.asm && ld -o hello hello.o
section .data
    msg db "Hello, World!", 10   ; string + newline
    len equ $ - msg              ; length

section .text
global _start

_start:
    ; write(1, msg, len)
    mov rax, 1          ; syscall: sys_write
    mov rdi, 1          ; fd: stdout
    mov rsi, msg        ; buf: pointer to msg
    mov rdx, len        ; count
    syscall

    ; exit(0)
    mov rax, 60         ; syscall: sys_exit
    xor rdi, rdi        ; status: 0
    syscall
```

## Function Call (C ABI)
```nasm
; int add(int a, int b)  — a in EDI, b in ESI, return in EAX
add_ints:
    push rbp
    mov  rbp, rsp
    ; a = edi, b = esi
    mov  eax, edi
    add  eax, esi
    pop  rbp
    ret

; Calling it
mov edi, 10         ; arg1 = 10
mov esi, 32         ; arg2 = 32
call add_ints       ; result in EAX = 42
```

## Stack Frame Layout
```
High addr  | caller's frame  |
           | return address  |  ← pushed by CALL
           | saved RBP       |  ← push rbp
RBP →      | local var 1     |  [rbp-8]
           | local var 2     |  [rbp-16]
RSP →      | ...             |
Low addr
```

## ARM64 (AArch64) Basics
```arm
// ARM64 registers
// X0-X7:  function arguments / return values
// X8-X15: scratch / temp
// X19-X28: callee-saved
// X29 = FP (frame pointer), X30 = LR (link register), SP = stack pointer

// Hello World (ARM64 Linux)
.section .data
msg:    .asciz "Hello ARM64!\n"
len = . - msg

.section .text
.global _start
_start:
    mov x8, #64         // sys_write
    mov x0, #1          // stdout
    adr x1, msg         // buf address
    mov x2, #len        // length
    svc #0              // syscall

    mov x8, #93         // sys_exit
    mov x0, #0
    svc #0
```

## Inline Assembly in C
```c
#include <stdint.h>

// GCC inline assembly
static inline uint64_t rdtsc(void) {
    uint32_t lo, hi;
    __asm__ volatile (
        "rdtsc"
        : "=a"(lo), "=d"(hi)
    );
    return ((uint64_t)hi << 32) | lo;
}

// CPUID
void get_cpuid(uint32_t leaf, uint32_t *eax, uint32_t *ebx,
               uint32_t *ecx, uint32_t *edx) {
    __asm__ volatile (
        "cpuid"
        : "=a"(*eax), "=b"(*ebx), "=c"(*ecx), "=d"(*edx)
        : "a"(leaf)
    );
}

// Atomic CAS (compare-and-swap)
int cas(int *ptr, int old_val, int new_val) {
    int result;
    __asm__ volatile (
        "lock cmpxchg %2, %1"
        : "=a"(result), "+m"(*ptr)
        : "r"(new_val), "0"(old_val)
        : "memory"
    );
    return result == old_val;
}
```

## Reading Disassembly (GDB / objdump)
```bash
# Disassemble binary
objdump -d -M intel binary | less
objdump -d -M intel binary | grep -A20 "<main>"

# GDB with Intel syntax
gdb ./binary
(gdb) set disassembly-flavor intel
(gdb) disassemble main
(gdb) break *0x401050    # break at address
(gdb) x/10i $rip         # show next 10 instructions
(gdb) info registers     # show all registers
(gdb) x/8gx $rsp         # show 8 qwords at stack pointer
```

## Performance Tips
- `XOR reg, reg` — fastest way to zero (1 cycle, no dependency)
- `LEA` for multiply/add without flags (e.g., `LEA rax,[rax+rax*4]` = ×5)
- `CMOV` — branchless conditional move, avoids branch misprediction
- Align loops to 16-byte boundaries (`ALIGN 16` in NASM)
- Prefer register operands over memory; memory access is 100× slower
