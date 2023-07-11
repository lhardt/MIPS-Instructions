# MIPS Instructions

Instructions to implement:

| **Instruction**   | **Name**  | **Representation**  | **ROM Code** | **Behavior**  |
|-------|---|-------------------------|---| ---|
| BLTZ  | Branch if less than zero  | 0000 01 SRC 0 0000 IMED | 01 |  Similar ao BEQ <br> `if $s < 0 advance_pc (offset << 2)); else advance_pc (4);` |
| DIV   | Divide | 0000 00 SRC TRG 0000 0000 0001 1010    | 00 |    `$LO = $s / $t; $HI = $s % $t; advance_pc (4); `|
| JAL   | Jump and Link  | 0000 11 IMED            | 03 |  JMP até o novo endereço. Salva o antigo em $31 |
| LB    | Load Byte  | 1000 00 SRC TRG IMED  | 20 | Similar ao LW. Carrega um byte. <br>`$t = MEM[$s + offset]`  |
| SLTIU | Set on less than immediate unsigned  | 0010 11 SRC TRG IMED        | 0B |  Se r2 <= IMED (unsigned), r1 := 1, senão r1 := 0 <br> `if $s < imm $t = 1; else $t = 0;` |


# Implementação

## Branch on Less Than Zero

A instrução BLTZ tem opcode `00 0001`. Portanto, ela é configurada na região `01` do ROM do bloco de controle. Além disso, ela deve ter um `ft2` fixo como `00000`. Como nenhuma outra instrução de mesmo opcode deve ser implementada (BGEZ, BGEZAL, BLTZAL), apenas devemos reconhecer o opcode e ativar o JMP se duas condições forem atingidas: `ft2 = 00000` e `BReg[ft1] < 0`. 

Para verificar se um número é menor que zero, basta ver seu bit mais significativo. 

- **(Multicycle)**: Assim como em outras instruções, escolhemos aproveitar códigos de instrução e saídas antigas. Como a instrução BEQ é a única outra `Branch`, e ela utiliza `ALUSrcA=1`, podemos definir `PCWriteCon=1 && ALUSrcA=0` como representando `BLTZ_Instr`. Então basta fazer um AND entre `BLTZ_INSTR`, `LTZ` e `BLTZ_Code` como uma outra possível condição de branch, similarmente à implementação no Monociclo.


## Division

O caminho de dados da divisão é praticamente igual ao de soma, nas três versões do MIPS. Porém, pela especificação, é necessário criar um novo par de registradores `HI` e `LO`. Também foi necessário criar uma nova saída da ULA, que representasse o resto da divisão. 

Dessa forma, só precisamos criar os caminhos de escrita do HI/LO, assim como reconfigurar as flags que decidem o `ENABLE` tanto do BReg quanto do HiLo. Se a operação é Divisão, então Enable_BReg=0 e `Enable_HiLo=1`. Senão, `Enable_HiLo=0`, e `Enable_BReg` continua com seu comportamento padrão. 

## Jump and Link

A instrução Jump and Link é muito semelhante à instrução JUMP. Aumentamos o MUX entre o MDR e o BReg Data In para acomodar mais um bit de seleção, de forma que possamos escolher a constante 31 como endereço de escrita. 

- **(Multicycle)** Podemos utilizar o estado `9` para ler o novo PC de `IMED`. \
    **Estado B:**  PCWrite = 1; ALUOP = 10, causando Link=1


        1. RI <- M[PC];  (Estado `0`)
        2. BReg[31] <- PC (Novo Estado `B`)
        3. PC <- IMED  (Estado `9`, usado em JUMP)
 
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

Para implementar SLTIU, utilizamos um comparador configurado como "unsigned". Utilizamos a saída `<` do comparador (após SGEXT de 1 para 32 bits). 

- **(Monocycle)**:  Essa saída é colocada como entrada de um MUX, cuja outra entrada é ALU_OUT. Depois disso, basta escrever esta saída normalmente em BReg. A estratégia de _Pipeline_ é semelhante.

- **(Multicycle)**: A melhor estratégia que encontramos, dentro do contexto do multiciclo, é apenas criar uma nova operação (e o comparador) _dentro_ da ALU. Assim, precisaremos de um novo estado que mude o valor de ALUOP para um comparador (ALU_OP=5). Essa é uma estratégia diferente daquela
que utilizamos no monocycle. 

- **(Multicycle)** Podemos utilizar um novo estado `D` para representar o passo de atribuição de ALU_OUT. Como nas outras instruções, poderemos economizar HW significativente deixando `PCWrite=0, PCWriteCond=0, PCSource=3` para representar a condição de `Use_Comparator`, já que `PCSource=3` não tinha nenhum uso até então. Além disso, precisaremos usar `ALUSourceA=`

        1. RI <- M[PC];   (estado `0`)
        2. A <- Breg[ft1]; B <- Breg[ft2]; (estado `1`)
        3. ALU_Out <- SgExt(A cmp (SgExt Imed)); (novo estado `D`, ALU_Control = 101)
        4. Breg[ft2] <- ALU_Out (estado `7`)
        3. ALU_Out <- SgExt(A cmp (SgExt Imed)); (novo estado `D`)
        4. Breg[ft2] <- ALU_Out (estado `7`)
