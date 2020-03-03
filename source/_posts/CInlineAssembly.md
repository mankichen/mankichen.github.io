---
title: C语言内联汇编
tags:
  - 汇编
  - C语言
categories:
  - Linux
comments: true
date: 2020-03-02 10:29:42
---




## 内联汇编格式

```c
__asm__(
	assembler template				/* 汇编语句模板 */
    : output operands				/* 输出部分 */
    : input operands 				/* 输入部分 */
    : list of clobbered registers	/* 破坏描述部分 */
);
```

如果`asm`不与程序中的变量冲突，可以代替`__asm__`，两者效果一样。下面对内联汇编中各个部分进行分析

<!-- more -->

### 汇编语句模板

这里写需要嵌入到C程序的汇编指令

* 每条指令都放在一个双引号内，或者把所有的指令都放到一个双引号内

  ```c
  asm(
      "movl r0, r0\n\t
      movl r0, r0\n\t
      movl r0, r0\n\t
      movl r0, r0"
  );
  ```

* 指令之间要用分隔符来分开（分隔符可以是`\n\t`或`;`）

  ```c
  asm(
      "movl r0, r0;"
      "movl r0, r0;"
      "movl r0, r0;"
      "movl r0, r0"
  );
  ```

* 访问C语言变量用%0,%1...等

  ```c
  int a=10, b;
  asm ( 
      "movl %1, %%eax;
      movl %%eax, %0;"
      :"=r"(b)
      :"r"(a)
      :"%eax"
  );
  ```

* 为了区分操作数和寄存器，寄存器前面多加一个%，eg: `%%eax`

### 操作数-输出/输出部分

操作数的格式如下：

```c
"constraint_1"(C_expression_1), "constraint_2"(C_expression_2), ...
```

* constraint: 约束
  * 主要用来指定操作数的寻址类型 (内存寻址或寄存器寻址)
  * 对于输出操作数要用 “=“在开头修饰
* C_expression: C表达式
  * 对于输出部分该变大时必须是一个左值，输入部分无所谓
  * 如果输出表达式不能直接寻址(比如是[bit-field]), constraint就必须指定一个寄存器。这种情况下，GCC将使用寄存器作为asm的输出。然后保存这个寄存器的值到输出表达式中。
* 操作数的编号时从第一个输出操作数(编号0),后面依次递增

#### constraints详解

constraint主要是确定指令中操作数的寻址方式；是否允许操作数位于寄存器中，以及它可以包括在哪些种类的寄存器中；操作数是否可以是内存引用，以及在这种情况下使用哪些种类的地址；操作数是否可以是立即数；

* 寄存器操作数constraints: `r`

  操作数将被存储在通用寄存器中，这个通用寄存器可能是(eax ebx ecd edx esi edi)，gcc会根据具体情况进行选择。

  同样除了自动选择，我们也可以自己制定某个特定的寄存器：

| r    | Register(s)     |
|:---|:---|
| a    | %eax, %ax, %al  |
| b    | %ebx, %bx, %bl  |
| c    | %ecx, %cx, %cl  |
| d    | %edx, %dx, %adl |
| S    | %esi, %si       |
| D    | %edi, %di       |

* 内存操作数constraint: `m`

  当操作数位于内存中时，任何对它们执行的操作都将在内存位置中直接发生，这与寄存器约束正好相反，后者先将值存储在要修改的寄存器中，然后将它写回内存位置中。但寄存器约束通常只在对于指令来说它们是绝对必需的，或者它们可以大大提高进程速度时使用。当需要在 "asm" 内部更新 C 变量，而您又确实不希望使用寄存器来保存其值时，使用内存约束最为有效。

* 匹配（数字）约束

  在某些情况下，一个变量既要充当输入操作数，也要充当输出操作数。

  ```c
  int var = 1;
  asm (
      "incl %0" 
      :"=a"(var)
      :"0"(var)
  );
  ```

  在上面的示例中，寄存器 %eax 既用作输入变量，也用作输出变量。将 var 输入读取到 %eax，增加后将更新的 %eax 再次存储在 var 中。**这里的 "0" 指定第 0 个输出变量相同的constraint。**即，它指定 var 的输出实例只应该存储在 %eax 中。该constraint可以用于以下情况：

  - 输入从变量中读取，或者变量被修改后，修改写回到同一变量中
  - 不需要将输入操作数和输出操作数的实例分开

  使用匹配constraint最重要的好处是可以更高效地使用变量寄存器。

* `o`

  使用一个内存操作数，但是要求内存地址范围在在同一段内。例如，加上一个小的偏移量来形成一个可用的地址。

* `V`

  内存操作数，但是不在同一个段内。换句话说,就是使用除了”o” 以外的”m”的所有的情况。

* `i`

  使用一个立即整数操作数(**值固定**)；也包含仅在编译时才能确定其值的**符号常量**。

* `n`

  一个确定值的立即数。很多系统不支持汇编常数操作数小于一个字(word)的长度的情况。这时候使用`n`就比使用`i`好。

  如果常数小于一个字的长度，用`n`会更好。

* `g`

  除了通用寄存器以外的任何寄存器，内存和立即整数。

#### constraint修饰符

* `=`

  只能用于输出部分，只能写在第一个字符的位置，指明这个操作数是只写的；之前保存在其中的值将被废弃而被输出值所代替。

* `+`

  只能用于输出部分，表示当前输出表达式的属性为可读可写，只能写在第一个字符的位置。

* `&`

  只能用于输出部分，用于约束寄存器的分配,但是只能写在约束部分的第二个字符的位置上。

  用符号&进行修饰时,等于向GCC声明:"GCC不得为任何Input操作表达式分配与此Output操作表达式相同的寄存器"，原因如下：

  > 其原因是修饰符&意味着被其修饰的Output操作表达式要在所有的Input操作表达式被输入之前输出;
  >
  > 即:GCC会先使用输出值对被修饰符&修饰的Output操作表达式进行填充,然后,才对Input操作表达式进行输入;
  >
  > 这样的话,如果不使用修饰符&对Output操作表达式进行修饰,一旦后面的Input操作表达式使用了与Output操作表达式相同的寄存器,就会产生输入输出数据混乱的情况;相反,如果没有用修饰符&修饰输出操作表达式,那么,就意味着GCC会先把Input操作表达式的值输入到选定的寄存器中,然后经过处理,最后才用输出值填充对应的Output操作表达式;
  >
  > 所以,修饰符&的作用就是要求GCC编译器为所有的Input操作表达式分配别的寄存器,而不会分配与被修饰符&修饰的Output操作表达式相同的寄存器;修饰符&也写在操作约束中,即:&约束;由于GCC已经规定加号(+)或等号(=)占据约束的第一个字符,那么&约束只能占用第二个字符;

  PS：不知道大家看懂了没有，我是没看懂，没看懂的没事，看看下面的示例程序就懂了。

### 破坏描述部分

* 如果汇编语句中有指令破坏了原来寄存器中值(**不论是显示还是隐式**，要注意隐式的情况)，那么我们必须在这里标示出该寄存器。
* 输入/输出部分指定的寄存器不需要在这边再次标示出来，因为GCC已经知道会用到这些寄存器。
* 如果指令中以无法预料的形式修改了内存值，需要在本部分中加上”memory”。从而使得GCC不去缓存在这些内存值。
* 如果要改变没有被列在输入和出部分的内存内容时，需要加上volatile关键字说明。
* 列出的寄存器可以被多次读写。

```c
asm( 
    "movl %0,%%eax;"
    "movl %1,%%ecx;"
    "call _foo"
    : /*no outputs*/
    : "g" (from), "g" (to)
    : "eax", "ecx"
);
```

上面的代码中显示的破坏了eax与ecx原来的值，所以需要标示出来。

## volatile

volatile是C语言中的一个关键字，该关键字可以用来修饰变量，让GCC对访问该变量的代码不进行优化，每次都从该变量所在的内存地址范访问。

同样，这个关键字也可以用在内联汇编中，表示不对本段汇编代码进行优化，表明本汇编代码必须在被放置的位置执行。

```c
__asm__ volatile (
	....
);
```



## 实例代码

### 一般内联汇编用法

#### 使用r约束

```c
int main(void)
{
    int x = 10, y;
    
    /* 简单的赋值：y = x; */
    asm (
        "movl %1, %%eax\n\t"
 		"movl %%eax, %0"
        :"=r"(y)  
        :"r"(x)    
        :"%eax"	// 破坏了eax寄存器
    );
    
    return 0;
}
```

将上面的代码生成汇编我们得到如下：

```assembly
main:
        pushl   %ebp
        movl    %esp, %ebp
        subl    $16, %esp
        movl    $10, -4(%ebp)
        movl    -4(%ebp), %edx 	# 将x存放在edx中
#APP
# 6 "main.c" 1
        movl %edx, %eax
        movl %eax, %edx
# 0 "" 2
#NO_APP
        movl    %edx, -8(%ebp) # 将edx的值放入y中
        movl    $0, %eax 
        leave
        ret

```

这里我们使用了`r`约束，所以GCC可以自由分配任何寄存器，在我们的实例中，GCC选择了%edx来存x。

#### 使用& 约束修饰符

```c
int main(void)
{
    int __out = 0, __in1 = 1, __in2 = 2;
    
	__asm__(
        "popl %0\n\t"		 	// pop栈顶的元素到后面即将被输出的寄存器eax中
        "movl %1,%%esi\n\t"
        "movl %2,%%edi\n\t"
        :"=&a"(__out)
        :"r"(__in1),"r"(__in2)
        :"%esi","%edi"
    );
    return 0;
}
```

以下是汇编

```assembly
main:
        pushl   %ebp
        movl    %esp, %ebp
        pushl   %edi	# 被破坏的寄存器提前被保存起来了edi
        pushl   %esi	# 被破坏的esi
        subl    $16, %esp
        movl    $0, -12(%ebp)
        movl    $1, -16(%ebp)
        movl    $2, -20(%ebp)
        movl    -16(%ebp), %edx	# __in1分配在edx中 
        movl    -20(%ebp), %ecx # __in2分配在ecx中
#APP
# 6 "main.c" 1
        popl %eax		# __out被优先分配在eax中，与__in1与__in2不同的寄存器
        movl %edx,%esi
        movl %ecx,%edi

# 0 "" 2
#NO_APP
        movl    %eax, -12(%ebp) # 将eax输出到__out中
        movl    $0, %eax
        addl    $16, %esp
        popl    %esi
        popl    %edi
        popl    %ebp
        ret

```

从上面的例子中，我们可以看到`__out`变量所分配的寄存器并没有与`__in1` `__in2`相同，这样就可以防止输入输出数据错乱。我们可以假设如果`__out`被分配在edx中，那么在第14行中，就会出现pop出来的值覆盖了输入值`__in1`，导致了数据错乱。

我们可以生成版没有使用`&`修饰符的汇编代码，汇编代码如下：

```assembly
main:
        pushl   %ebp
        movl    %esp, %ebp
        pushl   %edi
        pushl   %esi
        subl    $16, %esp
        movl    $0, -12(%ebp)
        movl    $1, -16(%ebp)
        movl    $2, -20(%ebp) 
        movl    -16(%ebp), %eax  # __in1分配在eax中 
        movl    -20(%ebp), %edx  # __in2分配在edx中 
#APP
# 6 "main.c" 1
        popl %eax			# Warning!!! eax又被分配了__out，导致eax的值被覆盖了
        movl %eax,%esi		# Error!!! 本来应该是将__in1的值放在esi的，现在成了栈顶的值，没有保证输入在输出产生之前被消耗(使用)
        movl %edx,%edi

# 0 "" 2
#NO_APP
        movl    %eax, -12(%ebp)
        movl    $0, %eax
        addl    $16, %esp
        popl    %esi
        popl    %edi
        popl    %ebp
        ret

```



### 使用匹配约束

```
#define _syscall4(type, name, type1, arg1, type2, arg2, type3, arg3, type4, arg4) \
        type name (type1 arg1, type2 arg2, type3 arg3, type4 arg4) \
        { \
            long __res; \
            __asm__ volatile ("int $0x80" \
                : "=a" (__res) \
                : "0" (__NR_##name),"b" ((long)(arg1)),"c" ((long)(arg2)), \
                  "d" ((long)(arg3)),"S" ((long)(arg4)) \
            ); \
            __syscall_return(type,__res); \
        }
```

在上例中，通过使用 b、c、d 和 S 约束将系统调用的四个自变量放入 %ebx、%ecx、%edx 和 %esi 中。

请注意，在输出中使用了 "=a" 约束，这样，位于 %eax 中的系统调用的返回值就被放入变量 \_\_res 中。通过将匹配约束 "0" 用作输入部分中第一个操作数约束，syscall 号 \_\_NR_##name 被放入 %eax 中，并用作对系统调用的输入。这样，这里的 %eax 既可以用作输入寄存器，又可以用作输出寄存器。没有其它寄存器用于这个目的。

另请注意，输入（syscall 号）在产生输出（syscall 的返回值）之前被消耗（使用）,所以数据正常。

### 内存操作数约束的使用

```c
__asm__ __volatile__(
    "lock; decl %0"
    :"=m" (counter)
    :"m" (counter)
);
```

翻译成汇编之后

```assembly
#APP
    lock
    decl -24(%ebp) /* counter is modified on its memory location */
#NO_APP.
```

lock:这个是一个前缀，它是为了在多处理器环境中，保证在当前指令执行期间禁止一切中断。

您可能考虑在这里为 counter 使用寄存器约束。如果这样做，counter 的值必须先复制到寄存器，递减，然后对其内存更新。但这样您会无法理解锁定和原子性的全部意图，这些明确显示了使用内存约束的必要性。

举个栗子:

如果使用寄存器约束，那么汇编将会如下：

```assembly
movl -24(%ebp), %ebx  # 先将counter存放在ebx中
decl %ebx			  # ebx自减1 
movl %ebx,-24(%ebp)   # 最后将ebx存放会counter中
```

看起来很正常，我们知道在执行第2行的指令后与第3条指令之前，counter内存中的值和寄存器中的值是不一样的，如果这个时候来了个中断，从内存中取了counter的值，那就是不正确的。



## 参考链接

本文大部分来自以下两个连接，只摘取了部分，方便后面查阅/复习，有需要了解更多，请看下面的链接：

https://www.ibm.com/developerworks/cn/linux/sdk/assemble/inline/index.html

https://www.jianshu.com/p/1782e14a0766

https://blog.csdn.net/farmwang/article/details/50153991