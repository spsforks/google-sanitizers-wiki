# Overview
MTE is an ISA extension (part of ARMv8.5) that introduces two types of tags:
* Allocation tags, 4 bits for each 16-byte granule of memory.
* Address tags, 4 bits in the upper byte of a memory address.

Most memory access instructions compare address and allocation tags, and generate an exception when they do not match.

# Stack instrumentation

LLVM can apply MTE to check memory safety of stack allocations. This is done by instrumenting generated code with MTE instructions to update allocation tags of individual stack variables.

Due to alignment requirements, all checked variables grow in size up to the next multiple of 16.

Please note that this is a work in progress, and some of the code snippets below may generate less efficient code with the current ToT LLVM.

### A basic example

`{ int x; use(&x); }`

    irg  x0, sp
    stg  x0, [x0]
    bl   use
    stg  sp, [sp]

- `IRG Xd, Xn` copies `Xn` to `Xd`, replacing the address tag with a randomly generated one.
- `STG Xd, [Xn]` updates allocation tag for `[Xn, Xn + 16]` to the address tag of `Xd`.

The second `STG` reverts the allocation tag back to match the address tag in `SP` register.

### Zero initialization

`{ int x = 0; use(&x); }`

    irg  x0, sp
    stzg x0, [x0]
    bl   use
    stg  sp, [sp]

- `STZG Xd, [Xn]` writes zero to `[Xn, Xn + 16]` and updates the allocation tag to the address tag of `Xd`.
- `ST2G` and `STZ2G` update two memory granules (i.e. 32 bytes) at once, with or without zero initialization.

Zero initialization and tagging has no code size overhead on top of just tagging.

### Value initialization

    {
      int x = 42;
      use(&x);
    }

    irg  x0, sp
    mov  w8, #42
    stgp x8, xzr, [x0]
    bl   use
    stg  sp, [sp]

- `STGP Xt, Xt2, [Xn]`, similar to `STP`, writes a pair of registers to memory at `[Xn, Xn + 16]` and updates allocation tag to match `Xn`.

For variables with non-zero initializers, setting allocation tags can often be done with little or no code size overhead with `STGP` instruction.

### Multiple allocations

    {
      int x, y;
      use(&x, &y);
      use(&x, &y);
    }

    irg  x19, sp
    addg x1, x19, #16, #1
    mov  x0, x19
    stg  x19, [x19]
    stg  x0, [x0]
    bl   use
    addg x1, x19, #16, #1
    mov  x0, x19
    bl   use
    st2g sp, [sp]

- `ADDG Rd, Rn, #imm1, #imm2` copies `Rn` to `Rd`, adding `#imm1` to the address part and `#imm2` to the tag part (modulo 16).
- `ST2G  Rd, [Rn]` is similar to `STG`, but updates allocation tags for 2 granules of memory (i.e. 32 bytes at once).

Instead of generating independent tags for `x` and `y` with two `IRG` instructions, we generate a single **base tagged pointer** for the function, and then use `ADDG` to obtain a tagged address of the second local variable. This trick helps reduce register pressure in functions with multiple local variables: in the worst case, a function with N independently tagged variables would require N live registers just to store their addresses. The use of the base tagged pointer allows materializing tagged address of any local variable with a single arithmetic instruction.


### SP-based load/store instructions

    {
      int x;
      use(&x);
      return x;
    }

    irg  x0, sp
    stg  x0, [x0]
    bl   use
    ldr  w0, [sp]
    stg  sp, [sp]

- All load and store instructions with SP base register and immediate offset do not check tags.

This allows the compiler to avoid allocating a callee-saved register for the tagged address of `x` in the above code snippet, and load the value of `x` after the call as if it has not been tagged.

SP-based loads and stores with immediate offset are assumed to be safe because they only appear in the code accessing local variables on the stack of the current function, and their correctness can be verified statically.
