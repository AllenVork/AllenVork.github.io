---
layout:     post
title:      JVM-class文件解析
subtitle:   通过简单 demo 来解析 class 文件
header-img: img/android3.png
author:     Allen Vork
catalog: true
tags:
    - jvm
---

## 1. Class 文件基本结构

### 1.1 常量池结构

```java
ClassFile {
  u4              magic;
  u2              minor_version;
  u2              major_version;
  u2              constant_pool_count;
  cp_info         constant_pool[constant_pool_count-1];
  u2              access_flags;
  u2              this_class;
  u2              super_class;
  u2              interfaces_count;
  u2              interfaces[interfaces_count];
  u2              fields_count;
  field_info      fields[fields_count];
  u2              methods_count;
  method_info     methods[methods_count];
  u2              attributes_count;
  attribute_info  attributes[attributes_count];
}
```

u1, u2, u4, u8 表示这个字段占用多少个字节。譬如 magic 为 u4 即魔数占用4个字节。16进制中，两个字符是一个字节（AA 占一个字节，不过 A 也是占一个字节）。

+ 魔数：表示该文件是可以被虚拟机加载的 class 文件。值是固定的 0xCAFEBABE（ps：创始人很喜欢咖啡）。
+ 次版本号
+ 主版本号：JDK的版本号。52代表 jdk 的版本号为1.8。高版本的 jdk 向下兼容低版本的 class 文件，但低版本的 jdk 不能运行高版本的 class 文件。
+ 常量池计数器：代表有多少个常量，譬如说值为 22，那么实际有 22-1=21个常量，因为常量池的下标是从1开始的，目的是当不引用任何一个常量池项时，用0表示。
+ 常量池：存放各种数据类型，可以看作是数组或者集合。既然是数组或者集合，就要确定它的长度，那么就是上面的常量池计数器。    
+ 访问标志：表示类和接口的访问权限和属性，如是否为 public，final，abstract、enum 等。
+ this_class && super_class && interfaces_count && interfaces[]：这几组数据共同确定类的继承关系。this_class 代表类的索引，用于确定该类的权限定名。super_class 指父类的索引，interfaces_count 指接口的数量，接口信息存储在 interfaces[] 中。
+ fields_count & field_info：字段表集合，表示该类中声明的变量。变量指的是成员变量，不包含方法中的局部变量。
+ methods_count && method_info：方法表集合，表示类中的方法。
+ attributes_count && attribute_info：属性表。前面的方法表，字段表都包含了属性表。属性表的种类很多，可以表示源文件名称、编译生成的字节码指令、final 定义的常量、方法抛出的异常等。

### 1.2 常量池数据类型

| 类 型   |      标 志      |  描 述 |
|----------|:-------------:|------:|
| CONSTANT_Utf8_info |  1  |  UTF-8 编码的字符串  |
| CONSTANT_Integer_info |  3  |  整型字面量  |
| CONSTANT_Float_info |  4  |  浮点型字面量  |
| CONSTANT_Long_info |  5  |  长整型字面量  |
| CONSTANT_Double_info |  6  |  双精度浮点型字面量  |
| CONSTANT_Class_info |  7  |  类或接口的符号引用  |
| CONSTANT_String_info |  8  |  字符串类型字面量  |
| CONSTANT_Fieldref_info |  9  |  字段的符号引用  |
| CONSTANT_Methodref_info | 10 |  类中方法的符号引用  |
| CONSTANT_InterfaceMethodref_info |  11  |  接口中方法的符号引用  |
| CONSTANT_NameAndType_infos |  12  |  字段或方法的部分符号引用  |
| CONSTANT_MethodHandle_info |  15  |  表示方法句柄  |
| CONSTANT_MethodType_info |  16  |  标识方法类型  |
| CONSTANT_InvokeDynamic_info |  18  |  表示一个动态方法调用点  |

常量池的数据类型有十几种，各自有自己的数据结构，但他们都有一个共有属性 `tag`。tag 是一个标志位，标志是哪一种数据结构。

### 1.3 访问标志

|   标志名称   |      标 志 值      |  含 义  |
|----------|:-------------:|------:|
| ACC_PUBIC |  0x0001  |  是否为 public 类型  |
| ACC_PRIVATE |  0x0002  |  是否为 private 类型  |
| ACC_PROTECTED |  0x0004  |  是否为 protected 类型  |
| ACC_STATIC |  0x0008  |  是否为 static 类型  |
| ACC_FINAL |  0x0010  |  是否声明为 final  |
| ACC_SUPER |  0x0020  |  JDK1.0.2 之后编译出来的类这个标志都必须为真  |
| ACC_VOLATILE |  0x0040  |  是否为 volatile  |
| ACC_INTERFACE |  0x0200  |  是否为接口  |
| ACC_ABSTRACT |  0x0400  |  是否为 abstract 类型  |
| ACC_SYNTHETIC |  0x1000  |  标记这个类并非由用户代码产生  |
| ACC_ANNOTATION |  0x2000  |  是否为注解  |
| ACC_ENUM |  0x4000  |  是否为枚举类型  |

## 2. 通过简单例子来解析 class 文件

我们采用一个最简单的类来分析 class 文件是如何识别内容的：  

```java
package com.example.classtest;

public class FieldClass {
    private final int i = 1;
}
```

采用 `xxd FieldClass.class myFile.txt` 将编译出来的 class 文件转换为16进制文件：

```java
00000000: cafe babe 0000 0033 0016 0a00 0400 1209  .......3........
00000010: 0003 0013 0700 1407 0015 0100 0169 0100  .............i..
00000020: 0149 0100 0d43 6f6e 7374 616e 7456 616c  .I...ConstantVal
00000030: 7565 0300 0000 0101 0006 3c69 6e69 743e  ue........<init>
00000040: 0100 0328 2956 0100 0443 6f64 6501 000f  ...()V...Code...
00000050: 4c69 6e65 4e75 6d62 6572 5461 626c 6501  LineNumberTable.
00000060: 0012 4c6f 6361 6c56 6172 6961 626c 6554  ..LocalVariableT
00000070: 6162 6c65 0100 0474 6869 7301 0022 4c63  able...this.."Lc
00000080: 6f6d 2f65 7861 6d70 6c65 2f63 6c61 7373  om/example/class
00000090: 7465 7374 2f46 6965 6c64 436c 6173 733b  test/FieldClass;
000000a0: 0100 0a53 6f75 7263 6546 696c 6501 000f  ...SourceFile...
000000b0: 4669 656c 6443 6c61 7373 2e6a 6176 610c  FieldClass.java.
000000c0: 0009 000a 0c00 0500 0601 0020 636f 6d2f  ........... com/
000000d0: 6578 616d 706c 652f 636c 6173 7374 6573  example/classtes
000000e0: 742f 4669 656c 6443 6c61 7373 0100 106a  t/FieldClass...j
000000f0: 6176 612f 6c61 6e67 2f4f 626a 6563 7400  ava/lang/Object.
00000100: 2100 0300 0400 0000 0100 1200 0500 0600  !...............
00000110: 0100 0700 0000 0200 0800 0100 0100 0900  ................
00000120: 0a00 0100 0b00 0000 3800 0200 0100 0000  ........8.......
00000130: 0a2a b700 012a 04b5 0002 b100 0000 0200  .*...*..........
00000140: 0c00 0000 0a00 0200 0000 0300 0400 0400  ................
00000150: 0d00 0000 0c00 0100 0000 0a00 0e00 0f00  ................
00000160: 0000 0100 1000 0000 0200 11              ...........
```

通过1.1中的 class 文件的结构可知：

+ 魔数：cafe baby。
+ 次版本号：0000
+ 主版本号：0033，即51。jdk 1.7
+ 常量池计数器：0016，即 22-1=21个常量
+ 常量池：存放各种数据类型，可以看作是数组或者集合。既然是数组或者集合，就要确定它的长度，那么就是上面的常量池计数器。

-----------------

### 2.1 常量池

常量池的结构基本为：

```java
cp_info {
    u1 tag;
    u1 info[];
}
```

#### 第1个常量

那么首先看第一个 u1 的 tag 为 0a。即为10。通过查表可知，这个结构体为:

```java
CONSTANT_Methodref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}
```

那么这是一个方法的符号引用。后面有两个 u2 类型的索引：  

+ class_index：表示定义该字段的类或接口在常量池中的索引
+ name_and_type_index：类/接口的**字段名和字段描述符**在常量池中的索引

那么 class_index:00 04，即为常量池4号位置，name_and_type_index：00 12，即为常量池18号位置。由于常量池还没解析完，还不知道这个索引对应的值是多少，所以我们继续往后看。

### 第2个常量

type：09，那么结构体为和 method_info 一样：

```java
CONSTANT_Fieldref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}
```

+ class_index：0003，即在常量池中的位置为3
+ name_and_type_index：0013，即在常量池中的位置为19

### 第3个常量

tag：07，那么结构体为：

```java
CONSTANT_Class_info {
    u1 tag;
    u2 name_index; // 常量池中该索引处的结构一定是 CONSTANT_Utf8_info，代表类/接口名
}
```

它用于表示一个类或接口。  

+ name_index：0014，即常量池中 20 的位置是一个 CONSTANT_Utf8_info 的结构体，代表类/接口的名字。    

### 第4个常量

tag：07，那么和上面一样：

+ name_index：0015，即常量池中 21 的位置是一个 CONSTANT_Utf8_info 的结构体，代表类/接口的名字。  

### 第5个常量

tag：01，那么结构体为：  

```java
CONSTANT_Utf8_info {
    u1 tag;
    u2 length; // utf-8 编码的字符占用的字节数
    u1 bytes[length]; // 字符串
}
```

+ length：00 01，即长度为1
+ bytes[1]：69，使用[在线转换工具](http://www.bejson.com/convert/ox2str/) 将起转换为字符串即为 i

### 第6个常量

tag：01，那么和还是上面相同：

+ length：00 01，即长度为1
+ bytes[1]：49，即为 I

### 第7个常量

tag：01，那么和还是上面相同：

+ length：00 0d，即长度为13
+ bytes[13]：43 6f6e 7374 616e 7456 616c 7565，即为 ConstantValue

### 第8个常量

tag：03，结构体为：  

```java
CONSTANT_Integer_info {
    u1 tag;
    u4 bytes;
}
```

即为一个整型：  

+ bytes：00 0000 01，值为 1。

### 第9个常量

tag：01，那么和还是上面相同：

+ length：00 06，即长度为6
+ bytes[6]：3c69 6e69 743e，即为 `<init>`

### 第10个常量

tag：01，那么和还是上面相同：

+ length：00 03，即长度为3
+ bytes[3]：28 2956，即为 ()V

### 第11个常量

tag：01，那么和还是上面相同：

+ length：00 04，即长度为4
+ bytes[4]：43 6f64 65，即为 Code

### 第12个常量

tag：01，那么和还是上面相同：

+ length：000f，即长度为15
+ bytes[15]：4c69 6e65 4e75 6d62 6572 5461 626c 65，即为 LineNumberTable

### 第13个常量

tag：01，那么和还是上面相同：

+ length：0012，即长度为18
+ bytes[17]：4c6f 6361 6c56 6172 6961 626c 6554 6162 6c65，即为 LocalVariableTable

### 第14个常量

tag：01，那么和还是上面相同：

+ length：00 04，即长度为4
+ bytes[4]：74 6869 73，即为 this

### 第15个常量

tag：01，那么和还是上面相同：

+ length：0022，即长度为34
+ bytes[34]：4c63 6f6d 2f65 7861 6d70 6c65 2f63 6c61 7373 7465 7374 2f46 6965 6c64 436c 6173 733b，即为 Lcom/example/classtest/FieldClass

### 第16个常量

tag：01，那么和还是上面相同：

+ length：00 0a，即长度为10
+ bytes[10]：53 6f75 7263 6546 696c 65，即为 SourceFile

### 第17个常量

tag：01，那么和还是上面相同：

+ length：000f，即长度为15
+ bytes[15]：4669 656c 6443 6c61 7373 2e6a 6176 61，即为 FieldClass.java

### 第18个常量

tag：0c，那么结构体为： 

```java
// 用于表示一个字段或方法
CONSTANT_NameAndType_info {
    u1 tag;
    // 该 index 指向常量池中的 CONSTANT_Utf8_info 结构，表示特殊的方法名 <init> 或者
    // 方法，字段，局部变量的非限定名(unqualified name）
    u2 name_index;

    // 该 index 指向常量池中的 CONSTANT_Utf8_info 结构，表示字段/方法的类型
    u2 descriptor_index;
}
```

+ name_index：0009，常量池 #9 位置，前面已经解析出来了为 `<init>`
+ descriptor_index：000a，常量池 #10 位置，为 ()V，即返回值类型为 V

解释下限定名和非限定名： 

+ 限定名：即为全名，带包路径的用点隔开，例如: java.lang.String
+ 非限定名：短名，不带包的，即 String

### 第19个常量

tag：0c，那么和上面一样：

+ name_index：00 05，那么值为常量池中 #5 位置，为1
+ descriptor_index：00 06，那么值在常量池 #6 位置，为 I

### 第20个常量

tag：01，那么和还是上面相同：

+ length：0020，即长度为32
+ bytes[32]：636f 6d2f 6578 616d 706c 652f 636c 6173 7374 6573 742f 4669 656c 6443 6c61 7373，即为 com/example/classtest/FieldClass

### 第21个常量

tag：01，那么和还是上面相同：

+ length：0010，即长度为16
+ bytes[16]：6a 6176 612f 6c61 6e67 2f4f 626a 6563 74，即为 java/lang/Object

### 2.1.1 常量池总结

到这里常量池就解析完了，可以看出它的结构为：  

```cmd
Constant pool:
   #1 = Methodref          #4.#18         // java/lang/Object."<init>":()V
   #2 = Fieldref           #3.#19         // com/example/classtest/FieldClass.i:I
   #3 = Class              #20            // com/example/classtest/FieldClass
   #4 = Class              #21            // java/lang/Object
   #5 = Utf8               i
   #6 = Utf8               I
   #7 = Utf8               ConstantValue
   #8 = Integer            1
   #9 = Utf8               <init>
  #10 = Utf8               ()V
  #11 = Utf8               Code
  #12 = Utf8               LineNumberTable
  #13 = Utf8               LocalVariableTable
  #14 = Utf8               this
  #15 = Utf8               Lcom/example/classtest/FieldClass;
  #16 = Utf8               SourceFile
  #17 = Utf8               FieldClass.java
  #18 = NameAndType        #9:#10         // "<init>":()V
  #19 = NameAndType        #5:#6          // i:I
  #20 = Utf8               com/example/classtest/FieldClass
  #21 = Utf8               java/lang/Object
```

-----------------

+ 访问标志：0021，即33，[查表](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.1-200-E.1) 可知为1+32，所以类的访问权限和属性为 public。
+ this_class：00 03，即为常量池3号位置 com/example/classtest/FieldClass
+ super_class：00 04，即为常量池4号位置 java/lang/Object
+ interfaces_count：00 00，即没有实现接口
+ interfaces[]：无

-----------------

### 2.2 filed

+ fields_count：00 01，即1个字段
+ field_info：字段表集合，表示该类中声明的变量。变量指的是成员变量，不包含方法中的局部变量。

```java
field_info {
    u2             access_flags; //表示访问权限和属性，通过值去查表可得
    u2             name_index;
    u2             descriptor_index;
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

+ access_flags：00 12，即18。[查表](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.5-200-A.1) 可知为2+16，即 private final 类型。
+ name_index：00 05，即常量池5号位置，为 i
+ descriptor_index：00 06，即常量池6号位置，为 I
+ attributes_count：00 01。表示有1个额外属性。一个方法可以有任意个数的属性
+ attribute_info：属性信息  

```java
attribute_info {
    u2 attribute_name_index; // 属性名称在常量池中的索引。不同的 attribute 都是通过它来区分
    u4 attribute_length; // 属性长度
    u1 info[attribute_length]; // 属性值
}
```

+ attribute_name_index：00 07，即该 attribute 类型为 ConstantValue：

```java
ConstantValue_attribute {  
   u2 attribute_name_index;  // 常量池中该位置的值为 ConstantValue
   u4 attribute_length;  // 定长，固定为 2
   u2 constantvalue_index;  // 常量池的索引
}
```

+ attribute_length：00 0000 02，即长度确实为2。
+ constantvalue_index：00 08，即常量池8号位置，为1

可以看出这个结构体就说明了有一个 field 为 private final int i = 1

> ConstantValue 是定长属性，只会出现在 field_info 中，用于表示**常量表达式的值**。该属性仅限于基本类型和 String，因为从常量池中之能饮用到基本类型和 String 的字面量。

-----------------

### 2.3 method

+ methods_count：00 01，即有一个方法
+ method_info：方法表集合，表示类中的方法。

method info 结构体如下，类似 filed_info：

```java
method_info {
    u2             access_flags;
    u2             name_index;
    u2             descriptor_index;
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

+ access_flags：00 01，即1。[查表](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.6-200-A.1) 可知为 public 类型。
+ name_index：00 09，即常量池9号位置，为 `<init>`
+ descriptor_index：00 0a，即常量池10号位置，为 ()V
+ attributes_count：00 01，表示有一个额外属性
+ attribute_info：

```java
attribute_info {
    u2 attribute_name_index; // 属性名称在常量池中的索引。不同的 attribute 都是通过它来区分
    u4 attribute_length; // 属性长度
    u1 info[attribute_length]; // 属性值
}
```

+ attribute_name_index：00 0b，即常量池11位置，为 Code，即该 attribute 属性的类型为 Code：  

```java
Code_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 max_stack;
    u2 max_locals;
    u4 code_length;
    u1 code[code_length];
    u2 exception_table_length;
    {   u2 start_pc;
        u2 end_pc;
        u2 handler_pc;
        u2 catch_type;
    } exception_table[exception_table_length];
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}
```

-----------------

+ attributes_count && attribute_info：属性表。前面的方法表，字段表都包含了属性表。属性表的种类很多，可以表示源文件名称、编译生成的字节码指令、final 定义的常量、方法抛出的异常等。

<!-- 
```java
00 0000 3800 0200 0100 0000  ........8.......
00000130: 0a2a b700 012a 04b5 0002 b100 0000 0200  .*...*..........
00000140: 0c00 0000 0a00 0200 0000 0300 0400 0400  ................
00000150: 0d00 0000 0c00 0100 0000 0a00 0e00 0f00  ................
00000160: 0000 0100 1000 0000 0200 11              ...........
该部分后面补充
```
-->

## References

+ [The class File Format](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.3.2)
+ [Class 文件格式详解](https://juejin.im/post/5c0932cee51d45090a1da07e)
+ [常量池中的11种数据类型的结构总表](https://blog.csdn.net/qq_39375211/article/details/79925127)
+ [javap Usage Unfolds](https://blog.overops.com/javap-usage-unfolds-whats-hidden-inside-your-java-class-files/)
+ [深入Java虚拟机之二：Class类文件结构](https://blog.csdn.net/ns_code/article/details/17675609)