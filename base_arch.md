#MCPU Instruction Architechture

The *Minecraft Processing Unit* (MCPU) is a 16-bit Reduced/Complex Hybrid
Instruction Set computer that runs using the *MCPU Instruction Set* (MCPUIS).
The MCPU architechture holds some similarities to the MIPS architechture, but
has some significant differences, including built-in support for a stack.

In the MCPUIS, there are only 14 instructions (and space for 2 more). Each
instruction is encoded in a 16bit value that stores the instruction to execute,
**3** flags affecting it's interpretation, and **3** register pointers. There
are **8** public registers, and **1** hidden register.

**4** of the public registers can be used for any function, **2** are used for
maintaining the stack, **1** is used for storing and retrieving processor flags
(mostly arithmetic), and **1** is the zero register. The one hidden register
is the *Program Counter*, which cannot be directly addressed with normal
opcodes, but is modified and read by the **CJMP** opcode. After execution of
an intruction, the **PC** is incremented by 1 (except when the **V** flag is
set, see below).

Each instruction includes 3 register indexes. The first is called the
destination index, or **DD** index. This index determines where the result of
the operation will be stored. The next two indexes represent the operands
to the opcode, and they are referred to as **X1** and **X2**. **X1** always
represents a register index, but **X2** can sometimes represent an immediate
value instead. (See below.)

3 flags affect the execution of each instruction.
* The **V** flag indicates that the instruction is immediately followed by a
16-bit immediate value, and this flag is the main method by which to introduce
constants to opcodes. What each opcode does with this value depends on the
opcode, but in most cases, it is added or multiplied by the value of the second
operand register. Note that after executing an instruction with the **V** flag
set, the **PC** will increase by 2 instead of 1.
* The **S** flag indicates that all operands represent signed values instead of
unsigned values. This affects the arithmetic performed with them.
* The **M** flag indicates that the **X2** index is *not* a pointer to a
register, but should instead be taken as an immediate value. Because of the
limited storage space, this value can only be a number between 0 and 7
(-3...4 signed), however many constants in code fit within this range. Note as
well that one opcode (**AND**) will ignore this value if the M flag is set.
(Because doing otherwise would severly limit the functionality of the opcode.)

###MCPU Instruction Format (16-bits)
    IIII VSMD DD11 1222
    IIII VSM DDD 111 222
    I - Instruction
    V - Instruction includes value
    S - Signed Math
    M - Index 2 is Immediate Value
    D - Destination Register
    1 - Index 1
    2 - Index 2

###Registers (8 public + 1 private)
    ZZ - Zero Register                 (Data stored here is lost, always returns 0)
    AX - General / Accumulator
    BX - General / Base
    CX - General / Counter
    DX - General / Data                (Function call convention stores return values here, though this is not required.)
    SP - Stack Pointer                 (Modified by PUSH/POP ops.)
    BP - Base Pointer
    FG - Flags                         (Modified by Arithmetic ops)
    PC - Program Counter               (Hidden)

###Instructions (14 + 2 reserved)
    0000/0 - ADD    DD = X1 +  (X2 + VV) X2 may be ignored by setting it to immediate 0
    0001/1 - SUB    DD = X1 -  (X2 + VV) X2 may be ignored by setting it to immediate 0
    0010/2 - MUL    DD = X1 *  (X2 * VV) X2 may be ignored by setting it to immediate 1
    0011/3 - DIV    DD = X1 /  (X2 * VV) X2 may be ignored by setting it to immediate 1
    0100/4 - AND    DD = X1 &  (X2 & VV) If X2 is immediate, it is ignored
    0101/5 - OR     DD = X1 |  (X2 | VV) X2 may be ignored by setting it to immediate 0
    0110/6 - XOR    DD = X1 ^  (X2 ^ VV) X2 may be ignored by setting it to immediate 0
    0111/7 - CJMP   IF(DD)  JMP(X2 + VV) DD and X1 are used as immediate arguments, see the CJMP section for more.
    1000/8 - LSHF   DD = X1 << (X2 + VV) X2 and VV are always unsigned.
    1001/9 - RSHF   DD = X1 >> (X2 + VV) If signed, MSB will be extended.
    1010/A - LOAD   DD = RAM[X1+X2+VV]
    1011/B - STOR   RAM[X1+X2+VV] = DD
    1100/C - PUSH   DD = RAM[--SP] = X1 + (X2 + VV)
    1101/D - POP    DD = RAM[SP++] + X1 + (X2 + VV)
    1110/E - resv
    1111/F - EXTD   Represents an extended instruction. These are unsupported at this time.

###PseudoInstructions
    SET    DD =        X2 + VV    (ADD DD ZZ X2 VV)
    INC    DD =        DD + 1     (ADD DD DD _1   )
    DNC    DD =        DD - 1     (ADDs DD DD -1  ) //s indicates the signed flag is set.
    ACUM   AX = X1 +  (X2 + VV)   (ADD AX X1 X2 VV)
    NOP                           (ADD ZZ ZZ ZZ   ) //there are alot of variants of this.
    CMP    Fill flags register    (SUB ZZ X1 X2 VV)
    JMP    Unconditional Jump     (CJMP 7 X1 X2 VV) //7 is a constant, X1 can have bit 1 set for absolute jump

###PC Register
The **PC** register is normally unavailable for direct use by code.
In the case that it must be accessed, the following code can be used:

    //Write to PC
    CJMP 7 1 VV    //PC = VV, this is an unconditional absolute jump to VV.
    //Read from PC
    CJMP 7 2 1     //PUSH PC+1 onto stack, then unconditionally relative jump to +1.
    POP X1 ZZ -1   //POP PC-1 from stack into register X1 (you may skip this instruction to keep it on the stack.)

###CJMP Instruction
**DD** is the comparison type.

    #   UNSIGNED      / SIGNED
    0 - ZERO          /
    1 - GREATER       /
    2 - LESS          /
    3 - OVERFLOW      / CARRY
    4 - NEGATIVE      / 
    5 - GREATER|EQUAL / SIGNED GE
    6 - LESS|EQUAL    / SIGNED LE
    7 - ALWAYS        /

**X1** is a flags variable

    bit 0 - clear = relative jump, set = absolute jump
    bit 1 - clear = normal, set = test is negated
    bit 2 - clear = normal, set = jmp is call (PUSH PC+1 is called before a successful jmp)

**X2** functions as normal (may be register or immediate)
**VV** functions as normal

Note that **CJMP** functions by checking the **FG** register flags. To set these flags
properly, you must use the **SUB** instruction, where **X1** is the left hand operand,
and **X2** is the right hand operand. **DD** does not matter, and may be **ZZ**.

For instance, for the conditional jump **X1** >= **X2**:

    SUB ZZ X1 X2 //set FG register
    CJMP 5 0 AA //AA is absolute destination, change CJMP flags for other jumps.

###Example Program
    Original Code   Compiled Code      Output Code           Binary Output
    SET AX 3        (ADD AX ZZ 3)      [ADDvsM AX ZZ _1]     0000 0 0 1 001 111 011
       //AX is now 3
    SET BX 5        (ADD BX ZZ 5)      [ADDvsM BX ZZ _1]     0000 0 0 1 010 111 101
       //BX is now 5
    ADD AX AX BX    (ADD AX AX BX)     [ADDvsm AX AX BX]     0000 0 0 0 001 001 010
       //AX is now AX + BX (8)
    SET CX 63       (ADD CX ZZ ZZ 63)  [ADDVsm CX ZZ ZZ 63]  0000 1 0 0 011 111 111 0000 0000 0011 1111
       //CX is now 63
    SUB BX CX AX    (SUB BX CX AX)     [SUBvsm BX CX AX]     0001 0 0 0 010 011 001
       //BX is now CX - AX (55)
    MUL AX AX AX    (MUL AX AX AX)     [MULvsm AX AX AX]     0010 0 0 0 001 001 001
       //AX is now AX * AX (64)
    DIV AX AX 16    (DIV AX AX 1 16)   [DIVVsM AX AX _1]     0011 1 0 1 001 001 001 0000 0000 0001 0000
       //AX is now AX / 16 (4)
    AND AX AX BX    (AND AX AX BX)     [ANDvsm AX AX BX]     0100 0 0 0 001 001 010
       //AX is now AX & BX (4)
    OR  AX AX 2     (OR AX AX 2)       [OR_vsM AX AX _2]     0101 0 0 1 001 001 010
       //AX is now AX | 2  (6)
    XOR AX AX 0xF   (XOR AX AX 15)     [XORVsM AX AX _0]     0110 1 0 1 001 001 000 0000 0000 0000 1111
       //AX is now AX ^ 15 (9)
