# MIPS Instructions

Instructions to implement:

- **JAL** (end) 
   Jump and Link. JMPs to the new address and saves the old one on register $31
- **BLTZ** (reg) (end)
   Branch if less than zero. Similar to BEQ. 
- **LB** 
   Load Byte. Loads only one byte from memory.
- **DIV** r1, r2, r3
   R3 := r1 / r2.
- **SLTIU** r1, r2, imed
   If r2 <= IMED (unsigned), r1 := 1, else r1 := 0