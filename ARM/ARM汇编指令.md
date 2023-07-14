# 指令的编码格式

`<opcode> {<cond> {s} <Rd>,<Rn> {,<operand2>}}`，其中`<>`中的是必选项，`{}`中的可选项

- `<opcode>`：是机器码的操作助记符，比如MOV、ADD等
- `cond`：执行条件，ARM为减少分支跳转指令个数，允许类似BEQ、BNE等形式的组合指令
- `s`：是否影响CPSR寄存器中的标志位，如SUBS指令会影响CPSR寄存器中的N、Z、C、V标志位，而SUB指令则不会
- `Rd`：目标寄存器
- `Rn`：第一个操作数的寄存器
- `operand2`：第二个可选操作数

# 存储访问指令

- `LDR R1,[R0]`：将R0中的值作为地址，将该地址上的数据保存到R1
- `STR R1,[R0]`：将R0中的之作为地址，将R1中的值存储到这个内存地址
- `LDRB/STRB`：每次读写一个字节，`LDR/STR`默认每次读写4个字节
- `LDM/STM`：批量加载/存储指令，在一组寄存器和一片内存之间传输数据
- `SWP R1,R1,[R0]`：将R1和R0中地址指向的内存单元中的数据进行交换
- `SWP R1,R2,[R0]`：将`[R0]`存储到R1，将R2写入到`[R0]`这个内存存储单元

不同类型的堆栈：

- FA（Full Ascending）满递增堆栈

- FD（Full Descending）满递减堆栈

- EA（Empty Ascending）空递增堆栈

- ED（Empty Descending）空递减堆栈

- `LDMFD SP!,{R0-R2,R14}`：将内存栈中的数据依次弹出到R14，R2，R1，R0

- `STMFD SP!,{R0-R2,R14}`：将R0，R1，R2，R14依次压入内存栈

- `PUSH {R0-R2,R14}`：R0，R1，R2，R14依次压入内存栈

- `POP {R0-R2,R14}`：将内存栈中的数据依次弹出到R14，R2，R1，R0

# 数据传送指令

MOV指令格式如下：

- `MOV {cond} {S} Rd, operand2`
- `MVN {cond} {S} Rd, operand2`：MVN指令用来将操作数operand2按位取反后传送到目标寄存器Rd，操作数operand2可以是一个立即数，也可以是一个寄存器

指令中各部分含义如下：

- `{cond}`：条件指令可选项
- `{S}`：用来表示是否影响CPSR寄存器的值，如MOVS指令会影响寄存器CPSR寄存器的值，而MOV则不会
- `Rd`：目标寄存器
- `operand2`：第二个可选操作数

用法如下：

- `MOV R1, #1`：将立即数1传送到寄存器R1中
- `MOV R1, R0`：将R0寄存器中的值传送到R1寄存器中
- `MOV PC, LR`：子程序返回
- `MVN R0, #0xFF`：将立即数0xFF取反后赋值给R0
- `MVN R0, R1`：将R1寄存器的值取反后赋值给R0

# 算术逻辑运算指令

指令格式如下：

- `ADD {cond} {S} Rd, Rn, operand2`：加法
- `ADC {cond} {S} Rd, Rn, operand2`：带进位加法
- `SUB {cond} {S} Rd, Rn, operand2`：减法
- `AND {cond} {S} Rd, Rn, operand2`：逻辑与运算
- `ORR {cond} {S} Rd, Rn, operand2`：逻辑或运算
- `EOR {cond} {S} Rd, Rn, operand2`：异或运算
- `BIC {cond} {S} Rd, Rn, operand2`：位清除指令

用法如下：

- `ADD R2, R1, #1`：R2=R1+1
- `ADC R1, R1, #1`：R1=R1+1+C（其中C为CPSR寄存器中进位）
- `SUB R1, R1, R2`：R1=R1-R2
- `SBC R1, R1, R2`：R1=R1-R2-C
- `AND R0, R0, #3`：保留R0的bit0和1，其余位清除
- `ORR R0, R0, #3`：置位R0的bit0和bit1
- `EOR R0, R0, #3`：反转R0中的bit0和bit1
- `BIC R0, R0, #3`：清除R0中的bit0和bit1

# operand2操作数说明

operand2操作数可以有两种格式：

- `#constant`：常数
- `Rm{, shift}`：寄存器+偏移的形式

`Rm{, shift}`可以选择位移方式如下：

- `#constant,n`：将立即数constant循环右移n位
- `ASR #n`：算术右移n位，n的取值范围`[1, 32]`
- `LSL #n`：逻辑左移n位，n的取值范围`[0, 31]`
- `LSR #n`：逻辑右移n位，n的取值范围`[1, 32]`
- `ROR #n`：向右循环移n位，n的取值范围`[1, 31]`
- `RRX`：向右循环移1位，带扩展
- `type Rs`：仅在ARM中可用，其中type指ASP、LSL、LSR、ROR，Rs是提供位移量的寄存器名称

使用方式如下：

- `ADD R3, R2, R1, LSL #3`：R3=R2+R1<<3
- `ADD R3, R2, R1, LSL R0`：R3=R2+R1<<R0
- `ADD IP, IP, #16, 20`：IP=IP+立即数16循环右移20位

# 比较指令

比较指令的运算结果会影响CPSR寄存器的N、Z、C、V标志位，比较指令的格式如下：

- `CMP {cond} Rn, operand2`：比较两个数大小
- `CMN {cond} Rn, operand2`：取负比较

使用示例如下：

- `CMP R1, #10`：R1-10，运算结果会影响CPSR寄存器的N、Z、C、V标志位
- `CMP R1, R2`：R1-R2，比较结果会影响CPSR寄存器的N、Z、C、V标志位
- `CMN R0, #1`：R0-(-1)，将立即数取负，然后比较大小

比较结果如下：

- 比较指令的运行结果Z=1时，表示运算结果为0，两个数相等
- 比较指令的运行结果N=1时，表示运算结果为负
- 比较指令的运行结果N=0时，表示运算结果为非负

# 条件执行指令

条件码如下：

| 条件码   | CPSR标志位  | 说明        |
| ----- | -------- | --------- |
| EQ    | Z=1      | 相等        |
| NE    | Z=0      | 不相等       |
| CS/HS | C=1      | 无符号数大于或等于 |
| CC/LO | C=0      | 无符号数小于    |
| MI    | N置位      | 负数        |
| PL    | N清零      | 整数或零      |
| VS    | V置位      | 溢出        |
| VC    | V清零      | 未溢出       |
| HI    | C置位，Z清零  | 无符号数大于    |
| LS    | C清零，Z置位  | 有符号数小于或等于 |
| GE    | N=V      | 有符号数大于或等于 |
| LT    | N!=V     | 有符号数小于    |
| GT    | Z清零，N=V  | 有符号数大于    |
| LE    | Z置位，N!=V | 有符号数小于或等于 |
| AL    | 忽略       | 无条件执行     |
| NV    | 忽略       | 从不执行      |

# 跳转指令

跳转指令格式如下：

- `B {cond} label`：跳转到标号label处执行，B跳转指令的跳转范围大小为`[0, 32MB]`，可往前跳也可以往后跳，主要用在循环、分支结构的汇编程序中
- `B {cond} Rm`：寄存器Rm中保存的是跳转地址
- `BL {cond} label`：BL跳转指令表示带链接的跳转，跳转之前BL指令会先将当前指令的下一条指令地址（返回地址）保存到LR寄存器中，然后再跳转到label出执行。BL指令一般用在函数调用的场景
- `BX {cond} label`：BX表示带状态切换的跳转，Rm寄存器中保存的是跳转地址，要跳转的目标地址处可能是ARM指令也可能是Thumb指令，处理器根据`Rm[0]`位决定是切换到ARM状态还是Thumb状态
  - 0：表示目标地址处是ARM指令，在跳转前要先切换至ARM状态
  - 1：表示目标地址处是Thumb指令，在跳转前要先切换至Thumb状态
- `BLX {cond} label`

# ARM寻址方式

- 寄存器寻址
- 立即数寻址
- 寄存器偏移寻址
- 寄存器间接寻址
- 基址寻址
- 多寄存器寻址
- 相对寻址

# ARM伪指令

- `ADR R0, LOOP`：将标号LOOP的地址保存到R0寄存器中
- `ADRL R0, LOOP`：中等范围的地址读取
- `LDR R0, =0x30008000`：将内存地址0x30008000赋值给R0
- `NOP`：空操作，用于延时或插入流水线中暂停指令的运行

# ARM汇编程序格式

```armasm
AERA COPY,CODE,READONLY ;当前段属性为代码段，只读，段名为COPY
    ENTRY
START
    LDR R0,=SRC
    LDR R1,=DST
    MOV R2,#10
LOOP
    LDR R3,[R0],#4
    STR R3,[R1],#4
    SUBS R2,R2,#1
    BNE LOOP

AREA COPYDATA,DATA, READWRITE ;数据段，读写权限，段名为COPYDATA
SRC DCD 1,2,3,4,5,6,7,8,9,0
DST DCD 0,0,0,0,0,0,0,0,0,0
    END
```

# 符号与标号

可以使用符号来标识一个地址、变量或数字常量。当用符号来标识一个地址时，这个符号称为标号。可以使用`%{F|B|A|T} N{routename}`来引用局部标号，其中`{}`部分是可选项，N表示局部标号，其余参数如下：

- `%`：引用符号，对一个局部标号产生引用
- `F`：指示编译器只向前搜索
- `B`：指示编译器纸箱后搜索
- `A`：指示编译器搜索宏的所有宏命令层
- `T`：指示编译器搜索宏的当前层
- `N`：局部标号的名字
- `routename`：局部标号作用范围名称，使用ROUT定义

# 伪操作

- `AREA`：定义一个段
- `GBLA a`：定义一个全局算术变量a，并初始化为0
- `ENTRY`：指定汇编程序的执行入口
- `a SETA 10`：给算术变量a赋值为10
- `GBLL b`：定义一个全局逻辑变量b，并初始化为false
- `b SETL 20`：给逻辑变量b赋值为20
- `GBLS STR`：定义一个全局字符串变量STR，并初始化为0
- `STR SETS "xxxx"`：给变量STR赋值为XXXX
- `LCLA a`：定义一个局部算术变量a，并初始化为0
- `LCLL b`：定义一个局部逻辑变量b，并初始化为false
- `LCLS name`：定义一个局部字符串变量name，并初始化为0
- `name SETS "xxxx"`：给局部字符串变量赋值
- `DATA1 DCB 10,20,30,40`：分配一片连续的字节存储单元并初始化
- `STR DCB "xxxx"`：给字符串分配一片连续的存储单元并初始化
- `DATA2 DCD 10,20,30,40`：分配一片连续的字存储单元并初始化
- `BUF SPACE 100`：给BUF分配100字节的存储单元并初始化为0
- `ALIGN`：地址对齐
- `CODE16/CODE32`：指示编译器后面的指令为THUMB/ARM指令
- `END`：告诉编译器源程序已经到了结尾，停止编译
- `EQU`：赋值伪指令，类似宏，给常量定义一个符号名
- `EXPORT/GLOBAL`：声明一个全局符号，可以被其他文件引用
- `IMPORT/EXTERN`：引用其他文件的全局符号前，要先IMPORT
- `GET/INCLUDE`：包含文件，并将该文件当前位置进行编译，一般包含的是程序文件
- `INCBIN`：包含文件，但不编译，一般包含的是数据，配置文件等

# ATPCS规则

ATPCS全称ARM-Thumb Procedure Call Standard，定义了ARM子程序调用的基本规则以及堆栈的使用约定。子程序调用具体规则如下：

- 子程序间要通过寄存器R0-R3传递参数，当参数个数大于4个时，剩余的参数使用堆栈来传递
- 子程序通过R0-R1返回结果
- 子程序中使用R4-R11来保存局部变量
- R12作为调用过程中的临时寄存器，一般用来保存函数的栈帧基址，记作FP
- R13作为堆栈指针寄存器，一般记作SP
- R14作为链接寄存器，用来保存函数调用者的返回地址，记作LR
- R15作为程序计数器，总是指向当前正在运行的指令，记作PC

# 在C程序中内嵌汇编代码

ARM使用：

```armasm
__asm
{
    指令 /*注释*/
    ...
    [指令]
}
```

# 常用的GNU ARM伪指令操作

| 伪操作                              | 说明                   |
| -------------------------------- | -------------------- |
| ENTRY(\_start_)                  | 定义汇编程序的执行入口          |
| @、#                              | 注释                   |
| .section .text，"x"               | 定义一个段，a：只读；w：读写；x：执行 |
| .align、.balign                   | 地址对齐方式，按照指定字节数对齐     |
| label:                           | 标号，以冒号结尾             |
| .byte                            | 把字节插入目标文件            |
| .quad、.long、.word、.byte、.short   | 分配不同大小的存储空间，插入目标文件   |
| .string、.ascii、.asciz            | 定义字符串、字符、以NULL结束的字符串 |
| .rept、.endr                      | 重复定义                 |
| .float                           | 浮点数定义                |
| .space 10 FF                     | 分配一片连续的10字节空间，填充为FF  |
| .equ、.set                        | 赋值语句                 |
| .type func,@function             | 指定符号类型为函数            |
| .type num,@object                | 指定符号类型为对象            |
| .include、.incbin                 | 展开头文件、二进制文件          |
| tmp .reg、.unreg r12              | 为寄存器取别名              |
| .pool、.ltorg                     | 声明一个文字池，一般用来存放32位地址  |
| .comm buf, 20                    | 申请一段buf              |
| OUTPUT_ARCH(arm)                 | 指定可执行文件运行平台          |
| OUTPUT_FORMAT("elf32-littlearm") | 指定输出可执行文件格式          |
| #、$                              | 直接操作数前缀              |
| .arch                            | 指定指令集版本              |
| .file                            | 汇编对应的C源文件            |
| .fpu                             | 浮点类型                 |
| .reg                             | 寄存器重新命名              |
| .size                            | 设置指定符号的大小            |

.section伪操作使用格式如下：

```armasm
.section <section name> {, "<flags>"}
.section .mysection "awx" @注释：定义一个可写、可执行的段
.align2
```

预留的段名：

- `.text`：代码段

- `.data`：数据段

- `.bss`：BSS段

- `rodata`：只读数据段


