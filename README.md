# MIPS Instructions

Instructions to implement:

| **Instruction**   | **Name**  | **Representation**  | **ROM Code** | **Behavior**  |
|-------|---|-------------------------|---| ---|
| BLTZ  | Branch if less than zero  | 0000 01 SRC 0 0000 IMED | 01 |  Similar to BEQ <br> `if $s < 0 advance_pc (offset << 2)); else advance_pc (4);` |
| DIV   | Divide | 0000 00 SRC TRG 0000 0000 0001 1010    | 00 |    `$LO = $s / $t; $HI = $s % $t; advance_pc (4); `|
| JAL   | Jump and Link  | 0000 11 IMED            | 03 |  JMPs to the new address and saves the old one on register $31 <br> `$31 = PC + 8 (or nPC + 4); PC = nPC; nPC = (PC & 0xf0000000) \| (target << 2);` |
| LB    | Load Byte  | 1000 00 SRC TRG IMED  | 20 | Loads one byte. <br>`$t = MEM[$s + offset]`  |
| SLTIU | Set on less than immediate unsigned  | 0010 11 SRC TRG IMED        | 0B |  If r2 <= IMED (unsigned), r1 := 1, else r1 := 0 <br> `if $s < imm $t = 1; else $t = 0;` |


# Progress Track

- (-) Not done
- 🟨 To test
- ✅  Done

| **Progress**   | **Monocycle**  | **Multicycle**  | **Pipeline**  |
|----------------|----------------|-----------------|---------------|
| BLTZ           | ✅             | 🟨               | -             |
| DIV            | ✅             | 🟨               | -             |
| JAL            | ✅             | 🟨               | -             |
| LB             | ✅             | 🟨             | -             |
| SLTIU          | ✅             | -               | -             |

# Implementação

## Branch on Less Than Zero

A instrução BLTZ tem opcode `00 0001`. Portanto, ela é configurada na região `01` do ROM do bloco de controle. Além disso, ela deve ter um `ft2` fixo como `00000`. Como nenhuma outra instrução de mesmo opcode deve ser implementada (BGEZ, BGEZAL, BLTZAL), apenas devemos reconhecer o opcode e ativar o JMP se duas condições forem atingidas: `ft2 = 00000` e `BReg[ft1] < 0`. 

Para verificar se um número é menor que zero, basta ver seu bit mais significativo. 

- **(Multicycle)**: Assim como em outras instruções, escolhemos aproveitar códigos de instrução e saídas antigas. Como a instrução BEQ é a única outra `Branch`, e ela utiliza `ALUSrcA=1`, podemos definir `PCWriteCon=1 && ALUSrcA=0` como representando `BLTZ_Instr`. Então basta fazer um AND entre `BLTZ_INSTR`, `LTZ` e `BLTZ_Code` como uma outra possível condição de branch, similarmente à implementação no Monociclo.


## Division

O caminho de dados da divisão é praticamente igual ao de soma, nas três versões do MIPS. Porém, pela especificação, é necessário criar um novo par de registradores `HI` e `LO`. Também foi necessário criar uma nova saída da ULA, que representasse o resto da divisão. 

Dessa forma, só precisamos criar os caminhos de escrita do HI/LO, assim como reconfigurar as flags que decidem o `ENABLE` tanto do BReg quanto do HiLo. Se a operação é Divisão, então Enable_BReg=0 e `Enable_HiLo=1`. Senão, `Enable_HiLo=0`, e `Enable_BReg` continua com seu comportamento padrão. 

## Jump and Link

- **(Multicycle)** Aumentamos o MUX entre o MDR e o BReg Data In para acomodar mais um bit. A partir daí, usamos a constante 0x1F (31) como endereço de escrita e PC como valor de escrita no BReg. Quando isso estiver pronto, basta usar o estado `9` para ler o novo PC de `IMED`.

1. RI <- M[PC];  (Estado `0`)
2. BReg[31] <- PC (Novo Estado `B`)
3. PC <- IMED  (Estado `9`, usado em JUMP)

**Estado B:**
PCWrite = 1; ALUOP = 10, causando Link=1

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