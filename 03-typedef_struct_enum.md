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
