---
title: 基础知识
categories:
    - 编程
        - 基础
tags:
    - Java
---

## 一、基本数据类型及其封装类型

Java基本类型共有八种，基本类型可以分为三类，字符类型char，布尔类型boolean以及数值类型byte、short、int、long、float、double。数值类型又可以分为整数类型byte、short、int、long和浮点数类型float、double。JAVA中的数值类型不存在无符号的，它们的取值范围是固定的，不会随着机器硬件环境或者[操作系统](http://lib.csdn.net/base/operatingsystem)的改变而改变。实际上，JAVA中还存在另外一种基本类型void，它也有对应的包装类 java.lang.Void，不过我们无法直接对它们进行操作。

<!-- more -->

8种类型表示范围如下：

| 简单类型 | 二进制位数 | 封装器类型 | 最大存储数据量 | 存储范围                        |
| -------- | ---------- | ---------- | -------------- | ------------------------------- |
| boolean  | 1          | Boolean    |                | true或者false                   |
| byte     | 8          | Byte       | 255            | -128~127                        |
| char     | 16         | Character  |                |                                 |
| short    | 16         | Short      | 65536          | -32768~32767                    |
| Int      | 32         | Integer    | 32次方减1      | 负的2的31次方到正的2的31次方减1 |
| long     | 64         | Long       | 2的64次方减1   | 负的2的63次方到正的2的63次方减1 |
| float    | 32         | Float      | ——             | 3.4e-45~1.4e38                  |
| double   | 64         | Double     | ——             | 4.9e-324~1.8e308                |
| void     | ——         | Void       |                |                                 |

Java决定了每种简单类型的大小。这些大小并不随着机器结构的变化而变化。这种大小的不可更改正是Java程序具有很强移植能力的原因之一。下表列出了Java中定义的简单类型、占用二进制位数及对应的封装器类。

对于数值类型的基本类型的取值范围，我们无需强制去记忆，因为它们的值都已经以常量的形式定义在对应的包装类中了。如：

基本类型byte 二进制位数：Byte.SIZE 最小值：Byte.MIN_VALUE 最大值：Byte.MAX_VALUE

基本类型short二进制位数：Short.SIZE 最小值：Short.MIN_VALUE 最大值：Short.MAX_VALUE

基本类型char二进制位数：Character.SIZE 最小值：Character.MIN_VALUE 最大值：Character.MAX_VALUE

基本类型double 二进制位数：Double.SIZE 最小值：Double.MIN_VALUE 最大值：Double.MAX_VALUE

**注意**：float、double两种类型的最小值与Float.MIN_VALUE、 Double.MIN_VALUE的值并不相同，实际上Float.MIN_VALUE和Double.MIN_VALUE分别指的是 float和double类型所能表示的最小正数。也就是说存在这样一种情况，0到±Float.MIN_VALUE之间的值float类型无法表示，0 到±Double.MIN_VALUE之间的值double类型无法表示。这并没有什么好奇怪的，因为这些范围内的数值超出了它们的精度范围。

Float和Double的最小值和最大值都是以科学记数法的形式输出的，结尾的"E+数字"表示E之前的数字要乘以10的多少倍。比如3.14E3就是3.14×1000=3140，3.14E-3就是3.14/1000=0.00314。

Java基本类型存储在栈中，因此它们的存取速度要快于存储在堆中的对应包装类的实例对象。从Java5.0（1.5）开始，JAVA虚拟机（Java Virtual Machine）可以完成基本类型和它们对应包装类之间的自动转换。因此我们在赋值、参数传递以及数学运算的时候像使用基本类型一样使用它们的包装类，但这并不意味着你可以通过基本类型调用它们的包装类才具有的方法。另外，所有基本类型（包括void）的包装类都使用了final修饰，因此我们无法继承它们扩展新的类，也无法重写它们的任何方法。

基本类型的优势：数据存储相对简单，运算效率比较高

包装类型的优势：有的地方必须使用包装器类型，比如集合的元素必须是对象类型，满足了java一切皆是对象的思想



## 二、自动装箱与拆箱

简单一点说，装箱就是自动将基本数据类型转换为包装器类型；拆箱就是自动将包装器类型转换为基本数据类型。



## 三、运算符

Java运算符按功能可分为：算数运算符、关系运算符、逻辑运算符、位运算符、赋值运算符和条件运算符。

### 3.1 算数运算符

算术运算符包括通常的加（+）、减（-）、乘（*）、除（/）、取模（%），完成整数型和浮点型数据的算术运算。

许多语言中的取模运算只能用于整数型，Java对此做了扩展，它允许对浮点数进行取模操作。例如，3%2 的结果是 1, 15.2%5 的结果是 0.2。取模操作还可以用于负数，结果的符号与第一个操作数的符号相同，例如，5%-3 的结果是 2，-5%3 的结果是-2。

此外，算术运算符还有“++”和“--”两种，分别称为加1和减1运算符。这两种运算符有前缀形式和后缀形式，含有有所不同。例如，i++ 和 ++i 的执行顺序是不一样的，i++ 在 i 使用之后再 +1，++i 在 i 使用之前先 +1。i-- 和 --i 的情况于此类似。

```java
// 例子
int i = 1;
System.out.println(i++); // i++, 使用之后 +1, 此处输出1

i = 1;
System.out.println(++i); // ++i, 先+1再使用，此处输出2

i = 1;
System.out.println(i--); // i--, 使用之后-1, 此处输出1

i = 1;
System.out.println(--i); // --i, 先-1再使用，此处输出0
```



### 3.2 关系运算符

关系运算符用来比较两个值，包括大于（>）、小于（<）、大于等于（>=）、小于等于（<=）、等于（==）和不等于（!=）6种。关系运算符都是二元运算符，也就是每个运算符都带有两个操作数，运算的结果是一个逻辑值。Java允许“==”和“!=”两种运算符用于任何数据类型。例如，既可以判断两个数的值是否相等，也可以判断对象或数组的实例是否相等。判断实例时比较的是两个对象在内存中的引用地址是否相等。

### 3.3 逻辑运算符

逻辑运算符包括逻辑与（&&）、逻辑或（||）和逻辑非（!）。前两个是二元运算符，后一个是一元运算符。Java对逻辑与和逻辑或提供“短路”功能，也就是在进行运算时，先计算运算符左侧的表达式的值，如果使用该值能得到整个表达式的值，则跳过运算符右侧表达式的计算，否则计算运算符右侧表达式，并得到整个表达式的值。

```java
// 逻辑与 &&， 有一个表达式的结果是false，整体结果就是false
String str = null;
if (str != null && str.length() > 2) {
    // str是null, &&左侧表达式的结果是false, 不会计算&&右侧表达式，直接整体返回false
}

str = "abc";
if (str != null && str.length() > 5) {
    // &&左侧表达式的结果是true, 会接着计算&&右侧的表达式，右侧表达式的值是false, 整个if中的表达式的结果就是返回false
}

// 逻辑或 ||， 有一个表达式的结果是true，整体就返回true。
str = "qq";
if (str == null || str.length() == 2) {
    // || 左侧的表达式的结果是false, 会接着计算 || 右侧的表达式，右侧结果是true，则整体返回true。
}

if (str != null || str.length() == 2) {
    // || 左侧的表达式的结果是true, 不会计算 || 右侧的表达式，直接整体返回true
}

// 逻辑非!, 表达式结果是true, 就返回false; 表达式结果是false，就返回true
boolean flag = true;
System.out.println(!flag); // 输出false
flag = false;
System.out.println(!flag); // 输出true;
```

### 3.4 位运算符

位运算符用来对二进制位进行操作，包括按位取反（~）、按位与（&）、按位或（|）、异或（^）、右移（>>）、左移（<<）和无符号右移（>>>）。位运算符只能对整数型和字符型数据进行操作。

**取反（~）**

参加运算的一个数据，按二进制位进行“取反”运算。

运算规则：~1=0； ~0=1；

即：对一个二进制数按位取反，即将0变1，1变0。

**按位与（&）**

参加运算的两个数据，按二进制位进行“与”运算。

运算规则：0&0=0; 0&1=0; 1&0=0; 1&1=1；即：两位同时为“1，结果才为“1，否则为0。

例如：3&5 即 0000 0011 & 0000 0101 = 0000 0001 因此，3 & 5的值得1。

**按位或（|）**

参加运算的两个对象，按二进制位进行“或”运算。

运算规则：0 | 0=0； 0 | 1=1； 1 | 0=1； 1 | 1=1；

即 ：参加运算的两个对象只要有一个为1，其值为1。

例如：3 | 5，即 0000 0011 | 0000 0101 = 0000 0111 因此，3 | 5的值得7。

**异或（^）**

参加运算的两个数据，按二进制位进行“异或”运算。

运算规则：0^0=0； 0^1=1； 1^0=1； 1^1=0；

即：参加运算的两个对象，如果两个相应位为“异”（值不同），则该位结果为1，否则为0。

**左移（<<）**

运算规则：按二进制形式把所有的数字向左移动对应的位数，高位移出（舍弃），低位的空位补零。例如： 12345 << 1，则是将数字12345左移1位：

![img](images/1ad5ad6eddc451da078a1edf5b646c60d0163221.jpeg)

位移后十进制数值变成：24690，刚好是12345的二倍，所以有些人会用左位移运算符代替乘2的操作，但是这并不代表是真的就是乘以2，很多时候，我们可以这样使用，但是一定要知道，位移运算符很多时候可以代替乘2操作，但是这个并不代表两者是一样的。

思考一下：如果任意一个十进制的数左位移32位，右边补位32个0，十进制岂不是都是0了？当然不是！！！ 当int 类型的数据进行左移的时候，当左移的位数大于等于32位的时候，位数会先求余数，然后再进行左移，也就是说，如果真的左移32位 12345 << 32 的时候，会先进行位数求余数，即为 12345<<(32%32) 相当于 12345<< 0 ，所以12345<< 33 的值和12345<<1 是一样的，都是 24690。

**右移（>>）**

同样，还是以12345这个数值为例，12345右移1位： 12345>>1。

![img](images/d6ca7bcb0a46f21f19ec19641bbd55660d33ae49.jpeg)

右移后得到的值为 6172 和int 类型的数据12345除以2取整所得的值一样，所以有些时候也会被用来替代除2操作。另外，对于超过32位的位移，和左移运算符一样，，会先进行位数求余数。

**无符号右移（>>>）**

无符号右移运算符和右移运算符是一样的，不过无符号右移运算符在右移的时候是补0的，而右移运算符是补符号位的。以下是-12345二进制表示：

![img](images/21a4462309f790522f7425fafd6ae9cc7acbd52f.jpeg)

对于源码、反码、补码不熟悉的同学，请自行学习，这里就不再进行补充了讲解了，这里提醒一下，在右移运算符中，右移后补0，是由于正数 12345 符号位为0 ，如果为1，则应补1。

![img](images/b17eca8065380cd7f23c4b284ddd933258828171.jpeg)



**原码、反码和补码说明：**

一个数可以分成符号位（0正1负）+ 真值，原码是我们正常想法写出来的二进制。由于计算机只能做加法，负数用单纯的二进制原码书写会出错，于是大家发明了反码（正数不变，负数符号位不变，真值部分取反）；再后来由于+0， -0的争端，于是改进反码，变成补码（正数不变，负数符号位不变，真值部分取反，然后+1）。二进制前面的0都可以省略，所以总结来说：计算机里的负数都是用补码（符号位1，真值部分取反+1）表示的。

**位运算符和2的关系**

位运算符和乘2、除2在大多数时候是很相似的，可以进行替代，同时效率也会高的多，但是两者切记不能混淆 ；很多时候有人会把两者的概念混淆，尤其是数据刚好是 2、4、6、8、100等偶数的时候，看起来就更相似了，但是对于奇数，如本文使用的12345 ，右移之后结果为6172 ，这个结果就和数学意义上的除以2不同了，不过对于int 类型的数据，除2 会对结果进行取整，所以结果也是6172 ，这就更有迷惑性了。

### 3.5 赋值运算符

赋值运算符的作用就是将常量、变量或表达式的值赋给某一个变量。

| 运算符 | 运算   | 范例          | 结果     |
| ------ | ------ | ------------- | -------- |
| =      | 赋值   | a=3;b=3       | a=3;b=2; |
| +=     | 加等于 | a=3;b=2;a+=b; | a=5;b=2; |
| -=     | 减等于 | a=3;b=2;a-=b; | a=1;b=2; |
| *=     | 乘等于 | a=3;b=2;a*=b; | a=6;b=2; |
| /=     | 除等于 | a=3;b=2;a/=b; | a=1;b=2; |
| %=     | 模等于 | a=3;b=2;a%=b; | a=1;b=2; |

除了“=”，其它的都是特殊的赋值运算符，以“+=”为例，x += 3就相当于x = x + 3，首先会进行加法运算x+3，再将运算结果赋值给变量x。-=、*=、/=、%=赋值运算符都可依此类推。

### 3.6 条件运算符

条件运算符（ ? : ）也称为 “三元运算符”或“三目运算符”。

语法形式：布尔表达式 ？ 表达式1 ：表达式2。

运算过程：如果布尔表达式的值为true，则返回表达式1的值，否则返回表达式2的值。

### 3.7 运算符的优先次序

在对一个表达式进行计算时，如果表达式中含有多种运算符，则要安运算符的优先次序一次从高向低进行。运算符的优先次序如下：

![img](images/a2cc7cd98d1001e96323d451569745ea55e797f5.png)



## 四、控制执行流程

### 4.1 if else

只有一个if

```java
if(布尔表达式){

} 
```

一个if 一个else

```java
if(布尔表达式){

}else{

}
```

一个if N个else if 可能可无else

```java
if(布尔表达式){

}else if(布尔表达式){

}else{

}
```

### 4.2 for循环

```java
for (初始化; 布尔表达式; 运算) {

}
```

每次循环都会进行布尔表达式的判断，布尔表达式为true时，相应的也会执行一次运算，当布尔表达式为false时，循环终止。

### 4.3 do while循环

```java
while (布尔表达式) {

}
```

与do while不同，while是先执行布尔表达式的判断，然后执行代码块，两者的选择就看布尔表达式的位置。

### 4.4 while循环

```java
while (布尔表达式) {

}
```

与do while不同，while是先执行布尔表达式的判断，然后执行代码块，两者的选择就看布尔表达式的位置。

### 4.5 break continue

任何迭代语句的代码块部分，都可以用break和continue控制循环的流程


### 4.5.1 break 终止并跳出循环

```java
int[] ints = {1,2,3};
for (int i : ints) {
    if(i == 2){
        break;
    }
    System.out.println(i);
}
//输出
//1
```

### 4.5.2 continue 结束当次循环，进入下次循环

```java
int[] ints = {1,2,3};
for (int i : ints) {
    if(i == 2){
        continue;
    }
    System.out.println(i);
}
//输出
//1 3
```

当有多层循环时，break和continue只对当前循环生效。这时候可以用另一种语法:

```java
out:
for (int i : ints) {
    for (int j : ints) {
        if(j == 2){
            continue out;
        }
        System.out.println(j);
    }
}
//输出
//1 1 1
```

跳到外层循环继续执行，break的用法也是类似。

### 4.6 return语句

方法直接返回，不再执行

### 4.7 switch语句

根据值来选择执行的代码块

```java
switch (key) {	//	值
    case value:	//	多个case下的value都是唯一的 case块也没有  key与值相等则执行代码块
        break;	//	可有可无 有的话直接跳出 没有的话 继续往下执行
    default:	//	如果前面都没有执行break 则执行
        break;
}
```

## 五、数组

### 5.1 数组的定义

数组是相同类型数据的有序集合。数组描述的是相同类型的若干个数据，按照一定的先后次序排列组合而成。其中，每一个数据称作一个元素，每个元素可以通过一个索引(下标)来访问它们。

### 5.2 数组的基本特点

1）长度是确定的。数组一旦被创建，它的大小就是不可以改变的。
2）其元素必须是相同类型，不允许出现混合类型。元素的类型可以是java 支持的任意类型
3）数组类型可以是任何数据类型，包括基本类型和引用类型。
4）数组的元素在堆内存中被分配空间，并且是连续分配的
5）使用new 关键字对数组进行 内存的分配。每个元素都会被jvm 赋予默认值。默认规则：整数：0 浮点数：0.0 字符：\u0000 布尔：false 引用数据类型：null。
6）数组的元素都是有序号的，序号从0开始，0序的。称作数组的下标、索引、角标

### 5.3 数组的声明

1）声明的时候并没有实例化任何对象，只有在实例化数组对象时，JVM才分配空间，这时才与长度有关。
2）声明一个数组的时候并没有数组真正被创建。
3）构造一个数组，必须指定长度

### 5.4  数组初始化

静态初始化：

```java
静态初始化：int[] arr = { 1, 2, 3 }; // 静态初始化基本类型数组；
```

动态初始化：

```java
int[] arr = new int[2]; // 动态初始化数组，先分配空间；
arr[0]=1; // 给数组元素赋值；
arr[1]=2; // 给数组元素赋值；
```

默认初始化：

```java
int arr[] = new int[2]; // 默认值：0,0
boolean[] b = new boolean[2]; // 默认值：false,false
String[] s = new String[2]; // 默认值：null, null
```



例子：

![img](images/1211462-20180901151338439-769400636.png)

![img](images/1211462-20180901151431717-277396520.png)

### 5.5 数组的遍历

fo循环

```java
int[] a = new int[4];
// 初始化数组元素的值
for (int i=0; i < a.length; i++) {
    a[i] = 100 * i;
}
// 读取元素的值
for (int i=0; i < a.length; i++) {
    System.out.println(a[i]);
}
```

for-each循环

```java
String[] ss = {"aa", "bbb", "ccc", "ddd"};
for (String temp : ss) {
    System.out.println(temp);
}
```



### 5.6 拷贝、删除、扩容操作

System类里也包含了一个`static void arraycopy(object src，int srcpos，object dest， int destpos，int length)`方法，该方法可以将src数组里的元素值赋给dest数组的元素，其中srcpos指定从src数组的第几个元素开始赋值，length参数指定将src数组的多少个元素赋给dest数组的元素。

函数原型如下：

```java
public static native void arraycopy(Object src,  int  srcPos,
                                    Object dest, int destPos,
                                    int length);
```

使用`arraycopy`函数实现的数组拷贝、删除、扩容操作如下：

```java
package com.boom.arrays;

/**
 * 关于数组的操作:拷贝，删除，扩容
 * 
 * @author Administrator
 *
 */
public class ArrayCopyTest {

    public static void main(String[] args) {
        arrayCopy();

        //        String[] str = { "Java", "C", "C++", "Python", "JScript" };
        //        removeElement(str, 1);
        //        
        //        String[] str = { "Java", "C", "C++", "Python", "JScript" };
        //        extendRange(str);


    }

    // 数组的拷贝
    public static void arrayCopy() {
        String[] s1 = { "aa", "bb", "cc", "dd", "ee", };
        String[] s2 = new String[7];
        // 从s1 里下标为1的数组开始拷贝到s2,存放在s2里下标为2的位置开始，拷贝3个数组。
        System.arraycopy(s1, 1, s2, 2, 3);
        for (int i = 0; i < s2.length; i++) {
            System.out.print(s2[i] + " ");
        }
    }

    // 删除数组中指定索引的位置,并返回原数组.实则还是拷贝数组，再覆盖原来的数组
    public static String[] removeElement(String[] s, int index) {
        System.arraycopy(s, index + 1, s, index, s.length - index - 1);
        // 特殊处理最后一个数组
        s[s.length - 1] = null;
        for (int i = 0; i < s.length; i++) {
            System.out.print(s[i] + " " + "\n");
        }
        return s;
    }

    // 数组的扩容
    public static String[] extendRange(String[] s1){
        // 传入的数组基础上空间+3
        String[] s2  = new String[s1.length+3];
        System.arraycopy(s1, 0, s2, 0, s1.length);
        for(int i = 0; i<s2.length;i++){
            System.out.println(s2[i]);
        }
        return s2;   
    }
}
```

### 5.7 java.util.Arrays类

JDK提供的java.util.Arrays类，包含了常用的数组操作，方便我们日常开发。Arrays类包含了：排序、查找、填充、打印内容等常见的操作。

**打印数组**`Arrays.toString(arr)`

```java
int[] a= {1, 2};
System.out.println(a); // 打印数组引用的值
System.out.println(Arrays.toString(a)); // 打印数组元素的值
```

输出结果:

```bash
[I@15db9742
[1, 2]
```

**数组元素的排序**`Arrays.toString(arr)`

```java
int[] arr = {1, 2, 323, 23, 543, 12, 59};
System.out.println(Arrays.toString(arr));
Arrays.sort(arr);
System.out.println(Arrays.toString(arr));
```

输出结果:

```bash
[1, 2, 323, 23, 543, 12, 59]
[1, 2, 12, 23, 59, 323, 543]
```

**二分查找**`Arrays.binarySearch(arr, key)`

```java
int[] arr = {1, 2, 323, 23, 543, 12, 59};
System.out.println(Arrays.toString(arr));
Arrays.sort(arr);  // 使用二分查找，必须先对数组进行排序
System.out.println(Arrays.toString(arr));
// 返回排序后新的索引位置，若未找到返回负数。
System.out.println("该元素的索引：" + Arrays.binarySearch(arr, 12));
```

输出结果：

```java
[1, 2, 323, 23, 543, 12, 59]
[1, 2, 12, 23, 59, 323, 543]
该元素的索引：2
```

**数组填充**`Arrays.fill(...)`

```java
int[] arr = {1, 2, 323, 23, 543, 12, 59};
System.out.println(Arrays.toString(arr));
Arrays.fill(arr, 2, 4, 100);
System.out.println(Arrays.toString(arr));
```

输出结果:

```bash
[1, 2, 323, 23, 543, 12, 59]
[1, 2, 100, 100, 543, 12, 59]
```

### 5.8 数组的排序

数组的排序算法有冒泡排序、选择排序、插入排序、归并排序、快速排序、基数排序、堆排序。

<font color="red">TODO 补充上述算法的实现</font>

### 5.9 数组的查找

数组的查找有遍历搜索、二分查找。

<font color="red">TODO 补充上述算法的实现</font>

### 5.10 数组的常见问题

```java
int []arr = new int[3];
System.out.println(arr[3]); // ArrayIndexOutofBoundsException
// 当访问到数组中不存在的时下标时：下标越界

arr = null;
System.out.println(arr[0]); // NullPointerException
// 当引用类型变量没有指向任何实体时，继续访问引用类型变量，发生空指针异常。

System.out.println(arr); // [I@3219ab8d:哈希值
```

### 5.11 数组的内存分析

![img](images/20161030125110163.jpeg)

![img](images/20161030125422730.jpeg)

![img](images/1211462-20180901152037185-1093109634.png)



### 5.12 数组的优缺点

**优点**

1：可以保存若干个数据。

2：随机访问的效率很高。根据下标访问元素效率高的原因是，元素存放在连续分配的空间上。

**缺点**

1：数组的元素类型必须一致。

2：连续分配的空间在堆中，如果数组的元素很多，对内存的要求更加严格。

3：根据内容查找元素效率比较低，需要逐个比较个。

4：删除元素、插入元素效率比较低，需要移动大量的元素。

5：数组定长，不能自动扩容。

6：数组没有封装，数组对象只提供了一个数组长度的属性，但是没有提供方法用来操作元素。

java 提供了一整套的 针对不同需求的 对于容器的解决的方案。集合框架部分。不同的容器有不同的特点，满足不同的需求。数组的缺点都会被干掉。

### 5.13 二维数组

二维数组实质就是存储元素是一维数组的数组。

**二维数组定义：**

数组类型[][] 数组名 = new 数组类型[一维数组的个数][每一个一维数组中元素的个数];

![img](images/20161030125652145.jpeg)

疑问： 为什么a.length = 3, a[0].length = 4?

![img](images/20161030125751375.jpeg)

**二维数组的初始化：**

静态初始化:  

```java
int [][] a = new int[][]{ {12,34,45,89},{34,56,78,10},{1,3,6,4} };
```

动态初始化：

```java
int[][] a = new int[3][4];   // 长度为3*4=12
for (int i=0; i < a.length; i++) {
    for (int j=0; j < a[i].length; j++) {
        a[i][j] =  ++value; // 例如：a[1][0] = 4;
    }
}
```



## 六、字符串

字符串是内存中连续排列的0个或多个字符。不变字符串是指字符串一旦创建，其内容就不能改变，Java中使用String类来处理不变字符串。

### 6.1 常量字符串与变量字符串

Java程序中的字符串分为常量和变量两种，其中，字符串常量使用双引号括起来的一串字符，系统为程序中出现的字符串常量自动创建一个String对象。例如：

```java
System.out.println("hello world!");
```

这句代码将创建一个String对象，值为“hello world!”。

对于字符串变量，在使用之前要显式声明，并进行初始化。字符串的声明方式有三种：

**直接创建**：`String str1 = "Hello";`

字符串是对象，虽然我们在这里没有用new创建对象，其实是编译器给我们做了这些操作。这种创建的字符串对象有一个特点，如果同样的对象如果存在了，就不会创建一个新的对象，而是指向了同样的对象。例如String str2 = "Hello";，则str1和str2是指向了字符串池中同样的内存地址，即 str1 == str2 为true。

**使用字符串连接创建：** `String str = "Hello" + "World";`

这种形式其实可以看做是第一种的形式的特殊形式。 "Hello" + "World"在编译期会被自动折叠为常量“HelloWorld”，所以，最后只会创建一个对象：String str = "HelloWorld";

JDK1.7开始，javac会进行常量折叠，全字面量字符串相加是可以折叠为一个字面常量，而且是进入常量池的。这个问题涉及到了**字符串常量池**和**字符串拼接**。`String a="a"+"b"+"c";`通过编译器优化后，得到的效果是：`String a="abc";`

**new创建字符串**： `String str1 = new String("Hello");`

用new关键字创建的字符串每次都会创建一个新的对象。即使这时创建一个字符串`String str2 = new String("Hello");`，str1与str2是两个对象，str1 == str2 为false。

![img](images/v2-784dc6d3bd25910757178f70f0e67a06_1440w.jpg)

**注意点：** `String str = new String("Hello");` 会产生几个对象？
如果字符串池里面没有“Hello”对象，会在字符串池里面生成一个对象，然后再生成一个字符串对象，str指向这个对象；如果字符串池里面已经有了“Hello”对象，则只会生成一个对象，str指向这个对象。

### 6.2 字符串不可变性的解释

面试经常会碰到一个问题，就是String不可变，大部分答的时候会讲因为String的源码里面，它是这样的：

```java
/** The value is used for character storage. */
    private final char value[];
```

它是被final修饰的，被final修饰的真正含义是什么呢？一定能讲出是不可变，那么到底是什么不可变啊？我们可以来试一试:

```java
 final int[] value = new int[]{1,2,3};
 value = new int[]{1,2,4};
```

会报错

```java
Error:(33, 9) java: 无法为最终变量value分配值
```

这说明引用不可变，value不能再指向另一个变量，但是这能说明value的值不可变吗？我们再来试试看：

```java
final int[] value = new int[]{1,2,3};
value[1]=4;
System.out.println(value[0]+" "+value[1]+" "+value[2]);
```

输出就是：

```java
1 4 3
```

因此，final不可变指的是引用对象不可变，而不是对象的值不可变，那么到底是什么让String对象不可变呢？再去源码看看，是不是还有个private，这个让value的值在外部是不可变的。

### 6.3 字符串常量池

作为最基础的引用数据类型，Java 设计者为 String 提供了字符串常量池以提高其性能，那么字符串常量池的具体原理是什么，我们带着以下三个问题，去理解字符串常量池：

- 字符串常量池的设计意图是什么？

- 字符串常量池在哪里？

- 如何操作字符串常量池？

#### （1）字符串常量池的设计思想

字符串的分配和其他对象分配一样，是需要消耗高昂的时间和空间的，而且字符串使用的非常多。JVM为了提高性能和减少内存的开销，在实例化字符串的时候进行了一些优化：使用字符串常量池。由于String字符串的不可变性，常量池中一定不存在两个相同的字符串。

JVM为了提高性能和减少内存开销，在实例化字符串常量的时候进行了一些优化：

- 为字符串开辟一个字符串常量池，类似于缓存区
- 创建字符串常量时，首先检查字符串常量池是否存在该字符串
- 存在该字符串，返回引用实例；不存在，实例化该字符串并放入池中

实现的基础如下：

- 实现该优化的基础是因为字符串是不可变的，可以不用担心数据冲突进行共享
- 运行时实例创建的全局字符串常量池中有一个表，总是为池中每个唯一的字符串对象维护一个引用,这就意味着它们一直引用着字符串常量池中的对象，所以，在常量池中的这些字符串不会被垃圾收集器回收

从字符串常量池中获取相应的字符串，代码示例：

```java
String str1 = “hello”;
String str2 = “hello”;

System.out.printl（"str1 == str2" : str1 == str2 ) //true 
```

####  （2）字符串常量池在哪里

在分析字符串常量池的位置时，首先了解一下堆、栈、方法区：

![820e4d34-b89d-33a6-a776-4c088a08d2a9.png](images/java内存分布.png)

堆区：

- 存储的是对象，每个对象都包含一个与之对应的class
- JVM只有一个堆区(heap)被所有线程共享，堆中不存放基本类型和对象引用，只存放对象本身
- 对象的由垃圾回收器负责回收，因此大小和生命周期不需要确定

栈区：

- 每个线程包含一个栈区，栈中只保存基础数据类型的对象和自定义对象的引用(不是对象)
- 每个栈中的数据(原始类型和对象引用)都是私有的
- 栈分为3个部分：基本类型变量区、执行环境上下文、操作指令区(存放操作指令)
- 数据大小和生命周期是可以确定的，当没有引用指向数据时，这个数据就会自动消失

方法区：

- 静态区，跟堆一样，被所有的线程共享
- 方法区中包含的都是在整个程序中永远唯一的元素，如class，static变量。

**字符串常量池内存区域**

在HotSpot VM中字符串常量池是通过一个StringTable类实现的，它是一个Hash表，默认值大小长度是1009；这个StringTable在每个HotSpot VM的实例中只有一份，被所有的类共享；字符串常量由一个一个字符组成，放在了StringTable上。要注意的是，如果放进String Pool的String非常多，就会造成Hash冲突严重，从而导致链表会很长，而链表长了后直接会造成的影响就是当调用String.intern时性能会大幅下降（因为要一个一个找）。

在JDK6及之前版本，字符串常量池是放在Perm Gen区(也就是方法区)中的，StringTable的长度是固定的1009；在JDK7版本中，字符串常量池被移到了堆中，StringTable的长度可以通过**-XX:StringTableSize=66666**参数指定。至于JDK7为什么把常量池移动到堆上实现，原因可能是由于方法区的内存空间太小且不方便扩展，而堆的内存空间比较大且扩展方便。


**字符串常量池中存放的内容**

在JDK6及之前版本中，String Pool里放的都是字符串常量；在JDK7.0中，由于String.intern()发生了改变，因此String Pool中也可以存放放于堆内的字符串对象的引用。

```java
String s1 = "AB";
String s2 = "AB";
String s3 = new String("AB");
System.out.println(s1 == s2); // true
System.out.println(s1 == s3); // false
```

由于常量池中不存在两个相同的对象，所以s1和s2都是指向JVM字符串常量池中的"AB"对象。new关键字一定会产生一个对象，并且这个对象存储在堆中。所以String s3 = new String(“AB”);产生了两个对象：保存在栈中的s3和保存堆中的String对象。

![这里写图片描述](images/字符串常量池内存示意图.jpeg)

当执行String s1 = "AB"时，JVM首先会去字符串常量池中检查是否存在"AB"对象，如果不存在，则在字符串常量池中创建"AB"对象，并将"AB"对象的地址返回给s1；如果存在，则不创建任何对象，直接将字符串常量池中"AB"对象的地址返回给s1。

**字符串对象的创建**

面试题：String str4 = new String(“abc”) 创建多少个对象？

step 1  在常量池中查找是否有“abc”对象，有则返回对应的引用实例，没有则创建对应的实例对象。

step 2  在堆中 new 一个 String("abc") 对象。

step 3  将对象地址赋值给str4,创建一个引用。

所以，常量池中没有“abc”字面量则创建两个对象，否则创建一个对象，以及创建一个引用

**延伸**

基础类型的变量和常量，变量和引用存储在栈中，常量存储在常量池中

```java
int a1 = 1;
int a2 = 1;
int a3 = 1;

public static int INT1 =1 ;
public static int INT2 =1 ;
public static int INT3 =1 ;
```

![798b04f9ga8e2c6779148&690](images/int类型常量池示意图.png)




#### （3）字面量和常量池的关系

字符串对象内部是用字符数组存储的，那么看下面的例子。

```java
// 会分配一个11长度的char数组，并在常量池分配一个由这个char数组组成的字符串，然后由m去引用这个字符串
String m = "hello,world";
// 用n去引用常量池里边的字符串，所以n和m引用的是同一个对象。
String n = "hello,world";
// 生成一个新的字符串，但内部的字符数组引用着m内部的字符数组。
String u = new String(m);
// 同样会生成一个新的字符串，但内部的字符数组引用常量池里边的字符串内部的字符数组，意思是和u是同样的字符数组。
String v = new String("hello,world");
```

使用图来表示的话，情况就大概是这样的(使用虚线只是表示两者其实没什么特别的关系)：

![string1.png](images/字符串创建示意图.png)

测试demo代码如下：

```java
String m = "hello,world";
String n = "hello,world";
String u = new String(m);
String v = new String("hello,world");

System.out.println(m == n); //true 
System.out.println(m == u); //false
System.out.println(m == v); //false
System.out.println(u == v); //false 
```

通过上述代码得到的结论如下：

- m和n指向的是同一个对象。
- m,u,v都是指向不同的对象。
- m,u,v,n但都使用了同样的字符数组，并且用equal判断的话也会返回true。



### 6.4 intern()方法

直接使用双引号声明出来的String对象会直接存储在字符串常量池中，如果不是用双引号声明的String对象，可以使用String提供的intern方法。intern 方法是一个native方法，intern方法会从字符串常量池中查询当前字符串是否存在，如果存在，就直接返回当前字符串；如果不存在就会将当前字符串放入常量池中，之后再返回。

JDK1.7的改动：

1. 将String常量池 从 Perm 区移动到了 Java Heap区
2. String.intern() 方法时，如果存在堆中的对象，会直接保存对象的引用，而不会重新创建对象。

下面是jdk中`intern()`方法的详细内容：

```java
/**
 * Returns a canonical representation for the string object.
 * <p>
 * A pool of strings, initially empty, is maintained privately by the
 * class {@code String}.
 * <p>
 * When the intern method is invoked, if the pool already contains a
 * string equal to this {@code String} object as determined by
 * the {@link #equals(Object)} method, then the string from the pool is
 * returned. Otherwise, this {@code String} object is added to the
 * pool and a reference to this {@code String} object is returned.
 * <p>
 * It follows that for any two strings {@code s} and {@code t},
 * {@code s.intern() == t.intern()} is {@code true}
 * if and only if {@code s.equals(t)} is {@code true}.
 * <p>
 * All literal strings and string-valued constant expressions are
 * interned. String literals are defined in section 3.10.5 of the
 * <cite>The Java&trade; Language Specification</cite>.
 *
 * @return  a string that has the same contents as this string, but is
 *          guaranteed to be from a pool of unique strings.
 */
public native String intern();
```



**intern() 的用法**

```java
static final int MAX = 1000 * 10000;
static final String[] arr = new String[MAX];

public static void main(String[] args) throws Exception {
    Integer[] DB_DATA = new Integer[10];
    Random random = new Random(10 * 10000);
    for (int i = 0; i < DB_DATA.length; i++) {
        DB_DATA[i] = random.nextInt();
    }
    long t = System.currentTimeMillis();
    for (int i = 0; i < MAX; i++) {
        //arr[i] = new String(String.valueOf(DB_DATA[i % DB_DATA.length]));
         arr[i] = new String(String.valueOf(DB_DATA[i % DB_DATA.length])).intern();
    }

    System.out.println((System.currentTimeMillis() - t) + "ms");
    System.gc();
}
```

运行的参数是：-Xmx2g -Xms2g -Xmn1500M 上述代码是一个演示代码，其中有两条语句不一样，一条是未使用 intern，一条是使用 intern。结果如下图

未使用intern，耗时826ms：

![这里写图片描述](images/未使用intern耗时.jpeg)

使用intern，耗时2160ms：

![这里写图片描述](images/使用intern耗时.jpeg)

细心的同学会发现使用了 intern 方法后时间上有了一些增长。这是因为程序中每次都是用了 new String 后，然后又进行 intern 操作的耗时时间。

### 6.5 "+" 连接符

#### （1）"+" 链接符的实现原理

Java语言为“+”连接符以及对象转换为字符串提供了特殊的支持，字符串对象可以使用“+”连接其他对象。其中字符串连接是通过 StringBuilder（或 StringBuffer）类及其append 方法实现的，对象转换为字符串是通过 toString 方法实现的，该方法由 Object 类定义，并可被 Java 中的所有类继承。有关字符连接和转换的更多信息，可以参阅 Gosling、Joy 和 Steele 合著的 《The Java Language Specification》。

我们可以通过反编译验证一下

```java
// 示例代码
public class Test {
    public static void main(String[] args) {
        int i = 10;
        String s = "abc";
        System.out.println(s + i);
    }
}
```

反编译后的结果如下：

```java
public class Test {
    public static void main(String args[]) {    //删除了默认构造函数和字节码
        byte byte0 = 10;      
        String s = "abc";      
        System.out.println((new StringBuilder()).append(s).append(byte0).toString());
    }
}
```

由上可以看出，Java中使用"+"连接字符串对象时，会创建一个StringBuilder()对象，并调用append()方法将数据拼接，最后调用toString()方法返回拼接好的字符串。由于append()方法的各种重载形式会调用String.valueOf方法，所以我们可以认为：

```java
//以下两者是等价的
s = i + "";
s = String.valueOf(i);

//以下两者也是等价的
s = "abc" + i;
s = new StringBuilder("abc").append(i).toString();
```

#### （2）“+”连接符的效率

使用“+”连接符时，JVM会隐式创建StringBuilder对象，这种方式在大部分情况下并不会造成效率的损失，不过在进行大量循环拼接字符串时则需要注意。

```java
String s = "abc";
for (int i=0; i<10000; i++) {
    s += "abc";
}

/**
 * 反编译后
 */
String s = "abc";
for(int i = 0; i < 1000; i++) {
    s = (new StringBuilder()).append(s).append("abc").toString();    
}
```

这样由于大量StringBuilder创建在堆内存中，肯定会造成效率的损失，所以在这种情况下建议在循环体外创建一个StringBuilder对象调用append()方法手动拼接（如上面例子如果使用手动拼接运行时间将缩小到1/200左右）。

```java
/**
 * 循环中使用StringBuilder代替“+”连接符
 */
StringBuilder sb = new StringBuilder("abc");
for (int i = 0; i < 1000; i++) {
    sb.append("abc");
}
sb.toString();
```

与此之外还有一种特殊情况，也就是当"+"两端均为编译期确定的字符串常量时，编译器会进行相应的优化，直接将两个字符串常量拼接好，例如：

```java
System.out.println("Hello" + "World");

/**
 * 反编译后
 */
System.out.println("HelloWorld");
```

```java
/**
 * 编译期确定
 * 对于final修饰的变量，它在编译时被解析为常量值的一个本地拷贝存储到自己的常量池中或嵌入到它的字节码流中。
 * 所以此时的"a" + s1和"a" + "b"效果是一样的。故结果为true。
 */
String s0 = "ab"; 
final String s1 = "b"; 
String s2 = "a" + s1;  
System.out.println((s0 == s2)); //result = true
```

```java
/**
 * 编译期无法确定
 * 这里面虽然将s1用final修饰了，但是由于其赋值是通过方法调用返回的，那么它的值只能在运行期间确定
 * 因此s0和s2指向的不是同一个对象，故程序执行结果为false。
 */
String s0 = "ab"; 
final String s1 = getS1(); 
String s2 = "a" + s1; 
System.out.println((s0 == s2)); //result = false 
 
public String getS1() {  
    return "b";   
}
```

综上，“+”连接符对于直接相加的字符串常量效率很高，因为在编译期间便确定了它的值，也就是说形如"I"+“love”+“java”; 的字符串相加，在编译期间便被优化成了"Ilovejava"。对于间接相加（即包含字符串引用，且编译期无法确定值的），形如s1+s2+s3; 效率要比直接相加低，因为在编译器不会对引用变量进行优化。



### 6.6 字符串的操作

字符串创建以后，可以使用字符串类中的方法对它进行操作。日常开发中常用的操作字符串的方法有：

#### （1）String当中与获取相关的常用方法

`public int length()`：获取字符串当中含有的字符个数，拿到字符串长度。

`public String concat(String str)`：将当前字符串和参数字符串**拼接**成为返回值新的字符串。

`public char charAt(int index)`：获取指定索引位置的单个字符，索引从0开始。

`public int indexOf(String str)`：查找参数字符串在本字符串当中首次出现的索引位置，如果没有返回-1值。

#### （2）字符串的截取方法

`public String substring(int index)`：截取从参数位置一直到字符串末尾，返回新字符串。

`public String substring(int begin, int end)`：截取从begin开始，一直到end结束，中间的字符串，不包括end。

#### （3）字符串转换的方法

`public char[] toCharArray()`：将当前字符串拆分成为字符数组作为返回值。

`public byte[] getBytes()`：获得当前字符串底层的字节数组。

`public String replace(CharSequence oldString, CharSequence newString)：`用`newString`替换所有的`oldString`。

#### （4）分割字符串

`public String[] split(String regex)`：按照正则表达式参数的规则，将字符串切分成为若干部分。

#### （5）字符串的比较

String字符串可以使用`==`和`equals()`方法比较。当两个字符串使用`==`进行比较时，比较的是两个字符串在内存中的地址。当两个字符串使用`equals`方法比较时，比较的是两个字符串的值是否相等。

### 6.7 String、StringBuffer和StringBuilder的关系

#### （1）继承结构

![这里写图片描述](images/String、StringBuilder和StringBuffer继承结构.jpeg)

#### （2）主要区别

1）String是不可变字符序列，StringBuilder和StringBuffer是可变字符序列。
2）执行速度StringBuilder > StringBuffer > String。
3）StringBuilder是非线程安全的，StringBuffer是线程安全的。

### 6.8 总结

String类是我们使用频率最高的类之一，也是面试官经常考察的题目，下面是一个小测验。

```java
public static void main(String[] args) {
    String s1 = "AB";
    String s2 = new String("AB");
    String s3 = "A";
    String s4 = "B";
    String s5 = "A" + "B";
    String s6 = s3 + s4;
    System.out.println(s1 == s2);
    System.out.println(s1 == s5);
    System.out.println(s1 == s6);
    System.out.println(s1 == s6.intern());
    System.out.println(s2 == s2.intern());
}
```

运行结果如下：

![这里写图片描述](images/String面试题运行结果.jpeg)

题目解析：真正理解此题目需要清楚以下三点
1）直接使用双引号声明出来的String对象会直接存储在常量池中；
2）String对象的intern方法会得到字符串对象在常量池中对应的引用，如果常量池中没有对应的字符串，则该字符串将被添加到常量池中，然后返回常量池中字符串的引用；
3） 字符串的+操作其本质是创建了StringBuilder对象进行append操作，然后将拼接后的StringBuilder对象用toString方法处理成String对象，这一点可以用javap -c命令获得class文件对应的JVM字节码指令就可以看出来。

![这里写图片描述](images/String面试题内存分析.jpeg)



### 6.9 参考资料

​		https://docs.oracle.com/javase/8/docs/api/
​		https://blog.csdn.net/sinat_19425927/article/details/38663461
​		https://www.cnblogs.com/xiaoxi/p/6036701.html
​		https://tech.meituan.com/in_depth_understanding_string_intern.html

