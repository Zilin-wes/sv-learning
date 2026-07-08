# 04 fuction_task_program
## 1.fuction & task
| 特性        | function              | task                        |
| --------- | --------------------- | --------------------------- |
| **仿真时间**  | 不消耗任何仿真时间（0时间执行）      | 可以消耗仿真时间                    |
| **可调用对象** | 只能调用 function         | 可以调用 task 和 function        |
| **返回值**   | 必须有返回值（默认返回最后一个表达式的值） | 无返回值（用 output/inout 参数传递结果） |
### 1.1规范形式
function模板
```systemverilog
function [automatic] [返回类型] [函数名] (
    [方向] [数据类型] [参数名],
    [方向] [数据类型] [参数名] = [默认值]    // 可选
);
    // ========== 内部变量声明区 ==========
    [数据类型] [变量名];

    // ========== 执行语句区 ==========
    [语句];
    
    // ========== 返回语句（void 可省略）==========
    return [返回值];    // 有返回值时必须写
    // 或
    return;             // void function 提前退出时用

endfunction
```
task模板
```systemverilog
task [automatic] [任务名] (
    [方向] [数据类型] [参数名],
    [方向] [数据类型] [参数名] = [默认值]    // 可选
);
    // ========== 内部变量声明区 ==========
    [数据类型] [变量名];

    // ========== 执行语句区 ==========
    [语句];             // 可以包含 #延时、@事件等
    
    // ========== 提前退出（可选）==========
    return;             // 提前结束 task，不能写返回值

endtask
```
### 1.2 二者return区别
#### ✅ Function：可以 return 一个值
```systemverilog
function int add(input int a, b);
    if (a < 0 || b < 0)
        return -1;          // ✅ 返回一个具体的 int 值
    
    return a + b;           // ✅ 返回计算结果
endfunction

int result;
result = add(3, 5);         // result = 8
result = add(-1, 5);        // result = -1
```

#### ✅ Task：只能用 return; 提前结束，不能返回值
```systemverilog
task check_positive(input int a);
    if (a <= 0) begin
        $display("错误：必须是正数");
        return;             // ✅ 提前退出task，后面代码不执行
    end
    
    $display("OK：%0d 是正数", a);
    // ... 继续其他操作
endtask

check_positive(5);          // 打印 "OK：5 是正数"
check_positive(-3);         // 打印 "错误：必须是正数"，然后提前return
```

#### ❌ Task 不能写 return() 尝试返回值
```systemverilog
task bad_task(input int a);
    // ❌ 错误！task 不能返回值
    return a + 1;           // 编译报错！
    
    // ❌ 错误！task 不能写 return()
    return (a);             // 编译报错！
    
    // ✅ 正确！只能写 return;
    return;
endtask
```
#### Task 的本质就是"做事"，不是"算值"。

### 1.3调用区别
function 不能调用 task，即使那个 task 内部没有显式的时间延迟。因为 task 的语义允许消耗时间，编译器不允许 function 调用 task。
function 和 task 都可以有多个参数，方向类型包括：
input — 输入
output — 输出
inout — 双向
ref — 引用（类似 C 语言指针）
默认规则：如果不指定方向，默认是 input logic。
| 方向         | 行为                             |
| ---------- | ------------------------------ |
| **input**  | 进入时，将外部实参的值**复制**给内部形参         |
| **output** | 退出时，将内部形参的值**复制**给外部实参         |
| **ref**    | 传递的是**引用/地址**，直接操作外部变量，类似 C 指针 |

ref 的好处是避免大数据量的复制开销，比如传递一个很大的数组时。

#### function 用于纯组合逻辑/计算（不耗时），task 用于带时序的验证行为（如激励驱动、等待响应）。
```systemverilog
byte b[$];

// 示例1: input 参数 — 值传递，修改的是形参副本
// ============================================
function void array_add4(input byte a[$]);
    foreach(a[i])
        a[i] = a[i] + 4;
endfunction

// 示例2: output 参数 — 结果通过 output 传出
// ============================================
function void array_add4_out(input byte a[$], output byte b[$]);
    foreach(a[i])
        b[i] = a[i] + 4;
endfunction

// 示例3: ref 参数 — 引用传递，直接修改原数组
// ============================================
function automatic void array_add4_ref(ref byte a[$]);
    foreach(a[i])
        a[i] = a[i] + 4;
endfunction

// 测试代码
initial begin
    #100;
    $display("----- function & task test -----");
    print_byte_array(a);
    
    // 测试 input: a 不会被修改（值传递）
    array_add4(a);
    print_byte_array(a);      // 1 2 4 8 16（不变）
    
    // 测试 output: 结果写入 b
    array_add4_out(a, b);
    print_byte_array(b);      // 5 6 8 12 20（a+4）
    
    // 测试 ref: a 直接被修改
    array_add4_ref(a);
    print_byte_array(a);      // 5 6 8 12 20（被修改了）
end
```


```systemverilog
task nothing;
    $display("call task: nothing.");
endtask

task automatic delay_10ns_add4(ref byte a[$]);
    #10;                    // task 可以包含时序控制
    array_add4_ref(a);      // task 可以调用其他 task
endtask

function void func_nest1();
    nothing;    // function 调用 task → 输出 warning
endfunction

// 下面这段被注释掉了，因为是语法错误：Function 不能调用带延时的 Task ❌
// function void func_nest2();
//     delay_10ns_add4(a);  // syntax error!
// endfunction

initial begin
    #200;
    nothing();              // 打印: call task: nothing.
    func_nest1();           // 打印: call task: nothing. (warning)
    delay_10ns_add4(a);     // 等待 10ns，打印并修改数组
    print_byte_array(a);    // 输出: 9 10 12 16 24
end

output
call task: nothing.
call task: nothing.
call task: delay_10ns_add4.
9 10 12 16 24
```
## 2.atumatic & static 

| 特性   | `static`（默认） | `automatic` |
| ---- | ------------ | ----------- |
| 存储空间 | 所有调用共享同一份    | 每次调用独立分配    |
| 内部变量 | 调用之间互相影响     | 调用之间互不影响    |
| 递归调用 | ❌ 有问题        | ✅ 安全        |
| 并发调用 | ❌ 会互相覆盖      | ✅ 各自独立      |

### 一句话：写 SV 代码时，养成习惯：所有 task/function 前面都加 automatic！
static容易发生竞争冒险导致结果不确定

static
```systemverilog
module test_static;

    // 默认是 static task
    task count;
        int i = 0;          // static: 只初始化一次，之后保持值
        i = i + 1;
        $display("i = %0d", i);
    endtask

    initial begin
        count();    // i = 1
        count();    // i = 2  ← 不是 1！因为共享同一个 i
        count();    // i = 3
    end

endmodule

```
atomatic
```systemverilog
module test_automatic;

    // 显式声明为 automatic
    task automatic count;
        int i = 0;          // automatic: 每次调用都重新初始化
        i = i + 1;
        $display("i = %0d", i);
    endtask

    initial begin
        count();    // i = 1
        count();    // i = 1  ← 每次都是新的 i
        count();    // i = 1
    end

endmodule
```

## 3.program
program 是 "纯软件" 的测试环境，和硬件描述（module）切割开。
在数字IC验证中：
    module = 描述硬件电路（DUT，被测设计）
    program = 写测试激励（TB，测试平台）

| 特性             | `module`         | `program`      |
| -------------- | ---------------- | -------------- |
| 描述对象           | 硬件（HW）           | 软件测试（SW）       |
| `always` 块     | ✅ 可以             | ❌ 不可以          |
| `initial` 块    | ✅ 可以             | ✅ 可以           |
| 嵌套 `module`    | ✅ 可以             | ❌ 不可以          |
| 嵌套 `interface` | ✅ 可以             | ❌ 不可以          |
| 执行完自动结束        | ❌ 需要手动 `$finish` | ✅ 自动 `$finish` |
| 执行顺序（0时刻）      | 先执行              | 后执行            |

```systemverilog
// ==================== 被测硬件（DUT）====================
module adder (
    input  logic [3:0] a, b,
    output logic [4:0] sum
);
    assign sum = a + b;     // 纯组合逻辑
endmodule

// ==================== 测试平台（TB）====================
program test_adder;         // program：纯软件测试环境

    logic [3:0] a, b;
    logic [4:0] sum;

    // 连接到DUT（通过interface或顶层连线）
    adder dut (.*);         // 实例化DUT

    initial begin
        $display("=== 开始测试 ===");
        
        a = 4'd3; b = 4'd5;
        #10;
        $display("%0d + %0d = %0d", a, b, sum);
        
        a = 4'd10; b = 4'd7;
        #10;
        $display("%0d + %0d = %0d", a, b, sum);
        
        $display("=== 测试结束 ===");
        // 不需要写 $finish()，自动结束！
    end

endprogram

// ==================== 顶层 ====================
module top;
    test_adder tb();        // 实例化program
endmodule


//output
=== 开始测试 ===
3 + 5 = 8
10 + 7 = 17
=== 测试结束 ===
```
顶层 = 测试板（PCB）
把芯片（DUT）放上去
把测试仪器（TB）接上去
提供电源、时钟等公共信号
没有顶层，DUT和TB就是"散件"，没法一起工作！
```systemverilog
// ==================== 1. 被测硬件（DUT）====================
module adder (
    input  logic [3:0] a, b,
    output logic [4:0] sum
);
    assign sum = a + b;
endmodule

// ==================== 2. 测试平台（TB）====================
program test_adder (
    output logic [3:0] a, b,    // 输出给DUT的激励
    input  logic [4:0] sum      // 从DUT读回的结果
);
    initial begin
        a = 4'd3; b = 4'd5;
        #10;
        $display("3+5=%0d", sum);
    end
endprogram

// ==================== 3. 顶层（Top）====================
// 顶层的作用：把DUT和TB"拼"在一起
module top;                     // ← 这就是顶层！

    // 1. 定义连线（相当于PCB上的铜线）
    logic [3:0] a_wire, b_wire;
    logic [4:0] sum_wire;

    // 2. 实例化DUT（把芯片焊到板上）
    adder dut (
        .a(a_wire),
        .b(b_wire),
        .sum(sum_wire)
    );

    // 3. 实例化测试程序（把测试仪器接到芯片上）
    test_adder tb (
        .a(a_wire),        // TB输出 → DUT输入
        .b(b_wire),        // TB输出 → DUT输入
        .sum(sum_wire)     // DUT输出 → TB输入
    );

    // 4. 还可以提供公共信号（时钟、复位等）
    logic clk = 0;
    always #5 clk = ~clk;   // 时钟发生器

endmodule
```

❌ 用 module 写测试（需要手动结束）
```systemverilog
module tb_module;           // 用module写测试

    initial begin
        $display("测试开始");
        #100;
        $display("测试结束");
        $finish;            // 必须手动调用，否则仿真一直跑！
    end

    // 如果忘记 $finish，仿真会一直运行，不会停止
endmodule
```
✅ 用 program 写测试（自动结束）
```systemverilog
program tb_program;         // 用program写测试

    initial begin
        $display("测试开始");
        #100;
        $display("测试结束");
        // 不需要 $finish！program 自动结束仿真
    end

endprogram
```
