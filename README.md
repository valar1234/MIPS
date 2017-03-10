Classic-MIPS

A classic 5-stage pipeline MIPS 32-bit processor, including a 2-bit branch predictor, a branch prediction buffer and a direct-mapped cache.

This MIPS processor is coded according to the fifth edition book of David A.Patternson & John L.Hennsessy, "Computer Organization and Design". The architecture is the referred to the descriptions in this book.

This MIPS is fully evaluated on Xilinx Vivado design tools and several testbenchs are providered to test the processor. It is also evaluated on Xilinx ZC706 board.

1: Classic 5-stage pipeline, including FETCH, DECODE, EXECUTE, MEM and WB.

2: Apply the Dynamic Branch Prediction technology to improve the branch accuracy. It contains a 2-bit branch predictor and a 1024 depth branch prediction buffer to predict the branch address. Furthermore, it takes the jump instruction(j, jal, jr .etc) as specital branch instructions to deal with, and this can simplfy the jump implementation.

3: Implement the Forwarding and Stalling mechanism to solve the data-hazard.

4: A 2KB direct-mapped data-cache is added to accelerate memory access. This cache has 64 blocks and each block has 8 words. A cache controller is designed to connect the MIPS processor and the 64KB main memory. The cache is implemented as LUT in Xilinx FPGAs, and the main memory is implemented as Block Rams(BRAM).

This MIPS processor supports the basic instruction as bellow:

1 TYPE-R: AND, OR, ADD, SUB, SLT, NOR, SLRV, SLLV

2 TYPE-I: ADDI, ANDI, ORI, XORI, LUI

3 MEM: LW, SW

4 BRANCH: BEQ, BNE

5 JUMP: J, JAL, JR

Some testbenchs are providered. All the codes are assembled by "mipsasm.exe", which is a MIPS assembler and simulator.

1 sort: use the search-maximum method to sort an array.

2 stack_sort: use two stacks to sort an array.

3 finonacci: calculate the Fibonacci series.

4 gcd: calculate the gcd of two numbers

5 function/fact: test the function recursion. Accumulate the result in recursively way.

Any problems, please contant jfjin14@fudan.edu.cn