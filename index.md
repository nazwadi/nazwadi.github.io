## Stack buffer overflow

### Definition
A Stack buffer overflow (also known as "smashing the stack") generally refers to copying data that is too large for a dynamically allocated buffer stored on the stack.  _It is dynamic because statically allocated variable are allocated at process load time on the data segment (for an ELF-formatted binary) and not at run time in a procedure's call stack_.  When a stack buffer overflow occurs, the excess data overwrites adjacent dynamically allocated memory in the current function's stack frame in the direction of the calling function's stack frame.  The specifics of how a stack buffer overflow works depends on the machine's architecture, operating system, and calling convention used (_cdecl_ and _stdcall_ are two popular ones). 

Since C does not have any built-in bounds checking, there are many standard library functions that copy data between buffers that are vulnerable to buffer overflows if the programmer does not implement bounds checking.  This is especially true with functions that copy or append strings.

### What is a stack frame? 

A stack frame, or stack, contains the parameters to a function, its local variables, and the data necessary to recover the previous stack frame, including the value of the instruction pointer at the time of the function call.  Every function (or procedure) in a program has its own stack frame.  The stack frame is built when the function is called, and torn down when the function returns.

The location where function parameters are located depends on the calling convention used.

Generally speaking, any data not explicitly stored on the heap (via a call to malloc) in a function will get stored in the stack frame - including the memory address (pointer) to malloc'd data.  In C, stack buffer overflows generally pertain to arrays.

### On Calling Conventions (or when function parameters are on the Stack)

#### x86 (32-bit Intel)
On x86 (32-bit) architecture using `cdecl` and `stdcall` calling conventions, we dynamically allocate local variables declared in a function by PUSHing them on the stack.  We pass parameters to a called function by PUSHing them on the stack before the `call` instruction (left-most parameter being the last one pushed on the stack before the function is called), and we return values from the function by POPing them from the stack into a register (EAX for integers and memory addresses, ST0 for floating point in `cdecl`, or just EAX in `stdcall`).  

Specifics on calling conventions, use of registers, and stack frame organization can be found in the [System V Application Binary Interface Intel386 Architecture Processor Supplement](https://www.sco.com/developers/devspecs/abi386-4.pdf).

#### x86-64 (64-bit Intel)
On x86_64 architecture, the change most relevant to this discussion is that function parameters are no longer pushed on the stack for the first six parameters but rather stored in general-purpose registers (`%rdi`, `%rsi`, `%rdx`, `%rcx`, `%r8`, `%r9` in that order) for non-floating point values.  Floating point parameter values will be stored in `%xmm0` - `%xmm7`.  If there are more than six parameters, the additional parameters are pushed onto the stack in _reversed (right-to-left)_ (i.e. last parameter first).

Specifics on calling conventions, use of registers, and stack frame organization can be found in the [System V Application Binary Interface (ABI) x86-64](https://www.intel.com/content/dam/develop/external/us/en/documents/mpx-linux64-abi.pdf).

### Registers Relevant to Stack Overflows
- `SP` - This is the stack pointer, and it points to the top of the stack (on x86 this is the lowest numerical memory address).  Its name depends on the word-size of the CPU architecture, but on x86 (32-bit word) machines it will be `ESP`.  On x86_64 (64-bit word) machines it will be `RSP`.
- `IP` - Instruction Pointer.  On x86 it is known as `EIP`, and on x86_64 it is known as `RIP`.  This register points to the current instruction being executed.  On Intel CPUs, the contents of the instruction register is pushed to the stack when the `call` instruction is executed.  This is an important detail in understanding how stack-based buffer overflow exploits work. Essentially, if an attacker can overflow a buffer (which will overwrite the stack in the direction of the calling function's stack frame), they can execute arbitrary code by placing the memory address of its start point at the location on the stack where the saved instruction pointer should be.  When the called function returns, the stack frame will get torn down and the saved instruction pointer (or the new value overwriting its location due to the buffer overflow) will be popped back off the stack into the instruction pointer register.
- `BP` - This is the base pointer.  Also known as `EBP` (x86) and `RBP` (x86_64).  The base pointer is normally set to the value in `ESP` at the start of a function (in the function prologue when the called is setting up the stack frame).  Compilers normally generate stack frames where parameters and local variables are referenced as offsets from the base pointer.  Due to the way the stack grows on Intel cpus, function parameters have positive offsets from `ebp` and local variables have negative offsents from `ebp`.
    - `FP` - Many compilers use a frame pointer which points to a fixed location in a frame; also known as a local base pointer (`BP`).  This is usually just another name for `BP`.  Without using a frame pointer, function parameters and local variables would have to be referenced as offsets from the stack pointer which will change as data is pushed and popped from the stack.


example1.c:
```c
void myfun(int a, int b) {
   char buffer1[5];
   char buffer2[10];
}

void main() {
  int value = 5;
  myfun(3, value);
}
```

The stack is either implemented growing down (towards lower memory addresses), or growing up (towards higher memory addresses).  On Intel, Motorola, MIPS, and SPARC architectures the stack grows down. A typical function call might look like the following (using Intel x86 asm format):

```asm
push ESP              // Save the stack pointer of the calling stack frame
push dword [EBP+20]   // Save the value at this address as the second function parameter
push 3                // Save the integer 3 as the first function parameter
call myfun            // Save the current instruction pointer, then call the function, "myfun"
```

This corresponds to a function `myfun(3, value)`.  Prior to pushing the parameters on the stack, you see we've saved the position of the stack pointer (ESP). To reiterate what is described in the comments above, the instruction pointer, `EIP`, is pushed onto the stack as part of the `call` instruction.

For the purpose of this response, I will be describing Intel x86 cpu architecture.  Due to the way our stack grows, actual parameters have positive offsets and local variables have negative offsets from FP.

```
bottom of                                                            top of
memory                                                               memory
           buffer2       buffer1   sfp   ret   3     value
<------   [            ][        ][    ][    ][    ][    ]
	   
top of                                                            bottom of
stack                                                                 stack
```

### Function Prologue
The function prologue represents the first few instructions in a function, and its purpose is to set up the new stack frame.

The first thing a function does when called is to save the previous frame pointer (so it can be restored when the function exits).  Then it copies the stack pointer into the frame pointer to create the new frame pointer, and advances the stack pointer to reserve space for the local variables.

```nasm
push %ebp       // save the previous frame pointer
mov %esp, %ebp // Set the base pointer equal to the stack pointer
sub $20, %esp // dynamically grow the stack frame by 56 bytes
```

Making `ebp` equal to `esp` at function entry, you will have a fixed, relative pointer inside the stack, that will not change for the lifetime of your function, and you will able to access parameters and locals as (fixed) positive and (fixed) negative offsets, respectively, to `ebp`.

Memory can only be addressed in multiples of the word size.  For x86, this is 4 bytes, or 32 bits.  For x86-64, this is 8 bytes, or 64 bits.  Thus our 5 byte buffer1 will be allocated 8 bytes.  The 10 byte buffer2 would be allocated 12 bytes.

### Function Epilogue
The function epilogue represents the last few instructions in a function, and its purpose is to tear down the stack frame of the current function that is returning to the calling function.

The epilogue will pop the saved frame pointer from the calling function back into the `%ebp` register, and it will pop the saved return address back into the `%eip` register.