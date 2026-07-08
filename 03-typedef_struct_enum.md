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
    print_byte_array(resp); // 打印数组内容
    
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
