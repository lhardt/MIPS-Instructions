# MIPS Instructions

Instructions to implement:

| **Instruction**   | **Name**  | **Representation**  | **Behavior**  |
|-------|---|-------------------------|---|
| JAL   | Jump and Link  | 0000 11 IMED            |  JMPs to the new address and saves the old one on register $31 <br> `$31 = PC + 8 (or nPC + 4); PC = nPC; nPC = (PC & 0xf0000000) | (target << 2);` |
| BLTZ  | Branch if less than zero  | 0000 01 SRC 0 0000 IMED |  Similar to BEQ <br> `if $s < 0 advance_pc (offset << 2)); else advance_pc (4);` |
| LB    | Load Byte  | 1000 00 SRC TRG IMED  | Loads one byte. <br>`$t = MEM[$s + offset]`  |
| DIV   | Divide | 0010 11 SRC TRG IMED    |   R3 := r1 / r2. <br> `$LO = $s / $t; $HI = $s % $t; advance_pc (4); `|
| SLTIU | Set on less than immediate unsigned  | 1000 00 TRG IMED        |  If r2 <= IMED (unsigned), r1 := 1, else r1 := 0 <br> `if $s < imm $t = 1; else $t = 0;` |
