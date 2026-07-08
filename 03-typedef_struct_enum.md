# 03 typedef_struct_enum
## 1.typedef
自定义数据类型
```systemverilog
typedef bit [1:0]       u2;    //定义u2为2位无符号数据类型
typedef bit [3:0]       u4;    //定义u4为4位无符号数据类型
typedef bit [7:0]       u8;    //定义u8为8位无符号数据类型
typedef bit [31:0]      u32;
typedef bit signed [1:0]  s2;    //定义s2为2位有符号数据类型
typedef bit signed [31:0] s32;

u2      my_u2;    //这里u2相当于int byte bit这种数据类型，my_u2是变量
u8      my_u8;
s32     my_s32;
```

## 2.struct
struct（结构体）就是把多个不同类型的数据打包成一个整体。就像你有一个"快递包裹"，里面可以同时装衣服、鞋子、书本等不同东西。
在这个例子中，trans_t 就是一个包裹，里面装了：
  一个命令 cmd
  两个操作数 op0、op1
  一个动态字节数组 resp（像一张可变长度的纸条）
  一个杂项 misc

```systemverilog
typedef struct {
    u8      cmd;        // 8位无符号：命令
    s32     op0, op1;   // 32位有符号：两个操作数
    byte    resp[$];    // 动态字节数组（队列）
    u8      misc;       // 8位无符号：杂项
} trans_t;    //给这个结构体起个别名 trans_t

// 声明变量
trans_t     tr0, tr1;   // 两个 trans_t 类型的结构体变量
byte        resp[];     // 定义一个动态数组（用于接收结构体中的数组）

initial begin
    #600;
    $display("---- struct test ----");
    
    // ====== 方式一：按"变量位置"赋值 ======依次对应：cmd=2, op0=-1, op1=2, resp={0,4,8}, misc=0
    tr0 = '{2, -1, 2, '{0,4,8}, 0}; 
    
    $display("cmd=%0d, op0=%0d, op1=%0d, misc=%0d.", 
             tr0.cmd, tr0.op0, tr0.op1, tr0.misc);    //tr0.cmd  访问 tr0 的 cmd 成员/  tr0.resp 访问 tr0 的 resp 数组/  tr1.op0 访问 tr1 的 op0 成员
    
    resp = tr0.resp;        // 把结构体中的队列赋值给外部数组
    print_byte_array(resp); // 打印数组内容  print_byte_array()是用户自定义函数，未写出来
    
    /*====== 方式二：按"数据类型+变量名"赋值 ======
     default:88 表示所有未指定成员的默认值是 88
     u8:6 表示把 u8 类型的字段（cmd, misc）赋值为6
     resp:{1,2,3,4,5,6} 给动态数组赋值*/
    tr1 = '{default:88, u8:6, resp:{1,2,3,4,5,6}};
    
    $display("cmd=%0d, op0=%0d, op1=%0d, misc=%0d.", 
             tr1.cmd, tr1.op0, tr1.op1, tr1.misc);
    
    print_byte_array(tr1.resp);
end
```

## 3.enum
枚举 = 给"数字"起个"名字"，让代码更易读、更安全。
比如：一周有7天，用数字 0~6 表示很抽象，用 mon, tue, wed... 就很直观。
```systemverilog
typedef enum {mon, tue=2, wed, thu, fri, sat=9, sun} week;
```
|   符号  |   数值   | 说明         |
| :---: | :----: | :--------- |
| `mon` |  **0** | 第一个，默认从0开始 |
| `tue` |  **2** | 显式指定为2     |
| `wed` |  **3** | 上一个+1      |
| `thu` |  **4** | 上一个+1      |
| `fri` |  **5** | 上一个+1      |
| `sat` |  **9** | 显式指定为9     |
| `sun` | **10** | 上一个+1      |
显式指定就按指定的来，没指定的 = 上一个值 + 1
```systemverilog
typedef enum {mon, tue=2, wed, thu, fri, sat=9, sun} week;  //定义枚举类型 week

// 声明枚举变量和数组
week    day_a[], day;   // day_a[] 是动态数组，day 是单个变量

// 测试代码
initial begin
    #700;
    
    // 初始化动态数组
    day_a = '{mon, wed, sun};
    
    // 遍历数组，打印每个元素的名称和数值
    foreach(day_a[i])
        $write("%s: %0d, ", day_a[i].name(), day_a[i]);
    $display("");           // 换行
    
    // day = 11;            // 语法错误！不能直接赋整数值
    // 正确做法：day = week'(11);  // 强制类型转换后，语法通过 但是week中没有值为11的，所以输出为空字符串和11
    
    day = sun;              // 直接赋枚举符号
    
    day = week'(day+1);     // 强制类型转换后赋值，day=11 (sun=10, +1=11)
    
    $display("%s: %0d", day.name(), day);  //output： ：11
    // 输出空字符串和11，因为11不是合法的枚举值
    
    $cast(day, 11);         // $cast动态检查值是否合法 运行时错误！11不是week的合法枚举值
end

```

## 4.union
union（联合）= 多个成员共享同一块内存空间
就像一间单人宿舍，里面可以住不同的人，但同一时间只能住一个。换人了，原来的东西就被覆盖了。
|          | `struct`（结构体）   | `union`（联合）       |
| :------- | :-------------- | :---------------- |
| **内存布局** | 每个成员**各自有独立空间** | 所有成员**共享同一块空间**   |
| **总大小**  | 所有成员大小**之和**    | 最大成员的大小           |
| **用途**   | 打包不同类型的数据       | **同一数据的不同解读方式**   |
| **同时访问** | 可以同时访问所有成员      | 同时访问所有成员会**互相干扰** |


```systemverilog
// 定义联合类型 op_u
typedef union {
    u32     u_i;        // 32位无符号整数= 4字节
    s32     s_i;        // 32位有符号整数= 4字节
    u8      u_char;     // 8位无符号整数= 1字节
} op_u;                //内存占用 = 最大成员 = 4字节（32位）

// 声明联合变量
op_u    op0;

// 测试代码
initial begin
    #800;
    
    // 通过 s_i 成员写入 -1
    op0.s_i = -1;
    
    // 用 %0d（十进制）打印三个成员的值
    $display("u_i:%0d, s_i:%0d, u_char:%0d.", op0.u_i, op0.s_i, op0.u_char);
    
    // 用 %0h（十六进制）打印三个成员的值
    $display("u_i:%0h, s_i:%0h, u_char:%0h.", op0.u_i, op0.s_i, op0.u_char);
end

output:
u_i:4294967295, s_i:-1, u_char:255.
u_i:ffffffff, s_i:ffffffff, u_char:ff.
```
### 写入 op0.s_i = -1后
内存中的32位二进制：1111_1111_1111_1111_1111_1111_1111_1111
                    = 0xFFFFFFFF
用不同成员"解读"同一块内存
### 十进制输出：
| 成员               | 解读方式              | 结果             |
| :--------------- | :---------------- | :------------- |
| `u_i`（u32 无符号）   | 32位全1 = 无符号数      | **4294967295** |
| `s_i`（s32 有符号）   | 32位全1 = 补码负数      | **-1**         |
| `u_char`（u8 无符号） | 只取低8位 `1111_1111` | **255**        |

### 十六进制输出验证：
u_i:ffffffff, s_i:ffffffff, u_char:ff.
| 成员       |    十六进制    | 说明               |
| :------- | :--------: | :--------------- |
| `u_i`    | `ffffffff` | 32位全1            |
| `s_i`    | `ffffffff` | 32位全1（有符号补码表示-1） |
| `u_char` |    `ff`    | 8位全1 = 255       |

## 5. packed struct/union
| 特性            | `unpacked`（默认）               | `packed`              |
| :------------ | :--------------------------- | :-------------------- |
| **内存布局**      | 成员按**对齐方式**存放，可能有间隙（padding） | 成员**紧密排列**，无间隙，像一个大向量 |
| **能否整体赋值/比较** | ❌ 不能                         | ✅ 可以                  |
| **能否作为向量操作**  | ❌ 不能                         | ✅ 可以（切片、位运算等）         |
| **成员类型限制**    | 无限制（可用动态数组、队列等）              | **必须是定宽类型**           |
| **综合友好性**     | 一般                           | 更好（布局确定，可综合）          |
### packed struct
```systemverilog
// 定义 packed struct（打包结构体）
typedef struct packed {
    u8          cmd;              // 8位
    s32         op0, op1;         // 两个32位有符号整数
    // byte     resp[$];          // ❌ 语法错误！packed 中不能用动态数组
    logic [255:0][7:0]  resp;     // ✅ 用定宽数组代替，256字节
    u8          misc;             // 8位
} trans_p_t;    //总位宽 = 8 + 32 + 32 + 256 x 8 + 8 = 2128 位
```
采用packed struct的好处：
```systemverilog
trans_p_t tr;

// ✅ 可以整体赋值（像操作一个向量）
tr = 2128'h0000_0000_..._0000_0001;

// ✅ 可以整体比较
if (tr == 0) begin ... end

// ✅ 可以切片取位
u8 cmd_bits = tr[7:0];          // 取最低8位（cmd）
u32 op0_bits = tr[39:8];        // 取op0
```
### packed union
```systemverilog
// 定义 packed union（打包联合）
typedef union packed {
    u32         u_i;              // 32位无符号
    s32         s_i;              // 32位有符号
    u8          u_char;           // 8位
} op_p_u;
```
### enum
enum 本身就是"打包"的！ 它内部就是整数，没有 packed/unpacked 之分。
enum 的值本质上就是整数（默认 int）
它天然就是紧密排列的
所以 SystemVerilog 语法不允许给 enum 加 packed 关键字
```systemverilog
// ❌ 语法错误示例
// typedef enum packed {mon, tue=2, wed, thu, fri, sat=9, sun} week_p;
// enum 不支持 packed 修饰！
```
| 场景          | 用 `packed` 的理由        |
| :---------- | :-------------------- |
| **硬件建模**    | 需要精确控制位宽和布局，方便综合      |
| **协议解析**    | 把数据包映射成结构体，按位切片       |
| **整体赋值/比较** | 需要像向量一样操作整个结构体        |
| **与C语言交互**  | C结构体通常是 packed 的，需要对齐 |

## 6.string
s.len() → 获取长度
s.getc(index) 按索引取字符，索引从 0 开始
s.putc(index, char) 将指定索引的字符替换
s={} 是 SV 的拼接运算符
s.substr(start, end) 提取从 start 到 end 的子串（包含两端）
$sformat(s, "%s: score %0d", "sky", 60); → 格式化（存到s变量）
$psprintf("%s: score %0d", "you", 100); → 格式化（不会存到s变量）相当于print
```systemverilog
string s;           // 声明一个 string 类型的变量 s

initial begin
    #900;           // 延迟 900 个时间单位
    
    $display("---- string test ----");
    
    s = "Hell SV ";                 // 给字符串赋值（注意末尾有个空格）
    
    $display("len of s is: %0d.", s.len());           // 获取字符串长度
    $display("last char is: %c.", s.getc(s.len()-1));  // 获取最后一个字符
    s.putc(s.len()-1, ",");        // 将最后一个字符替换为 ","
    
    s = {s, " give me money!"};     // 字符串拼接
    
    $display(s);                    // 打印当前字符串
    $display("%s", s.substr(5,6));  // 提取子串，从索引5到索引6
    
    $sformat(s, "%s: score %0d", "sky", 60);   // 格式化字符串，结果存入 s
    $display(s);
    
    s = $psprintf("%s: score %0d", "you", 100); // 格式化字符串，返回新字符串
    $display(s);

end
```
| 代码行                                                 | 输出                            |
| --------------------------------------------------- | ----------------------------- |
| `$display("len of s is: %0d.", s.len());`           | `len of s is: 8.`             |
| `$display("last char is: %c.", s.getc(s.len()-1));` | `last char is: .`（空格，所以看起来空白） |
| `$display(s);`（拼接后）                                 | `Hell SV, give me money!`     |
| `$display("%s", s.substr(5,6));`                    | `SV`                          |
| `$display(s);`（\$sformat 后）                         | `sky: score 60`               |
| `$display(s);`（\$psprintf 后）                        | `you: score 100`              |

