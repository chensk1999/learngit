

参考教程：https://www.chipverify.com/verilog/verilog-tutorial

在线练习&仿真：https://hdlbits.01xz.net/wiki/Problem_sets

# Verilog简介

Verilog是一种硬件描述语言（Hardware Description Language，HDL），它的语法和C语言相似，但是本质是完全不同的：C语言编译后是顺序执行的指令，而Verilog综合（synthesize）后是一个电路。它可以从不同抽象级别描述一个硬件电路：

| level                                             | description                                                  |
| ------------------------------------------------- | ------------------------------------------------------------ |
| Switch Level，开关级                              | 控制各种mos器件实现功能                                      |
| Gate Level，门级                                  | 用各种门实现                                                 |
| Register Transfer Level，RTL，数据流级            | 描述寄存器数据流，一般用assign和各种wire实现，也有说凡是能综合的都可以叫做RTL级 |
| Behavior Level / Algorithm Level，行为级 / 算法级 | 描述系统算法行为，主要用always块实现                         |
| System Level，系统级                              | 描述模块性能的模型，没看懂怎么实现                           |

Verilog是不断发展的语言，其中比较重要的有Verilog-1995和Verilog-2001这两个标准

# 语法基础

## 常量与标识符

常量的表示方式是`[size]'[base_format][number]`。number可以用下划线隔开以方便阅读。基数包括二进制b、八进制o、十进制d、十六进制h

```verilog
8'b0000_0000;     // 8bit，值为0
9'h1FA;           // 9bit，值为0x1FA = 506
-8'd3;            // 8bit，值为3的反码
"Hello, world!";  // 字符串，ascii，一个字符1字节（基本用不上）

// 几种不好的常量写法
'd132;            // 省略了位宽，综合器会给它分配一个默认位宽（与机器有关，通常是32位）
132;              // 省略了位宽和进制，会被当作十进制，并分配默认位宽
8'b1;             // 定义的位宽和数值宽度不同，不太确定verilog标准有没有规定这种情况

// x表示不定态，z或?表示和高阻态
8'b1x11_xxxx;
16'h4azz;         // 16bit十六进制数，低8位为高阻态

// 符号常量
// 标识符由数字、字母（区分大小写）、下划线、$组成，不能以$或者数字开头，不能与保留关键字重名
`define UART_CNT 10'd1024   // 在整个工程中有效。可以在别的模块使用
parameter MSB = unsigned integer 7;
parameter SIZE = MSB + 1;   // parameter作用于本模块，可用于模块间参数传递
localparam IDLE = 4'b0001;  // localparam不能用于参数传递，常用作状态机状态定义
```

## 数据类型

Verilog描述的是硬件，因此数据类型的定义和高级语言截然不同

### Wire

Wire是连线的抽象，它不能储存值，取值取决于它的驱动电路（Driver）。Wire可以用assign语句来连线（Reg不行）实现组合逻辑电路

```verilog
wire i1, i2;
wire out;
wire [1:0] data;

assign out = i1 & i2;
assign data = {i2, i1};

// 假设给一个net加多个驱动源，当不同驱动源的电平不同时连线的电平为X
assign out 1'b1;   // 如果i1 & i2 == 0，out = X

// implicit assignment
wire out_implicit = i1 & i2;
```

### Reg

Reg是一种能储存数据的电路的抽象，比如锁存器、触发器。它的名字来源于Register

Reg有阻塞赋值与非阻塞赋值两种连接方式

```verilog
a <= c;   // non blocking，不同非阻塞赋值并发执行，顺序无所谓
b = c;    // blocking，这一句执行完之前后面的不能执行（即：这一句的结果会影响之后的值）
```

通常的使用规则是

* 时序电路中使用非阻塞赋值

```verilog
always @(posedge clk) begin
    a <= ~a;
    b <= a;
end
```

<img src="./images/verilog_non-blocking.png" style="zoom: 33%;" />

可见时钟正跳变沿到来时b的D输入还是上一个周期的a，故输出始终有`b == ~a`

* 组合逻辑电路中使用阻塞赋值

```verilog
always @(*) begin
    a = a & in;
    b = a;
end
```

<img src="./images/verilog_blocking.png" style="zoom:33%;" />

如果违反以上两个原则，比如在时序逻辑电路中使用阻塞赋值`a = ~a; b = a;`，综合后就是a经过一个非门连到b的D端，当电路比较复杂的时候会产生一大坨难以理解的逻辑门 / 查找表。在组合逻辑使用非阻塞赋值或者在一个always块中混合使用两种赋值同理

### Scalar, Vector and Array

Scalar是宽度为1bit的数据，Vector是宽度大于1bit的数据，Array是多个数据的阵列

```verilog
wire clk;        // 声明wire scalar
reg [7:0] addr;  // 声明位宽为8的reg vector，方括号中是[msb:lsb]（1个8位寄存器）
reg y [7:0];     // 声明深度为8的scalar reg array（8个1位寄存器的寄存器组）
reg [7:0] mem [1:0][3:0]  // 声明vector reg array，2行4列，每个8bit宽

addr[0] = 1;     // 访问msb
addr[7:4] = 4'hF // part-select
addr = 8'hff     // 整体访问
y[7] = 1'b1;     // 访问Array，注意scalar array不能整体赋值而vector可以
mem[0][0] = 8'haa;  // 将0行0列元素赋值0xAA
```

### 补充

本段介绍一些其他的数据类型（基本不会用到它们，只是补充说明）

Wire是Net的子集，以下列出部分其他的Net类型：

| name      | description                                        |
| --------- | -------------------------------------------------- |
| tri       | 有多个驱动器的线。与wire完全一样，仅用来提高可读性 |
| wand, wor | 线与（wire and），线或（wire or）                  |

Reg是Variable的子集，其他部分Variable类型有

| name    | description                |
| ------- | -------------------------- |
| integer | 32 bits wide               |
| time    | 64 bits wide，好像不可综合 |
| real    | floating point，不可综合   |

## 运算符

和C的运算符基本相同。特别在此列出位运算符

| Symbol     | Performance |
| ---------- | ----------- |
| &          | 与          |
| \|         | 或          |
| ~          | 取反        |
| ^          | 异或        |
| `<<`, `>>` | 左移、右移  |

需要注意：`!`表示逻辑非，`~`表示取反。前者用于条件判断，后者用于位运算。1bit数据两者混用也能得到正确结果，但是为了避免多比特运算时弄错，还是分清楚为好。`&&`和`&`的区别、`||`和`|`的区别同理

还有几个特别的运算符

- 组合位运算符，例如`~&`与非，`~^`同或
- 不定态与高阻态判等：`==, !=`遇到不定态、高阻态时结果为不定态；`===, !==`在两边完全相同（如，都是x或都是z）时得到1，否则为0，不会得到不定态。注意`===, !==`是不可综合的
- 位拼接运算符`{}`，如`{1'b0, 1'b1}, {var_1, 4{var_2}, 2'b10}`。例1结果为`2'b01`，例2中将1个var_1、4个var_2、一个2'b10拼接到一起（拼接必须声明位数）
- 缩减运算符：&B，第一位与第二位与，结果与第三位与，如此类推直到最后一位

## 控制流

注意verilog中没有break和continue，因为电路不可能从一个触发器“跳出”

```verilog
// if分支
if (state == 2'b00) begin
    out <= in0;
    state <= 2'b01;
end else if (state == 2'b01) begin
    out <= in1;
end else
    out <= in2;
    state <= 2'b00;
end

// 另外一种书写格式：如果每个分支只有一行
// 不要两种格式混用
if (state == 2'b00) out <= in0;
else if (state == 2'b01) out <= in1;
else out <= in2;

// case分支
// 两种不可综合的case变种：casez比较z时判断为相等；casex比较x与z时判断为相等
case (state)
    3'b000: begin
        out <= in0;
        state <= 2'b01
    end
    3'b001: out <= in1;
    3'b010, 3'b011: begin
        out <= in1;
        state <= 2'b00;
    end
    default: begin
        out <= in2;
        state <= 2'b00;
    end
endcase

// 注意：如果分支没有else / default，综合器可能会认为没有包含全部情况，从而
// 综合出额外的寄存器

// while循环
while (cnt < 10'h8ff) begin
    cnt <= cnt + 1'b1;
end

// for循环。和while循环不同，它常用于生成多块相同电路
// 没太看懂它的循环变量到底是不是需要综合的东西。弄懂了再补这段笔记
integer i;
for(i = 0; i < 8; i = i + 1) begin
    op[i] <= op[7-i];
end


```

不可综合的循环

```verilog
// forever循环
forever begin
    #100 vin = ~vin;
end

// repeat循环
repeat (100) begin
    #100 vin = ~vin;
end
```



# 块语句

```verilog
begin: block1
    // this is a block
    integer i = 1;
end

// 在块外访问块里的局部变量（不可综合）
top.block1.i

// 禁用块（不可综合），可以在仿真时充当类似break的效果
while (1) begin: block2
    if (i == 0) disable block2
end
```

## Always Block

当信号跳变时将会触发，block中的语句顺序进行，不同的block并行执行。跳变条件可以是边沿敏感（正跳变或者负跳变）或者电平敏感（电平变化），但不能有的信号是边沿敏感、有的信号是电平敏感

```verilog
module tff(input d, input clk, input rst_n, output reg q);
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            q <= 1'b0;
        else
            q <= q;
    end
endmodule
```

可以用always block实现组合逻辑电路：

```verilog
module combo (input a, b, c, d, e,
              output reg z);
    always @ (a or b or c or d or e) begin
        // 或者写成always @ (*)，自动包含对组合逻辑所有输入的电平敏感
        // 注意：z是reg，但综合出来是一个组合逻辑输出，不是寄存器
        z = ((a & b) | (c ^ d) & ~e);
    end
endmodule
```

<img src="./images/verilog_combo_with_always.png" style="zoom: 33%;" />

上图是Quartus综合的结果，reg变量z综合为组合逻辑的输出，而不是寄存器或者触发器。使用Vivado综合，得到的是一个查找表。如果更改项目的硬件，综合结果大概还会有不同吧。同时也可以看出，reg不一定是锁存器/触发器输出。需要具体问题具体分析

仿真时可以用always块产生波形

```verilog
always #10 clk = ~clk;
```

## Generate Block

应该算一种特殊的宏，综合的时候展开成合适的结构（并不常用）

```verilog
genvar i;   // 声明generate variable，综合后它并不存在
generate
    for (i=0; i < 8; i++) begin
        // do something, e.g. instanciate an array of submodules
    end
endgenerate
```

## 并行块

不可综合，常用于仿真的时序控制。并行块所有语句同时执行，因此所有延时都以0开始

```verilog
fork
    vin = 1'b0;
    #10 vin = 1'b1;
    #50 vin = 1'b0;
join
```

## 初始化块

有的硬件可以综合初始化块，有的硬件不能。而且一般也只能综合寄存器初始化的代码

```verilog
initial begin
    // 初始化寄存器
end
```

# 模块（Module）

可综合的代码都要包含在module中。module描述了一个具有特定逻辑功能的电路模块，它包含若干输入和输出端口（Port）以及内部的逻辑电路。以下是一个同步清零的D触发器module：

```verilog
// 旧风格verilog-95的模块声明
module dff(d, clk, rst_n, q);
    // 声明IO
    input        d;
    input        clk;
    input        rst_n;
    output reg   q;

    // 逻辑功能
    always @ (posedge clk) begin
        if (!rst_n)
            q <= 0;
        else
            q <= d;
    end
endmodule
```

<img src="./images/verilog_dff_sync_schematic.png" style="zoom:67%;" />

一个module中可以包含其他的module，比如下例是一个4bit移位寄存器：

```verilog
// verilog-2001风格的模块声明
module shift_reg(
    input    d,
    input    clk,
    input    rst_n,
    output   q
);  // 不要忘了模块声明语句的分号！！

    // 声明内部连线
    wire [2:0] q_net;

    // 实例化(instantiate)D触发器
    dff u0 (d, clk, rst_n, q_net[0]);   // Port connection by ordered list
    dff u1 (.d(q_net[0]), .clk(clk), .rst_n(rst_n), .q(q_net[1]));  // by name
    dff u2 (.d(q_net[1]), .clk(clk), .rst_n(rst_n), .q(q_net[2]));
    dff u3 (.d(q_net[2]), .clk(clk), .rst_n(rst_n), .q(q));
endmodule
```

<img src="./images/verilog_dff_shift_reg_schematic.png" style="zoom:60%;" />

端口连线时，可以用如`.name()`的方式表示引脚悬空

`input`和`inout`类型的端口要被外部电路驱动，因此不能是reg类型

开发时，会用一个top-level module包含整个系统，顶层模块不会被其他的模块实例化。模拟仿真时使用的testbench也算一种顶层模块

```verilog
// 定义带有parameter的模块
module counter #(
    parameter N = 10
)(
    input clk, rst_n, en,
    output reg[N-1:0] out
);
    
    always @(posedge clk) begin
        if (!rst_n) out <= 0;
        else if (en) out <= out + 1'b1;
        else out <= out;
    end
endmodule

// 实例化带有parameter的模块
counter #(
    .N(4)
) cnt4b (
    .clk(clk),
    .rst_n(rst_n),
    .en(en)
);
```



# 任务（Task）与函数（Function）

任务和函数主要用于仿真。实现可综合的代码时，应该使用模块，不要使用任务与函数；在写不可综合的代码时，任务与函数比模块写起来方便一些

```verilog
module testbench ();
    reg [7:0] Mux_Addr_Data = 0;
    reg       Addr_Valid = 1'b0;
    reg       Data_Valid = 1'b0;

    // 定义task
    task do_write;
        // 向addr地址写入data
        input [7:0] addr, data; 
        begin
            Addr_Valid = 1'b1;
            Mux_Addr_Data = addr;
            #10;
            Addr_Valid = 1'b0;
            Data_Valid = 1'b1;
            Mux_Addr_Data = data;
            #10;
            Data_Valid = 1'b0;
            #10;
        end
    endtask
    
    // 定义function
    // 函数一般用于计算/组合逻辑，不能使用时序控制，也不能调用任务
    // 至少一个输入，返回值是与函数名同名变量
    // 函数不能递归调用，也不能在多处被同时调用
    function calc_parity(input [7:0] data);
        calc_parity = ^data;
    endfunction

    initial begin
        #10;
        do_write(8'h00, 8'hAB);     // 调用task
        parity = calc_parity(8'hAB);// 调用function
        do_write(8'h01, 8'hBC);
        do_write(8'h02, 8'hCD);
    end
endmodule
```



# 仿真

## testbench编写

一般定义一个专门用来仿真的testbench顶层模块，产生几个虚拟的信号提供给被测试的模块，观察测试输出是否正确。有许多只能在仿真时进行的操作，以下列出常用的几个

```verilog
module testbench();
    wire clk, foo, bar;             // 声明仿真时需要用到的变量
    some_sub_module sub(foo, bar);  // 被仿真的模块
    
    always #10 clk = ~clk;

    initial begin
        clk <= 0;        // 仿真时变量都可以直接赋值，而且如果不赋初值就会变成X
        foo <= 0;
        bar <= 0;
        #10 foo = 1;     // 显式延时，不可综合
        #20 bar = 1;
        #400 $finish;    // 结束仿真
    end

    always @(posedge clk) begin
        $display("%d", foo);
        $write("%d", bar);  // display输出完会自动换行，write不会
    end
endmodule
```

ModelSim使用

1. File - New - Library，选择文件夹（之后仿真产生的文件都会放在这里。也可以给它起别名），确认之后这个文件夹会显示在Library面板
2. Compile - Compile，选择所有需要仿真的源文件。编译成功之后Library面板下会多出它们编译出的module
3. 双击top-level module，会打开Wave面板，Sim面板和Objects面板
4. 在Objects面板选择需要查看的波形，添加到Wave面板，Simulate - Run

# 状态机

## 简述

时序逻辑电路都可以称作状态机。它的结构可以抽象成下图

![新建 Microsoft Visio Drawing](images/FSM.svg)

一般将状态机分类为

- Mealy状态机：输出与输入有关，`O = O(I, S)`
- Moore状态机：输出与输入无关，`O = O(S)`

Moore状态机是完全同步的，因此稳定性和抗干扰能力都更强：状态S是与时钟同步的，因此`O(S)`和时钟不会同时跳变，亚稳态可控。反之，对于Mealy型，如果`I`在时钟沿附近跳变，`O(I, S)`有可能和时钟同时跳变，在下一级引入亚稳态

用状态图表示：

```mermaid
stateDiagram-v2
    s1: 状态1/Moore输出
    s2: 状态2/Moore输出
    s1 --> s2: 输入/Mealy型输出
```

## 状态机的写法

```verilog
//------------------------------------------------------------------------------
// 一段式状态机
// 把状态转移条件、状态转换和输出写在一个块内
// 代码任意性较高，不好分析、难以约束时序，且容易写出不规范的代码
// 除非是几行就能写完的小状态机，尽量不要用
wire in;
reg state;
reg out;
always @(posedge clk) begin
    case (state)
        STATE_IDLE: begin
            if (start) begin
                state <= STATE_1;   // 状态转换条件 & 状态转换
                out <= 1;           // 输出
            end else begin
                state <= STATE_IDLE;
                out <= 0;
            end
        end
        STATE_1:
            state <= IDLE;
            out <= 0;       // 注意：是下一个状态的输出。在此对应IDLE态输出
        end
        default: begin
            state <= STATE_IDLE;
            out <= 0;
        end
    endcase
end

//------------------------------------------------------------------------------
// 两段式状态机
// 把时序电路（状态转换）写在一个块内，组合电路（状态转移条件、输出）写在另一个块内
// 代码最简洁，且比较规范。但有组合逻辑输出，输出有毛刺
reg state;
reg next_state;

always @(posedge clk) begin
    // 状态转换
    state <= next_state;
end

always @(*) begin
    // 状态转移条件
    case (state)
        STATE_IDLE: next_state = start ? STATE_1 : STATE_IDLE;
        STATE_1:    next_state = STATE_IDLE;
        default:    next_state = STATE_IDLE;
    endcase
    
    // 输出。注意这种写法的输出是组合电路，有毛刺
    // 把这个case放到第三个always块里面，看上去也像是三段式，但效果并不一样
    case (state)
        STATE_IDLE: out = 0;
        STATE_1:    out = 1;
        defualt:    out = 0;
    endcase
end

//------------------------------------------------------------------------------
// 三段式状态机
// 将状态转移条件、状态转换、输出分开在三个块中
// 最规范，且均为Moore输出
reg state;
reg next_state;

always @(*) begin
    // 状态转移条件
    case (state)
        STATE_IDLE: next_state = start ? STATE_1 : STATE_IDLE;
        STATE_1:    next_state = STATE_IDLE;
        default:    next_state = STATE_IDLE;
    endcase
end
    
always @(posedge clk) begin
    // 状态转换
    state <= next_state;
end

always @(posedge clk) begin
    // 输出
    case (next_state)   // 注意使用next_state做判断，否则会多一个周期的延迟
        STATE_IDLE: out <= 0;
        STATE_1:    out <= 1;
        default:    out <= 0;
    endcase
end
```



其他技巧

```verilog
//------------------------------------------------------------------------------
// 同步复位与异步复位

// 对于一段式状态机，区别在于复位信号是否列在敏感列表中
always @(posedge clk) begin                 // 同步复位
    if (rst) state <= STATE_IDLE;
    else ; // 状态机代码
end

always @(posedge clk or posedge rst) begin  // 异步复位
    if (rst) state <= STATE_IDLE;
    else ; // 状态机代码
end

// 对于多段式状态机，同步复位写在组合逻辑，异步复位写在时序逻辑
always @(*) begin                           // 同步复位
    if (rst) next_state = STATE_IDLE;
    else ; // 状态转换条件代码
end

always @(posedge clk or posedge rst) begin  // 异步复位
    if (rst) state <= STATE_IDLE;
    else ; // 状态转换代码
end

//------------------------------------------------------------------------------
// 快速的状态机：将状态与输出联系起来
module fast_fsm(
    input clk,
    input rst_n,
    input A,
    output [1:0] K
);
    reg [4:0] state;
    assign K = state[4:3];
    
    // 然后是状态机
endmodule
```

