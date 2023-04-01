# 引导

计算机类型

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233145155.png" alt="image-20230401233145155" style="zoom:50%;" />

注意没有实现MISD的计算机。

评价指标：

Response time响应时间：

CPU time：cpu的计算时间，不包括等待IO的时间，氛围用户模式和系统模式时间两种。

Throughput吞吐量：单位时间内完成工作的量

MIPS(Millions of Instructions per Second): 

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233151091.png" alt="image-20230401233151091" style="zoom:50%;" />

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233158566.png" alt="image-20230401233158566" style="zoom:40%;" />

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233218868.png" alt="image-20230401233218868" style="zoom:50%;" />





# 流水线Pipeline

思想：一个流程分成多个步骤，每个步骤有单独负责的硬件，每个时刻每个硬件都在工作，处理不同的指令。

流水线寄存器：RISC-V的五个步骤IF -> ID ->EX ->MEM -> WB，每个阶段后都有一个寄存器，可以存运算的结果。

hazard冒险/竞争：指令无法在流水线正常线性执行的情况

structural hazard：硬件资源冲突

data hazard：读取数据冲突，通常是后面的指令需要的数据还没有被之前的指令计算完毕或存储完毕。只有写后读才算是真正逻辑意义上的数据冲突。

control hazard：涉及到条件跳转时，后面的指令在执行时还不知道是否应该跳转。

解决方法：

stall（阻塞）；暂停流水线，也称加入气泡bubble。已经在流水线的工作继续执行，但不再进行Instruction Fetch。 等到之前的指令已经基本完成，后面指令所需要的资源已经有了的时候，再继续流水线。stall过多会降低流水线性能，使得CPI比1大。

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233227264.png" alt="image-20230401233227264" style="zoom:50%;" />

- 结构冒险

    - 前后指令同时访问相同的硬件资源：一般是同时访问reg file或同时读写内存。

    - 解决方法：

        直接stall（效率低）

        double pump：一个时钟内，前半个时钟沿写，后半个时钟沿读（先写后读，保证指令顺序逻辑），这样不会同时读或同时写寄存器或内存。此方法不需要检测hazard

- 数据冒险

    - 指的是指令在某一阶段所需要的数据并未被之前的指令处理完毕（如计算完但未写入reg file / memory）。导致流水线上出现数据逻辑与指令顺序不一致的情况。

    - R型：第一条指令为R型，需要最后一步WB才写回寄存器，但下面指令再第二阶段就已经需要访问该数据。注意由于double pump的存在，同一个时钟内不会导致数据冒险（先写后读，逻辑正确）。所以这种情况会影响未来两个时钟周期内的数据访问。

    - 解决方法：旁路（forwading，bypass）

        R型指令的计算结果（目标寄存器的值）在EX阶段就已经被计算出来，所以可以设置pipeline register存储计算出的值，在下一条指令需要的时候直接读即可，不需要读写回的数据。在五级流水线中，旁路可以完全解决R型数据冒险。
		
    - load数据冒险：ld指令也是在最后才写回，但是其需要先计算地址值然后在MEM阶段才能真正读到寄存器的目标值，所以比R型晚了一步。一样可以设置MEM/WB的寄存器，但此时只能解决其指令隔一条指令的指令的数据冒险问题，对于load情况下的下一条指令，如果发生数据冒险，必须stall。
    
        <img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233235431.png" alt="image-20230401233235431" style="zoom:50%;" />
    
    - 汇编优化：load stall比较影响性能，生成汇编代码时，可以通过改变指令顺序的方式，尽量避免出现load stall的情况。
    
    - 广义数据冒险
    
        以上提到的数据冒险都是RAW（Read After Write）型，也是唯一在普通五级流水线上会发生的类型。
    
        但如果流水线较为复杂，如涉及到多步的浮点运算，冒险未必都是RAW。也可能是WAR或者WAW。
    
        RAW：写后读，A是写，其后的指令B是读，但是由于流水线的存在，B的读实际发生在了A的写前面，导致B读到了旧值。可以用double bump及forwading的方法解决。是唯一可能在RISC-V普通五级流水线中出现的依赖类型，也是唯一真正的逻辑上的真依赖。
    
        WAR：读后写，A是读，其后的指令B是写，但是由于流水线的存在，B的写实际发生在了A的读前面，导致A读到了新值（然而逻辑上A不应该读到还出现的值）。较为少见，一般在复杂流水线中出现，stall解决。
    
        WAW：写后写，A是写，其后的指令B也是写，但是由于流水线的存在，B的写实际发生在了A的写前面，导致A写的值又再次覆盖了B（逻辑上应该最终是B写的值）。WAW依赖存在的流水线，一定有存在多个阶段会写相同目标，否则不可能发生。也不常见，stall解决。

- 控制冒险

    - 由于存在跳转控制指令（beq，jal等），这些指令在执行完毕后会得到应该跳转的PC，改变指令执行顺序，但这是非流水线的逻辑，流水线中，下一条指令不知道跳转指令跳转的位置就需要马上读取下一条指令（PC顺序）。控制冒险较难解决。

    - stall：

        遇到跳转指令时（第一阶段IF就可以发现），马上stall下一条指令，直至跳转逻辑得到下一个PC（MEM阶段），然后执行下一条指令（如果发生了跳转，需要IF新PC）。需要阻塞3个时钟，效率很低。

        可以通过提早分析跳转条件（ID阶段）来减少stall时钟：

        <img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233242067.png" alt="image-20230401233242067" style="zoom:40%;" />

        这样在在ID/EX处就可以知道是否跳转，从而只需要stall一个时钟。

    - delay slot（延迟槽）

        在跳转指令后执行一些与跳转逻辑无关的指令，需要编译器优化。

        如果不跳转，则继续正常执行。如果发生跳转。
        
        为了性能考虑，延时槽里放的最好是原本在跳转指令之前的指令，且调度后逻辑顺序没有影响。这样无论是否跳转，延时槽内的指令计算都不会白费（跳抓也不需要撤销）。
        
        一般的流水线中，延时槽的大小为1。
    
- Latency：指令产生结果和结果被读取之间所隔的最小时钟周期数

- Initiation interval：改功能单元指令发射上去的最小间隔

    非流水化时，后者是前者的+1

    <img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233259612.png" alt="image-20230401233259612" style="zoom:50%;" />




# 异常处理

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233311453.png" alt="image-20230401233311453" style="zoom:40%;" />



# CSR状态控制寄存器

是架构中重要的寄存器组（区别于x0～x31这32个通用寄存器）。

- mstatus（machine status）

    <img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233323058.png" alt="image-20230401233323058" style="zoom:50%;" />

    Global interrupt-enable bits（中断激活比特）

    UIE / SIE / MIE：分别是U、S、M模式（权限由低到高）下的Interrupt Enable bit。注意，如果高权限下屏蔽中断可以顺带屏蔽更低权限下的中断，而低权限若想允许中断，必须自身和高于其权限的模式的IE都设为1。

    为了嵌套中断，应该为IE设栈。

    To support nested traps, each privilege mode *x* has a two-level stack of interrupt-enable bits and

    privilege modes. *x*PIE holds the value of the interrupt-enable bit active prior to the trap, and

    *x*PP holds the previous privilege mode. The *x*PP fields can only hold privilege modes up to *x*, so

    MPP is two bits wide, SPP is one bit wide, and UPP is implicitly zero. When a trap is taken from

    privilege mode *y* into privilege mode *x*, *x*PIE is set to the value of *x* IE; *x* IE is set to 0; and *x*PP

    is set to *y*.

​		xRET指令可以实现x模式下从中断中返回，会出栈。

​		此外，mstatus还有许多权限相关的比特

- mtvec（Machine Trap-Vector Base-Address Register ）



# Cache缓存

## Cache的概念

为了解决不同存储的速度差异问题，提出缓存概念。

两个原则：

时间局部性原则Temporal Locality：被访问到的对象很可能马上会被**再次**访问

空间局部性原则Spatial Locality：被访问到的对象**附近**的对象很可能会被接下来访问（“附近”一般指内存地址相近的位置）

cpu经常需要读写mem，mem中一部分将拷贝到cache（更小）中。当访问到mem中某个cache中没有的数据时，会将访问的数据所在的块（block）一整个拷贝到cache**对应**位置（cache的基本单位是block）。

- 写hit策略

    write through：直写，每次同时写缓存和内存。

    write back：回写，先只写到缓存，后面被替换时才写回内存

- 写miss策略：

    write allocate：先将目标载入cache，再写

    write around：直接写进内存，不先载入cache
    
- 写回策略

    write stall：write through中，CPU等待写入内存的过程

    write buffers：准备一个小的cache暂存将要写回mem的内容
    

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233336332.png" alt="image-20230401233336332" style="zoom:50%;" />

根据block在cache中的位置分配方式，可以将cache分为全关联、组关联、直接关联。

cache中有若干个block的位置，mem中的block在载入cache时会根据某中规则确定位置，由于mem中block数远多于cache，所以一定会出现位置冲突的情况，此时需要进行block的替换。

全相连比较简单，mem中的block在cache中有固定位置，比如mem中的block索引号对cache中block总数取模，就是在cache中的索引。全相连不存在挪动空间，发生冲突的可能性较高。

全相连是block可以到cache的任何位置，较灵活，但是有两个问题，一个是如果全满，替换策略实现起来可能较为复杂。第二个是block在任何位置就意味着访问block时需要检查cache中的每一个位置是不是目标block。

两者之间有折中策略：n路组相连（n-way set associative）

cache分为若干个组，每个组内都有n个block的位置，mem中的block会先根据索引确定其分到cache的哪个组，在组内随意给位置，也就是“组外直接相连，组内全相连“。一般n是二次幂。这种访问时，只需要在组内检查block即可。

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233342794.png" alt="image-20230401233342794" style="zoom:50%;" />

实际工作中，给出的是需要访问的字节的地址（比如64位，32位，可能是虚地址），然后对这个地址进行拆分解析，分为Tag、Index、Byte offset三部分。

其中Index确定的是cache中对应block的位置，在cache中并不实际存储。TAG是在cache中也会存储的部分（和Data存在一起），用于确定block中的内容是否是真正匹配的内容。根据相连方式的不同，先根据index找到所有cache中匹配的block，然后比较cache中的TAG和目标地址的TAG，找到一致的则说明命中。

注意全相连中index为0位（不存在），全相连中index的位数等于log2（cache的block数）。

byteoffset用于从cache的block中提取目标字节。

具体解析方式为：

**Index位数 = log2（组数）=  log2（cache的block数 / 每组的block数）**

**Offset位数 = log2（block内的字节数）**

**Tag位数 = mem地址位数 - Index位数 - Offset位数**



## Cache优化

性能指标

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233413698.png" alt="image-20230401233413698" style="zoom:50%;" />



<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233429842.png" alt="image-20230401233429842" style="zoom:40%;" />

可以看到跟存储有关的主要影响指标是内存访问率和AMAT，前者主要跟程序有关，后者是主要控制对象。

AMAT即平均访存时间，AMAT = Hit time + Miss Rate * Miss Penalty，提高cache性能主要就是围绕

**减少hit time、减少miss rate、减少miss penalty**三者进行，另外还有并行技术

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233439537.png" alt="image-20230401233439537" style="zoom:50%;" />

### Reduce Miss Penalty

- multilevel cache多级缓存

    设立多级cache，hight level的cache中找不到，就去low level的cache中去找。在这种情况下，miss penalty会向下层cache迭代计算。要注意题目说的miss rate值得是整体访存miss还是单层cache的miss 。

- Early start，critical word first提前启动、关键字优先

    CPU所需要的往往是字节或者word，但cache却是以block为单位移动内存块的，所以访问word如果miss，不必马上搬运word所在的整个block进入cache，再交给CPU计算，而是优先搬运需要的那个word，剩余的慢慢搬，这就是关键字优先。

    另外，一旦关键字搬入cache，可以立即将其交给CPU运算，而不必等待其余的内容全搬入cache后再运算，这就是提前启动。

- Priotity to Read Miss Over Write读miss优先

    （比较难理解的一个）：写操作（未必是写miss，哪怕是写hit），往往需要花费比read更长的时间。对于一条Read miss指令，如果它前面有**不相关的写指令**，那么可以不等前面的写指令执行完毕，就开始进行read miss之后的cache操作，从而节省时间。此时指令乱序执行完毕。

- Merging write buffer合并写buffer

    指write buffer一次性写回较多的word。合并写效率较高可以减少写时间。宏观少减少miss peanalty

- Victim cache

    victim cache通常是另设的一个较小的、全相连的cache，存的内容是近期主cache中被替换(牺牲)掉的块。当miss的时候，并不是直接去内存中找，而是先去victim cache中查看是否有需要的块，如果有，将其跟内存中的对应block交换。

    <img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233504064.png" alt="image-20230401233504064" style="zoom:40%;" />

    注意这和多级cache不同，实际上地址跟主cache、victim cache的访问是并行的。

### Reduce Miss Rate

要想减少miss rate首先要知道miss的主要原因：3C原因

Compulsory：冷启动，即首次进入代码执行段，cache中没有任何数据的情况，初始情况下miss会非常多。

Capacity：cache容量有限

Conflict：由于cache相连模式，一个区域只能放有限个块，多余的块会被替换掉

- Larger block size

    增大block size，但注意这是U型作用的，如果block太大将导致跳转适应性差，并且较大的block会导致一次从mem搬运较为耗时，增大MP

- Larger cache size

    采用更大cache，这样可以容纳更多的block，block也可以更大。缺点是cache地址线增多容易导致cache访问速度（hit time）变慢。

- Higher Associaticity

    更高的关联度，从而减少confict的发生。但是如果关联度太高，硬件实现上会比较麻烦，并且Tag的比较增多，同样会降低cache的访问速度。

- Way Prediction和伪相连

    在全相连和组相连中，需要检查每个组中的block（way）来找到到底哪个是所需要的块。可以在cache中额外存储用于预测way选择的比特位，如果预测正确直接选择对应way即可（共1个时钟），否则尝试其他way，修改predictor的值（共3个时钟）。

    所谓伪相连，值得是在直接相连的基础上，如果发生miss，按某种组相连的方式查看组内其他block是否hit，如果有，进行一次“伪命中”。这是平衡直接相连访问快和组相连miss rate低的方法。

- 编译技术

    <img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233514262.png" alt="image-20230401233514262" style="zoom:50%;" />

    编译提升性能的主要思路是利用空间局部性原则和时间局部性原则，尽可能让相近的数据被连续访问、让同样的数据被短时间内访问。

    

### Reduce Hit Time

- Small / Simple cache

    用较小的、简单（全相连）的cache，可以让hit更为快捷。但是往往会增大miss rate。

- 避免频繁地址转换

    由于虚拟内存机制的存在，CPU访问的目标地址是虚地址，而最终需要根据物理地址到内存中寻找数据。所以可以直接在cache中存虚拟地址而非物理地址。

- 并行访问缓存

- Trace cache

    尽量让需要连续访问的（大概率非连续）地址，按照trace cache的某种规则，映射在cache的同一个block里。





# Memory内存

## 带宽及其提升方法

Bandwidth带宽：每个时钟内存能输出的字节数

Bandwidth = bytes in word / word access cycle = (4 * 8) / (send address + access a word + send a word)

提升带宽的方法

- 更宽的主内存

    <img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233524674.png" alt="image-20230401233524674" style="zoom:40%;" />

    可以并行地同时从多个bus中取数据，这样取多个word的时候，可以并行取，线性提高带宽。

- 交错内存

    <img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233536043.png" alt="image-20230401233536043" style="zoom:40%;" />

    

- 独立交错内存

    ？？



# 分支预测

CPU中经常需要进行分支，分支结果分为两种，跳转（1）、不跳转（0）。如果进行分支预测，可以根据预测结果决定下一个PC的值，如果预测错误，需要放弃执行一些指令（分支指令开始到结束这段时间执行的指令），这些个时钟就浪费了。如果预测成功则可以节省这些时钟，所以设计合理的预测器，让预测准确率提高，是很重要的。

## 预测器的位数bit

一个预测器由几个bit组成，决定了预测器的状态数。最简单的预测器就是单个单比特预测器，即0预测不跳转，1预测跳转。发生跳转将预测器置为1，否则置为0。这种预测器最简单、最不“坚定”。

2bit预测器就有了00、01、10、11四种状态，涉及到状态转化图，主要有两种。

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233546693.png" alt="image-20230401233546693" style="zoom:40%;" />



基本思路是一直跳转的情况下，至少不跳转两次才预测不跳转，反之同理。可以看到，多比特预测器的状态转化策略不唯一。2bit预测器用的比较多。



## 预测器的个数 Correlating Predictor

单预测器有缺点：每次只根据前面一次的跳转结果进行预测。

预测器的内部个数可以增多为n个（2的幂），根据前面log2 n次的跳转结果（共有n种可能）进行预测，并且针对每种跳转结果，使用特定的预测器进行预测。

第i个预测器只针对前面log2 n次跳转情况为情况i时进行预测，其他情况下不进行预测，其他情况下的跳转结果也不会影响这个预测器的值（1<=i<=n）

每个预测器的位数可以自行决定，一般都是2bit。



# 流水线的动态调度（重点?）

在宽流水线中，EX阶段可能有很多个时钟，不同类型指令时钟数可能不一样，整个流水线不像标准五级那样整齐。难以在硬件设计上（如流水线寄存器bypass、double pump）来完全解决，必须设计一种灵活的动态方法来调度指令执行。

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233607368.png" alt="image-20230401233607368" style="zoom:40%;" />



流水线五个流程被改成这样。注意issue（指令发射）是重要概念，指的是指令从等待队列真正进入流水线开始解析执行的过程。指令的发射有条件。

## 记分板Scoreboard

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233622382.png" alt="image-20230401233622382" style="zoom:40%;" />



<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233633637.png" alt="image-20230401233633637" style="zoom:50%;" />

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233647861.png" alt="image-20230401233647861" style="zoom:40%;" />

记分板维护三张表（如图，其实真正硬件上存在的只有下面两张表，第一张是辅助理解的）

- 第二张表：功能单元状态表

  行：功能单元，如乘法器、加法器。

  Busy：功能单元是否繁忙，发射的指令如果需要用到需要置为yes，直到执行完毕（而不是只在EX阶段）。此位用来杜绝结构竞争。

  Op：指令操作类型，不太重要

  Fi：当前指令目的寄存器号

  Fj、Fk：当前指令源寄存器号

  Qj、Qk：如果对应的Rj（或者Rk）位No，此数据显示正在占用这个源操作数的功能单元。

  Rj、Rk：（Ready位）两个对应的源操作数是否准备好

  

- 第三张表（目的寄存器表）：记录每个寄存器是否正在被当作目的寄存器，如果是，记下这个操作的功能单元。

- 条件：

  - 发射条件：功能单元不busy（避免结构竞争），且目的寄存器没有被其他指令当作目的寄存器（解决WAW竞争）
  - RO条件：Rj、Rk都为yes，即两个操作数均准备好。其中Ri，Rj会被写指令在发射阶段置为No（解决RAW竞争）。
  - EX条件：无
  - 写回条件：对于这条指令的目的寄存器，它如果同时是另一条指令的源寄存器，那么它在那条指令中一定不ready。也就是说写回指令的目的寄存器不能作为其他指令的可用源寄存器（解决WAR竞争）

- 状态的清除

  在指令WB完毕后会清除相关行的所有信息（以及目的寄存器表），还要将其它行的以自己的目的寄存器为源寄存器的Rj/Rk值设为Yes。另外在一条指令执行过程中，RO完毕后会清除其Rj、Rk的信息, 同时会置Qj、Qk为0（清除时由于已经是RO阶段，R值必为Yes。已经读到，不必再管，其实清除与否不重要）。



## Tomasulo算法

记分板算法的最大问题在于对于假竞争，即WAW和WAR竞争也会stall。这两种竞争本质上是写覆盖，而非读写依赖。写覆盖问题完全可以采用另找一个目的寄存器的方法（算法上表现上寄存器重命名）来解决。

**寄存器重命名**是tomasulo算法的核心思想

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233702306.png" alt="image-20230401233702306" style="zoom:40%;" />

- 保留站 Reservation Station

    对于每一种操作（如占用加法器的加、占用乘法器的乘），各开辟一个保留站，有若干行，每一行都会记录类似记分板算法中第二张表的行信息。

​		<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233714595.png" alt="image-20230401233714595" style="zoom:40%;" />

​	一个重大区别是，用Vj、Vk代替Fj、Fk，记录的不是源寄存器编号，而是源寄存器在取出时候的**值**。当然，也要记录该寄存器是否正在被作为其他指令的目的寄存器，方法是在Qj、Qk中存入写入该源寄存器的指令的对应保留站及其行索引，如果为0代表ready，不需要额外的R值帮助记录。

​	一条指令一旦发射，其源寄存器的值已经存入保留站，不用再考虑被改写的问题，所以解决了WAW和WAR，真依赖RAW则依靠保留站的状态转换来维持。另外，由于发射后不需要再从寄存器组中获得寄存器的值，所以RO阶段消失了（也可以理解为RO跟IS合并了，寄存器读到保留站中而非流水线内部），现在只有IS、EX、WB三个阶段。

​	以上讲的是算术运算的保留站，Load指令的保留站较为简单，只有Busy位和Address，Address也是在IS阶段直接读出。在tomasulo算法中，指令的发射条件很宽松（保留站有位置），如果源操作数依赖于之前的写指令（算术或者LD），只要设定相应Q值、V值不设即可，知道依赖的指令EX或WB完毕，将其Q设为0，V写入真正的寄存器值，此时才可以进行EX（记分板上RO条件苛刻，EX无条件）。

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233731103.png" alt="image-20230401233731103" style="zoom:40%;" />

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233740970.png" alt="image-20230401233740970" style="zoom:40%;" />



## 投机Speculation

tomasulo和记分板都对分支跳转没有很好的处理。

方法是改进tomasulo算法，保留站改为Reorder Buffer(ROB), 支持乱序提交。

此时在IS、EX、WB阶段之后，还有一个Commit阶段，commit的作用是：

When instruction at head of reorder buffer & result present, update register with result (or store to memory) and remove instr from reorder buffer. **Mispredicted** **branch flushes reorder** buffer (sometimes called “graduation”)

也就是只有ROB头部的指令才能被提交，真正地改写寄存器或者内存，WB阶段只能发送信号到总线上并通知ROB。之所以需要引入Commit阶段，是因为投机由于是按预测结果来执行指令，如果预测失败，需要放弃提前执行的这些指令，所以投机时，指令可能并非按序发射/执行！但是提交commit可以保证是按序的。

ROB中记录的数据：目的寄存器及其值(未计算出时无效)、指令信息、是否执行完成(执行完成才可以提交)

提交过程中，将记录的目标寄存器的值写入寄存器，只能提交ROB头部。

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233755148.png" alt="image-20230401233755148" style="zoom:50%;" />







## 多发射

- 超标量superscalar

    一个时钟上同时发射多条指令。

    N-发射时，每个时钟可能发射0-N条指令，需要进行N(N-1)次比较。

    <img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233810248.png" alt="image-20230401233810248" style="zoom:40%;" />

    

    在填写指令周期表时，要注意：

    - 各功能单元个数，算术指令ALU、地址计算ALU、跳转计算ALU是否有重合，注意避免结构竞争问题

    - 是否采用投机（speculation）策略，如果不是，则跳转指令后面的指令需要等待跳转指令的EX结束后才能进入EX阶段。如果采用了投机，后面的指令可以无视跳转直接进入EX，只不过后面需要ROB保证顺序提交。也就是tomasulo with speculation。

        <img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233823952.png" alt="image-20230401233823952" style="zoom:40%;" />

    

- VLIWV(ery Long Instruction Word超长指令字)

    每个时钟，拼装多条指令到一条长指令上，发射这条长指令



## DLP（Data Level Parallelism）

提高并行度的方法主要三个层次：指令层次、数据层次、线程层次。

流水线及其优化、多发射、超标量等都是指令层次的概念。

- Vector Processor

    <img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233835357.png" alt="image-20230401233835357" style="zoom:40%;" />

    

    SIMD有高效、适合编程者思考的有点，经常使用在移动设备中。

    

    

## 线程并行

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233847202.png" alt="image-20230401233847202" style="zoom:50%;" />

可以将CPU需要处理的任务划分为若干线程，每个线程拥有独立的PC、**寄存器组**、栈（独立内存空间）。

在流水线中，将不同线程的指令交叉执行，从而完全避免冲突的发生（注意寄存器组、内存空间的互不干扰性）。

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233853985.png" alt="image-20230401233853985" style="zoom:50%;" />

实际中需要考虑的一个问题是如何安排不同线程指令的顺序。分为固定交错、软件控制交错、硬件调度三种方法。

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233858959.png" alt="image-20230401233858959" style="zoom:40%;" />



## 多处理器

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233908030.png" alt="image-20230401233908030" style="zoom:40%;" />

SMP：symmetrical multi-processing

UMA：uniform memory access

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233917728.png" alt="image-20230401233917728" style="zoom:50%;" />

DSM：distributed shared-memory

NUMA：non-uniform memory access

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233924039.png" alt="image-20230401233924039" style="zoom:50%;" />



## Cache Coherence 缓存一致性

多处理器的基本问题：Pi对cache的写可能无法及时同步到其他进程的cache中。

write through中，直写到memory，但是其他进程的cache如果有这个数据，可能并不知道要刷新内部对应数据的值。

write back中，除了直写的问题，还有一个问题是如果写数据没及时写入mem，其他cache里面没有这个数据，miss的时候载入cache，载入的依然是旧值。

- Snoopy算法

    适用于SMP，写的时候会在总线上进行广播broadcast。分为三种协议：写无效、写广播、写串行。其中写广播是SMP中的直写，同时广播给其他cache更新值。写串行不重要。重点研究写无效write invalidate protocol

    <img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233933031.png" alt="image-20230401233933031" style="zoom:40%;" />



Cache的任一个cacheline，都有三种状态

Invalid：该数据无效（旧值/无意义）

Shared：数据有效，且跟disk一致

Exclusive：数据有效，为最新值，尚未写回disk

状态转化如图，引起状态转化的有CPU的读写请求和由其引起的总线信号两种。

注意在CPU发送请求后，首先该CPU对应的cache会根据状态图进行转换，然后可能广播信号到总线，其他cache接受到信号再完成状态转化。

注意，**在Shared状态的block发生write hit变成Exclusive时，给总线上依然传递write miss信号**，将其它缓存对应block无效化。



- Directory-based cache coherence目录算法

    适用于DSM，即各自的节点拥有自己的mem，不存在统一的mem。不同mem需要通过网络共享某些数据。此时cache中存的未必是本地mem的数据，可能是其他node的，需要目录记录。

    每个节点拥有一个目录，用于记录数据信息、帮助维护cache一致性。

    对于一个CPU请求数据X，节点有以下三个概念：

    local node本地节点：发送CPU请求的节点。

    home node家节点：数据X隶属的mem所在的节点。

    remote node远程节点：cache中拥有X的节点

    这三者不一定是分开的，同一个节点可能同时是两个或三个node类型。

    <img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233942649.png" alt="image-20230401233942649" style="zoom:40%;" />

    <img src="/Users/jerryliterm/Database/Notes/assets/image-20230401233954072.png" alt="image-20230401233954072" style="zoom:40%;" />

    P代表节点编号，A请求的地址。

    <img src="/Users/jerryliterm/Database/Notes/assets/image-20230401234007850.png" alt="image-20230401234007850" style="zoom:40%;" />

    目录的一行中记录是依次是：

    **本节点**mem的数据地址X + 数据状态 + 节点list。目录中不记录数据本身。

    数据状态分为U(uncached), S(shared), E(Exclusive)三种，意义与cache状态不完全一样。

    Uncached：X不在任何节点cache的有效cacheline中

    Shared：X与自身内存中的值一致，且至少一个节点cache中有该有效数据。list中是所有cache中有X的节点号
    
    Exclusive：自身cache中的X与内存中不一致。list中存的是cache中拥有最新值的节点号（唯一）
    
    状态转化图如上。



## 数据同步

 <img src="/Users/jerryliterm/Database/Notes/assets/image-20230401234017794.png" alt="image-20230401234017794" style="zoom:40%;" />

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401234038002.png" alt="image-20230401234038002" style="zoom:40%;" />

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401234106305.png" alt="image-20230401234106305" style="zoom:50%;" />

<img src="/Users/jerryliterm/Database/Notes/assets/image-20230401234111577.png" alt="image-20230401234111577" style="zoom:40%;" />



















