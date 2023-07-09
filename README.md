# MIPS Instructions

Instructions to implement:

| **Instruction**   | **Name**  | **Representation**  | **ROM Code** | **Behavior**  |
|-------|---|-------------------------|---| ---|
| BLTZ  | Branch if less than zero  | 0000 01 SRC 0 0000 IMED | 01 |  Similar to BEQ <br> `if $s < 0 advance_pc (offset << 2)); else advance_pc (4);` |
| DIV   | Divide | 0010 11 SRC TRG 0000 0000 0001 1010    | 00 |   R3 := r1 / r2. <br> `$LO = $s / $t; $HI = $s % $t; advance_pc (4); `|
| JAL   | Jump and Link  | 0000 11 IMED            | 03 |  JMPs to the new address and saves the old one on register $31 <br> `$31 = PC + 8 (or nPC + 4); PC = nPC; nPC = (PC & 0xf0000000) \| (target << 2);` |
| LB    | Load Byte  | 1000 00 SRC TRG IMED  | 20 | Loads one byte. <br>`$t = MEM[$s + offset]`  |
| SLTIU | Set on less than immediate unsigned  | 0010 11 SRC TRG IMED        | 0B |  If r2 <= IMED (unsigned), r1 := 1, else r1 := 0 <br> `if $s < imm $t = 1; else $t = 0;` |


# Progress Track

- (-) Not done
- ✅  Done

| **Progress**   | **Monocycle**  | **Multicycle**  | **Pipeline**  |
|----------------|----------------|-----------------|---------------|
| BLTZ           | ✅             | -               | -             |
| DIV            | ✅             | -               | -             |
| JAL            | ✅             | -               | -             |
| LB             | ✅             | -               | -             |
| SLTIU          | ✅             | -               | -             |

# Implementação

## Branch on Less Than Zero

## Division

## Jump and Link

## Load Byte

A instrução Load Byte tem opcode `10 0000`. Portanto, ela é configurada na região `20` da ROM do bloco de controle. Seu comportamento é muito similar ao da instrução `LW`, mas é necessário fazer alguns ajustes:

- Criamos um bit de controle chamado `ByteMemToReg`.
- A memória é lida sempre de palavra em palavra, de forma que os dois últimos bits do endereço eram descartados. Para carregar apenas um byte, usamos esses dois últimos bits para alimentar um MUX que escolhe qual byte dentro da palavra lida deve ser usado. 
- O resultado desse mux deve sofrer um SGEXT de 8 para 32 bits.
- O resultado do SGEXT deve ser tratado da mesma forma que o resultado do `LW`. Usamos o `ByteMemToReg` para decidir qual dos dois será escrito no BReg. 

## Set on Less than Immediate Unsigned