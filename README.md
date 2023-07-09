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
| LB             | ✅             | ✅             | -             |
| SLTIU          | ✅             | -               | -             |

# Implementação

## Branch on Less Than Zero

## Division

## Jump and Link

## Load Byte

A instrução Load Byte tem opcode `10 0000`. Portanto, ela é configurada na região `20` da ROM do bloco de controle. Seu comportamento é muito similar ao da instrução `LW`, mas é necessário fazer alguns ajustes:

1. Criamos um bit de controle chamado `ByteMemToReg`.
2. A memória é lida sempre de palavra em palavra, de forma que os dois últimos bits do endereço eram descartados. Para carregar apenas um byte, usamos esses dois últimos bits para alimentar um MUX que escolhe qual byte dentro da palavra lida deve ser usado. 
3. O resultado desse mux deve sofrer um SGEXT de 8 para 32 bits.
4. O resultado do SGEXT deve ser tratado da mesma forma que o resultado do `LW`. Usamos o `ByteMemToReg` para decidir qual dos dois será escrito no BReg. 

5. **(Multicycle)**: Na ROM de próximo estágio, utilizaremos no LB (entradas `20X`) os mesmos estágios de LW (entradas `23X`). Exceto no quarto estágio, em que `MDR <- M[ALU_OUT]`, em que utilizaremos um novo estado para discernir que só salvaremos um byte. Utilizaremos o primeiro código de estado livre, `A`.

6. **(Multicycle)**: No estado `A`, ativaremos todos os bits do estágio `3` de LW, mais o bit de `ByteMemToReg`. No entanto, não conseguimos criar um novo bit saindo da ROM de saída da máquina de estados sem aumentar sua largura e criar muito hardware. Por isso, escolhemos *reaproveitar* o bit 0 de `ALUop` para *também* significar `ByteMemToReg`, dado que esse bit não será utilizado nesse estágio. Assim, o código da saída de `A`é `3004`

7. **(Multicycle)** Escolhemos fazer essa mudança no estágio 4 pois é quando o endereço está na saída de ALU OUT. Basta que o MUX do item (2) esteja entre a saída de MEM e a entrada de MDR.

## Set on Less than Immediate Unsigned