
## 早期（编译期）优化

### 编译器

1. 前端编译器。把*.java文件转变成*.class文件。包括Sun的Javac、Eclipse JDT中的ECJ等；
2. 虚拟机的后端运行期编译器（JIT编译器，Just In Time Compiler）。字节码转变成机器码。包括HotSpot VM的C1、C2编译器；
3. 静态提前编译器（AOT编译器，Ahead Of Time Compiler）。直接把*.java文件编译成本地机器代码。GNU Compiler for the Java。

### Javac编译器

Sun Javac的编译过程大致可以分为3个过程，分别是：

- 解析与填充符号表过程。
	- 词法、语法分析
	- 填充符号表
- 插入式注解处理器的注解处理过程。
- 分析与字节码生成过程。语义分析的主要任务是对结构上正确的源程序进行上下文有关性质的审查，如进行类型审查。
	- 标注检查。标注检查步骤检查的内容包括诸如变量使用前是否已被声明、变量与赋值之间的数据类型是否能够匹配等。以及常量折叠。
	- 数据及控制流分析。
	- 解语法糖。
	- 字节码生成。


### Java语法糖

#### 1. 泛型与类型擦除
泛型是，所操作的数据类型被指定为一个参数。这种参数类型可以用在类、接口和方法的创建中，分别称为泛型类、泛型接口和泛型方法。

泛型技术在C#和Java之中的使用方式看似相同，但实现上却有着根本性的分歧，C#里面泛型无论在程序源码中、编译后的IL中（Intermediate Language，中间语言，这时候泛型是一个占位符），或是运行期的CLR中，都是切实存在的，List<int>与List<String>就是两个不同的类型，它们在系统运行期生成，有自己的虚方法表和类型数据，这种实现称为**类型膨胀**，基于这种方法实现的泛型称为**真实泛型**。

Java语言中的泛型则不一样，它只在程序源码中存在，**在编译后的字节码文件中，就已经替换为原来的原生类型**（Raw Type，也称为裸类型）了，并且在相应的地方插入了强制转型代码，因此，对于运行期的Java语言来说，ArrayList<int>与ArrayList<String>就是同一个类，所以泛型技术实际上是Java语言的一颗语法糖，Java语言中的泛型实现方法称为**类型擦除**，基于这种方法实现的泛型称为**伪泛型**。

类型擦除前：

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/jvm/jvm_type_erasure_before.png)

把上面这段Java代码编译成Class文件，然后再用字节码反编译工具进行反编译后，将会发现泛型都不见了，程序又变回了Java泛型出现之前的写法，泛型类型都变回了原生类型。类型擦除后：

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/jvm/jvm_type_erasure_after.png)

<br/>
原生类型相同：
```
ArrayList<String> list1 = new ArrayList<String>();
list1.add("abc");

ArrayList<Integer> list2 = new ArrayList<Integer>();
list2.add(123);

System.out.println(list1.getClass() == list2.getClass()); // true
```

#### 2. 自动装箱、拆箱、遍历循环、变长参数

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/jvm/jvm_syntatic_sugar.png)

#### 3. 条件编译
todo



## 晚期（运行期）优化

在部分的商用虚拟机（Sun HotSpot、IBM J9）中，Java程序最初是通过**解释器**（Interpreter）进行解释执行的，当虚拟机发现某个方法或代码块的运行特别频繁时，就会把这些代码认定为“热点代码”（Hot Spot Code）。为了提高热点代码的执行效率，在运行时，虚拟机将会把这些代码编译成与本地平台相关的机器码，并进行各种层次的优化，完成这个任务的编译器称为**即时编译器**（Just InTime Compiler，简称JIT编译器）。

### Hotspot虚拟机内的即时编译器

尽管并不是所有的Java虚拟机都采用解释器与编译器并存的架构，但许多主流的商用虚拟机，如HotSpot、J9等，都同时包含解释器与编译器。解释器与编译器两者各有优势：

- 当程序需要迅速启动和执行的时候，解释器可以首先发挥作用，省去编译的时间，立即执行。
- 在程序运行后，随着时间的推移，编译器逐渐发挥作用，把越来越多的代码编译成本地代码之后，可以获取更高的执行效率。

HotSpot虚拟机中内置了两个即时编译器，分别称为**Client Compiler和Server Compiler**，或者简称为C1编译器和C2编译器（也叫Opto编译器）。

目前主流的HotSpot虚拟机中，默认采用解释器与其中一个编译器直接配合的方式工作，程序使用哪个编译器，取决于虚拟机运行的模式。

#### 编译对象与触发条件

在运行过程中会被即时编译器编译的“热点代码”有两类，即：

- 被多次调用的方法。
- 被多次执行的循环体。

判断一段代码是不是热点代码，是不是需要触发即时编译，这样的行为称为热点探测。


### 编译优化技术

即时编译器优化技术：

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/jvm/jvm_jit_compiler_technique.png)