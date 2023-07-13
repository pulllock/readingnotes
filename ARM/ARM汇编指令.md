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