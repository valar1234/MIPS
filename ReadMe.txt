SimpleMIPS说明：
	1：实现单周期的MIPS指令集
	2: 最高时钟频率可达 5.4ns <=> 185MHz


PipelineMIPS说明：
	1: 实现IF, ID, EX, MEM, WB的五级流水结构
	2:未使用旁路与阻塞前:
		1)数据冒险，要求等待3个nop指令。
			ori $t0, $zero, 10
			nop
			nop
			nop
			add $s0, $s0, $t0
		ori与add之间需要等待3个nop指令，以满足$t0的正常功能。
	
		2)按照书Page206方式修改IF级(decode.v)
			a:使时钟周期的前半段执行写操作，后半段执行读操作，使得读操作读取到最新的内容。
			b:经过实践分析发现:
				I: 这种方法确实可以缩短数据冒险等待时间，由3个nop指令减少为2个nop指令。
				II: 但是严重影响了时序！！！这种方法使得寄存器输出reg1_data和reg2_data只能维持半个时钟周期，
					缩短了后续参与ALU运算的时间。
				III: 考虑到时序的代价， 本设计放弃了这个优化方法。
	3: 最高时钟频率可达: 3.4ns <=> 294MHz
		
		
PipelineMIPS_opt说明：
	1：在PipelineMips中加入旁路与阻塞机制。
	2: 在WB级引入旁路机制，使得读操作可以读取到最新的内容， 作用与上一节.2.2)相同，可以减少一个nop指令的等待时间， 但是对时序没有影响。
		具体修改在decode.v中，对reg1_data和reg2_data使用一个选择器MUX进行旁路，如果读操作与写操作同时进行时，对读操作进行
		旁路，使得读操作获取最新的数据。
		    /* create a bypath for reg1 output */
            if(regwrite_flag && write_reg == rs)
                reg1_data <= write_data;
            else
                reg1_data <= registers[rs];
            /* create a bypath for reg2 output */
            if( regwrite_flag && write_reg == rt)
                reg2_data <= write_data;
            else
                reg2_data <= registers[rt];
	3:按照书Page.211，加入了旁路机制， 包括多选器和旁路单元。
		旁路单元的设计参照Page.210，但是MEM冒险的控制策略使用上面的，而不是下面的， 即没有加黑体的。
		assign forwardA_ex_mem_condition = (ex_mem_regwrite_flag == 1'b1) && (ex_mem_rd != 0) && (ex_mem_rd == id_ex_rs);
		assign forwardA_mem_wb_condition = (mem_wb_regwrite_flag == 1'b1) && (mem_wb_rd != 0) && ( mem_wb_rd == id_ex_rs);
    
		assign forwardB_ex_mem_condition = (ex_mem_regwrite_flag == 1'b1) && (ex_mem_rd != 0) && (ex_mem_rd == id_ex_rt);
		assign forwardB_mem_wb_condition = (mem_wb_regwrite_flag == 1'b1) && (mem_wb_rd != 0) && ( mem_wb_rd == id_ex_rt);
	4：按照书Page.214，加了了阻塞机制。
		lw $t0, 0($zero)
		addi $s0, $t0, $t1
		由于addi需要访问lw的$t0寄存器，即使加了了旁路机制，lw之后仍然必须阻塞一个时钟周期，否则addi无法访问到$s0的正确值。
	5: 1)当使用Block_Ram作为数据存储器时， 由于读操作有一个时钟周期的延时，因此读操作的结果没有使用寄存器缓冲。在加入旁路机制后，Block_Ram的输出到ALU的输入是关键路径，
		严重影响了时序。
	   2)如果使用Distributed Ram作为数据存储器， 读操作是没有延时的， 那么读操作的结果可以缓存到流水线寄存器中，时序能够得到缓解。
	   3)在Ram.v中同时例化了Block_Ram和Distributed Ram，用户可通过定义宏 __BLOCK_RAM__ 来选择Ram的种类。
	6: 1)在不优化控制冒险的条件下:
			beq, bne, j等分支和跳转指令后面， 需要等待3个nop指令。
	   2)实现了书Page.215中"假设分支不发生"的控制冒险解决策略。
			假设分支不会发生。如果MEM级检测分支条件不成立， 则继续执行；
								如果MEM级检测分支条件成立， 则清除(FLUSH)分支指令后的三个指令，具体做法如下:
								1)IF级的reg_instruction设置成NOP
								2)ID级的控制信号全部清除成0。
								3)EX级的控制信号全部清除成0.
	7:最高时钟频率：5.4ns <=> 185MHz
	
PipelineMIPS_opt2说明：
	1)在Ram.v模块，RAM的输入地址舍弃低2位。
		因为MIPS中， 要求RAM的地址总是被4整除，而本设计的RAM的数据宽度都是32-bit的， 因此可以将地址右移2位， 相当于舍弃低2位。
		这要求，用户写汇编时， 必须保证地址总是4的整数倍！！！
		assign addr = ALU_out[9:2];
	2)a:经综合后时序分析可知， PipelineMIPS_opt工程相对于PipelineMIPS工程时序恶化的原因是：
			EX级加入旁路机制后，增加了EX级的关键路径。关键路径如下:
				mem/reg_write_reg ->EX级旁路单元(bypath_ctr) -> EX旁路选择器(bypass) ->ALU/ALU_out -> ALU/zero_flag
	  b:优化方法1：
		为了缩短上述关键路径， 将旁路单元bypath_ctr移到ID级执行。
		在ID级提前执行旁路单元，因为旁路单元所需的数据都可以提前拿到，然后将计算到的forward_a, forward_b寄存器缓存， 然后传输到EX级。
			优化后的关键路径如下:
			mem/ram_out -> EX旁路选择器(bypass) -> ->ALU/ALU_out -> ALU/zero_flag
		优化方法将关键路径延时从5.4ns减少到了4.4ns.
	  c)优化方法2:
		为了进一步缩短关键路径，将zero_flag的计算从EX级的ALU转移到MEM级计算。
		因为ALU单元先计算出ALU_out, 然后根据ALU_out计算出zero_flag， 因此zero_flag的延时比ALU_out更长。
		由于ALU_out会通过流水线寄存器传到MEM级，因此可以在MEM级计算zero_flag，从而缩短EX级的关键路径。
			优化后的关键路径如下:
			mem/ram_out ->EX旁路选择器(bypass) ->ALU/ALU_out
		优化方法将关键路径延时从4.4ns减少到了3.8ns.
	3)相比于PipelineMIPS工程， PipelineMIPS_opt2加入了旁路、阻塞与分支预测机制，解决了数据冒险和控制冒险问题，代价是关键路径延长了0.4ns(3.4ns->3.8ns).
		相比于SimpleMIPS工程，使用流水线， 时钟频率从185MHz提高到263MHz.
	
	4)最高时钟频率: 3.8ns <=> 263MHz
	
PipelineMIPS_opt3说明:
	1:在IF阶段，调整Flush和Hazard关系。
		Flush的优先级高于Hazard.
	2: 优化JUMP指令。
		在IF阶段提前解码，判断当前指令是否是JUMP指令。
		如果当前指令是JUMP指令， 判断后续流水线中是否有branch指令，如果有branch指令，阻塞直到branch指令完成.(因为如果Jump前面有branch指令， 那么Jump不一定会执行)
					如果后续流水线中没有branch指令， IF级马上JUMP到新地址， 而不需要等到MEM级。
	3:通过面积换时序的方法，将时间延时从3.8ns减少到3.1ns。
		具体改动如下：
		1) ALU.v中添加了c_addsub_0模块实现add/sub功能，缩短了ALU运算延时。
			!!发现DSP实现比LUT实现慢很多！！！所以c_addsub_0是用16个LUT实现的。
		2)添加了bypath2.v模块， 将ALUSrc_flag放入了旁路选择器中bypath2， 这样ALU.v缩短了一个选择器的延时，
			但是代价是增加了较多的LUT实现bypath2模块。
		3)mem/isFlush信号曾经是关键信号路径，因为zero_flag的运算使得isFlush信号延时很长。因此使用了ALU_out比较器代替了
			zero_flag参与运算，缩短了isFlush延时， 代价是使用较多的LUT实现zero_flag的比较器。
			//    assign isFlush =  ((ctr_m[3:2] == `BRANCH_OP_BEQ) && (zero_flag == 1'b1)) || 
			//                ( (ctr_m[3:2] == `BRANCH_OP_BNE) && (zero_flag == 1'b0) );
				assign isFlush =  ((ctr_m[3:2] == `BRANCH_OP_BEQ) && (ALU_out == 32'h0)) || 
						( (ctr_m[3:2] == `BRANCH_OP_BNE) && (ALU_out != 32'h0) );
		4)execute/ALU_ctr中， instruction->ALUcmd->ALU_out是关键路径。因此，将ALU_ctr模块从execute迁移到decode级计算。
	4:经过上述优化，最高时钟频率: 3.1ns <=> 322MHz
	
PipelineMIPS_opt4说明:
	1:为了实现jal jr等跳转指令， 将MIPS寄存器中的$ra(第31号寄存器)挪到fetch中， 因为只有fetch级会使用到$ra.
		jal与jr的使用，可实现函数调用的功能。保存调用函数的地址，返回后继续执行。
		###jal调用函数, jr函数返回。
		具体例子见 /TestBenchs/function/code.smd
		
		！！！但是不支持函数的嵌套调用！！！(即主函数调用的子函数调用其他函数， 否则$ra被覆盖导致功能错误)
		
	2: jal指令与j指令大致相同， 只是额外执行 $ra = [PC] + 4， 即暂存下一条指令的地址。因此使用jal调用函数。
	3: jr指令做了修改，在硬件上进行了缩减， 使其只支持 jr $ra, 即只能跳转到$ra指令的位置.因此使用jr实现函数返回。
	4:经过上述优化，最高时钟频率: 3.1ns <=> 322MHz
	5：资源使用情况， 基于xc7z045ffg900-2芯片。
		FF: 1463
		LUT: 1744
		
PipelineMIPS_opt4_example说明:
	1: 例化PipelineMIPS_opt4，烧录流水灯/TestBenchs/led/code.smd程序，创建工程、生成位流并上板子测试。测试工程正常。


PipelineMIPS_opt5_cache
	1: 在PipelineMIPS_opt4基础上加入一个直接映射的cache
	2: cache说明
		1)cache: 块大小是8个字(8x32)，64项。故大小:64x8x32= 16K bit = 2K byte
		2)cache命中时，读写都是单周期内完成。
		3)cache对于写缺失采用写回策略。
	3：加入cache后，时序稍微变差，最高时钟频率: 3.4ns <=> 294MHz
	
PipelineMIPS_opt5_cache_example3
	1: 例化PipelineMIPS_opt5_cache， 烧录除法运算/TestBenchs/cache_example/code.smd程序，创建工程、生成位流并上板子测试。测试工程正常。
	2：测试了以下功能， 且正常。
		1)使用BRAM_Controller IP实现对主存的访问. (主存是True Dual Port， 一个Port给cache， 一个Port给用户使用)
		2)测试了MIPS调用除法函数DIV。
		3)测试了cache的读写、缺失、重分配和写回等功能。
	3：存储器的正常工作如下:
		1)通过用户Port(BRAM_Controller)向主存写入运算的操作数，addr = 0, 4
		2)MIPS读操作数,这时发生读缺失，从主存分配数据到cache中。
		3)MIPS进行除法运算，计算结果保存到cache的地址addr = 32, 36
		
		/* 一：为了让MIPS的cache每次加载到用户写入到主存的最新操作数， 那么应该让addr = 0, 4的cache失效，以便下次
		从主存重新分配 */
		4) "lw $t0, 2048($zero)", 该指令让 addr = 0,4的cache失效
		
		/* 二：为了让用户读取到计算结果， 需要让addr = 32, 36对应的cache写回到主存中 */
		5) "lw $t1, 2080($zero)", 该指令让 addr = 32, 36对应的cache写回到主存中，然后用户通过BRAM_Controller可以
		读取计算结果。
	4：在有cache的系统中，当用户通过BRAM_Controller访问主存时，需要注意cache与主存的一致性问题。	

	
PipelineMIPS_opt6_cache_bht
	1: 在fetch级，加入了一个1024项的BHT(branch history table)，用动态分支预测提高分支的准确率。
	2：BHT包含1024项， 使用分支指令地址的低位进行索引，每项大小是33-bit:
		1):bit:32, 预测位，预测该分支指令是否发生。
			如果该位为1，表示预测分支发生，将bit[31:0]作为下一条指令地址。
			如果该位为0，表示预测分支不发生， 下一条指令地址为当前指令地址 + 4.
		2): bit[31:0] 分支指令发生时对应的跳转目标地址。
	3：测试
		1)没有使用bht前，采用预测不发生的策略，预测准确率大约50%
		  使用bht后，根据局部性原理， 预测准确率可 >50%.
		2)测试发现, bht确实可以有效提高预测准确率，以提高运行速度。
		
	4: 集成cache和bht后，综合报告如下:
		1) 加入bht后， 最高工作频率: 3.5ns => 285.7MHz
			关键路径依然是cache, 从reg_ALU_out -> isCacheDone.
		2) 资源使用
			FF:  			1626
			LUT: 			3004
			MEMORY LUT:		983
			BRAM:			7.5
			
PipelineMIPS_opt7_cache_bht
	1: 将bht的预测位，由1-bit扩展到2-bit,以提高准确率.
	4: 集成cache和bht后，综合报告如下:
		1) 加入bht后， 最高工作频率: 3.6ns => 277.8MHz
			关键路径依然是cache, 从reg_ALU_out -> isCacheStall.
		2) 资源使用
			FF:  			1630
			LUT: 			3163
			MEMORY LUT:		1015
			BRAM:			7.5

PipelineMIPS_opt8_cache_bht
	1: 在PipelineMIPS_opt7_cache_bht基础上， 对SimpleCache做了稍微修改。
		去掉了cache_done信号， 直接输出isCacheStall信号， 使接口简洁一些。
	2：资源和时序几乎相同。
	
PipelineMIPS_opt9_cache_bht
	1：目的是完善j, jal, jr三个跳转指令，可实现对$ra寄存器的读写，进而实现函数的嵌套调用。
		目前j, jal, jr指令已完善。 jr指令可支持任何寄存器， 不限于$ra。
		
		此外还修复了lw指令在decode的一个bug。 当isHazard是， 应保持reg_pc <= reg_pc，否则会导致跳转跳转指令
		地址出错。
		
	2：修改
		1)将$ra从fetch级挪回到decode级， 并放置在registers寄存器堆中，使得$ra能够正常访问。
		2) jal指令的执行较为特殊， 需要生成regwrite_flag信号、特殊设置$rt、特殊计算reg_ALU_out等值，
			以实现 $ra <-- [PC] + 4.
		3)将jal. j, jr等指令当成特殊的条件分支指令处理，使用BHT进行分支动态预测。
			相对于branch指令， 跳转指令的分支一定会发生，因此可以特殊的条件分支处理，并且在fetch级利用BHT
			进行分支动态预测， 在mem级对BHT进行修正。
	3：修改后，资源报告如下:
		FF: 			1709
		LUT: 			3171
		MEMORY LUT:		1015
		BRAM:			7.5
	4:修改后时序得到了优化，最高工作频率: 3.5ns => 285.7.1MHz。
	5: 此外在大多数测试程序中发现，消耗的周期数有所减少，性能略有提升。
	
