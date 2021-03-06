虚拟机执行子系统

## 类文件结构

### 语言无关性
各种不同平台的虚拟机与所有平台都统一使用的程序存储格式——字节码（ByteCode）是构成平台无关性的基石。

商业机构和开源机构已经在Java语言之外发展出一大批在Java虚拟机之上运行的语言。

实现语言无关性的基础是虚拟机和字节码存储格式。Java虚拟机**不和包括Java在内的任何语言绑定**，它只与“Class文件”这种特定的二进制文件格式所关联，Class文件中包含了Java虚拟机指令集和符号表以及若干其他辅助信息。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/jvm/jvm_jvm_multi_language.png)


### Class类文件结构

任何一个Class文件都对应着唯一一个类或接口的定义信息，但反过来说，类或接口并不一定都得定义在文件里（譬如类或接口也可以通过类加载器直接生成）。

Class文件是一组以8位字节为基础单位的二进制流。

根据Java虚拟机规范的规定，Class文件格式采用一种类似于C语言结构体的伪结构来存储数据，这种伪结构中只有两种数据类型：**无符号数和表**。

- 无符号数属于基本的数据类型，以u1、u2、u4、u8来分别代表1个字节、2个字节、4个字节和8个字节的无符号数，无符号数可以用来描述数字、索引引用、数量值或者按照UTF-8编码构成字符串值。
- 表是由多个无符号数或者其他表作为数据项构成的复合数据类型，所有表都习惯性地以“_info”结尾。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/jvm/jvm_class_file_structure.png)



#### 1.魔数与Class文件的版本

每个Class文件的头4个字节称为魔数（Magic Number），它的唯一作用是确定这个文件是否为一个能被虚拟机接受的Class文件。

紧接着魔数的4个字节存储的是Class文件的版本号：第5和第6个字节是次版本号（MinorVersion），第7和第8个字节是主版本号（Major Version）。

如下所示，开头4个字节的十六进制表示是魔数0xCAFEBABE，代表次版本号的第5个和第6个字节值为0x0000，而主版本号的值为0x0032，也即是十进制的50，该版本号说明这个文件是可以被JDK 1.6或以上版本虚拟机执行的Class文件。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/jvm/jvm_class_example.png)


#### 2.常量池

常量池中主要存放两大类常量：字面量（Literal）和符号引用（Symbolic References）。

- 字面量比较接近于Java语言层面的常量概念，如文本字符串、声明为final的常量值等。
- 而符号引用则属于编译原理方面的概念，包括了下面三类常量：
	- 类和接口的全限定名（Fully Qualified Name）
	- 字段的名称和描述符（Descriptor）
	- 方法的名称和描述符

	
在Class文件中不会保存各个方法、字段的最终内存布局信息，因此这些字段、方法的符号引用不经过运行期转换的话无法得到真正的内存入口地址，也就无法直接被虚拟机使用。当虚拟机运行时，需要从常量池获得对应的符号引用，再在类创建时或运行时解析、翻译到具体的内存地址之中。

常量池项目类型：

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/jvm/jvm_class_constant_pool.png)


#### 3.访问标志

用于识别一些类或者接口层次的访问信息。包括：这个Class是类还是接口；是否定义为public类型；是否定义为abstract类型；如果是类的话，是否被声明为final等。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/jvm/jvm_class_access_flags.png)


#### 4.类索引、父类索引与接口索引集合

Class文件中由这三项数据来确定这个类的继承关系。类索引用于确定这个类的全限定名，父类索引用于确定这个类的父类的全限定名，接口索引集合就用来描述这个类实现了哪些接口。


#### 5.字段表集合

描述接口或者类中声明的变量。字段（field）包括类级变量以及实例级变量，但不包括在方法内部声明的局部变量。

字段表结构：

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/jvm/jvm_class_field_info.png)

字段访问标识：

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/jvm/jvm_class_field_access_flags.png)

字段表集合中不会列出从超类或者父接口中继承而来的字段，但有可能列出原本Java代码之中不存在的字段，譬如在内部类中为了保持对外部类的访问性，会自动添加指向外部类实例的字段。


#### 6.方法表集合

包含对方法的描述。而方法里的Java代码，经过编译器编译成字节码指令后，存放在方法属性表集合中一个名为“Code”的属性里面，属性表作为Class文件格式中最具扩展性的一种数据项目

方法访问标识：

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/jvm/jvm_class_method_access_flags.png)



#### 7.属性表集合


虚拟机规范预定义的属性：

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/jvm/jvm_class_attribute_info.png)



**ConstantValue属性：**
ConstantValue属性的作用是通知虚拟机自动为静态变量赋值。只有被static关键字修饰的变量（类变量）才可以使用这项属性。

类似“int x=123”和“static int x=123”这样的变量定义在Java程序中是非常常见的事情，但虚拟机对这两种变量赋值的方式和时刻都有所不同。

- 对于非static类型的变量（也就是实例变量）的赋值是在实例构造器<init>方法中进行的；
- 而对于类变量，则有两种方式可以选择：在类构造器<clinit>方法中或者使用ConstantValue属性。目前Sun Javac编译器的选择是：如果同时使用final和static来修饰一个变量（按照习惯，这里称“常量”更贴切），并且这个变量的数据类型是基本类型或者java.lang.String的话，就生成ConstantValue属性来进行初始化，如果这个变量没有被final修饰，或者并非基本类型及字符串，则将会选择在<clinit>方法中进行初始化。

<br/>

### 字节码指令

Java虚拟机的指令由一个字节长度的、代表着某种特定操作含义的数字（称为**操作码**，Opcode）以及跟随其后的零至多个代表此操作所需参数（称为**操作数**，Operands）而构成。

由于Java虚拟机采用**面向操作数栈**而不是寄存器的架构（这两种架构的区别和影响将在第8章中探讨），所以大多数的指令都不包含操作数，只有一个操作码。

#### 字节码与数据类型

在Java虚拟机的指令集中，大多数的指令都包含了其操作所对应的数据类型信息。例如，iload指令用于从局部变量表中加载int型的数据到操作数栈中，而fload指令加载的则是float类型的数据。

i代表对int类型的数据操作，l代表long，s代表short，b代表byte，c代表char，f代表float，d代表double，a代表reference。

大部分的指令都没有支持整数类型byte、char和short，甚至没有任何指令支持boolean类型。大多数对于boolean、byte、short和char类型数据的操作，实际上都是使用相应的int类型作为运算类型（Computational Type）


#### 加载和存储指令

用于将数据在栈帧中的局部变量表和操作数栈之间来回传输。

- 将一个局部变量加载到操作栈：iload、iload\_\<n>、lload、lload\_\<n>、fload、fload\_\<n>、dload、dload\_\<n>、aload、aload_\<n>。
- 将一个数值从操作数栈存储到局部变量表：istore、istore\_\<n>、lstore、lstore\_\<n>、fstore、fstore\_\<n>、dstore、dstore\_\<n>、astore、astore\_\<n>。
- 将一个常量加载到操作数栈：bipush、sipush、ldc、ldc_w、ldc2\_w、aconst\_null、iconst\_m1、iconst\_\<i>、lconst\_\<l>、fconst\_\<f>、dconst\_\<d>。
- 扩充局部变量表的访问索引的指令：wide。



<br/>

#### 运算指令

运算或算术指令用于对两个操作数栈上的值进行某种特定运算，并把结果重新存入到操作栈顶。

所有的算术指令如下：

- 加法指令：iadd、ladd、fadd、dadd。
- 减法指令：isub、lsub、fsub、dsub。
- 乘法指令：imul、lmul、fmul、dmul。
- 除法指令：idiv、ldiv、fdiv、ddiv。
- 求余指令：irem、lrem、frem、drem。
- 取反指令：ineg、lneg、fneg、dneg。
- 位移指令：ishl、ishr、iushr、lshl、lshr、lushr。
- 按位或指令：ior、lor。
- 按位与指令：iand、land。
- 按位异或指令：ixor、lxor。
- 局部变量自增指令：iinc。
- 比较指令：dcmpg、dcmpl、fcmpg、fcmpl、lcmp。


<br/>

#### 类型转换指令

类型转换指令可以将两种不同的数值类型进行相互转换。

Java虚拟机直接支持（即转换时无需显式的转换指令）以下数值类型的**宽化类型转换**（WideningNumeric Conversions，即小范围类型向大范围类型的安全转换）：

- int类型到long、float或者double类型。
- long类型到float、double类型。
- float类型到double类型。

相对的，处理**窄化类型转换**（Narrowing Numeric Conversions）时，必须显式地使用转换指令来完成，这些转换指令包括：i2b、i2c、i2s、l2i、f2i、f2l、d2i、d2l和d2f。窄化类型转换可能会导致转换结果产生不同的正负号、不同的数量级的情况，转换过程很可能会导致数值的精度丢失。


<br/>

#### 对象创建与访问指令

- 创建类实例的指令：new。
- 创建数组的指令：newarray、anewarray、multianewarray。
- 访问类字段（static字段，或者称为类变量）和实例字段（非static字段，或者称为实例变量）的指令：getfield、putfield、getstatic、putstatic。
- 把一个数组元素加载到操作数栈的指令：baload、caload、saload、iaload、laload、faload、daload、aaload。
- 将一个操作数栈的值存储到数组元素中的指令：bastore、castore、sastore、iastore、fastore、dastore、aastore。
- 取数组长度的指令：arraylength。
- 检查类实例类型的指令：instanceof、checkcast。

<br/>

#### 操作数栈管理指令

- 将操作数栈的栈顶一个或两个元素出栈：pop、pop2。
- 复制栈顶一个或两个数值并将复制值或双份的复制值重新压入栈顶：dup、dup2、dup_x1、dup2_x1、dup_x2、dup2_x2。
- 将栈最顶端的两个数值互换：swap。

<br/>

#### 控制转移指令

让Java虚拟机有条件或无条件地从指定的位置指令而不是控制转移指令的下一条指令继续执行程序，从概念模型上理解，可以认为控制转移指令就是在有条件或无条件地修改PC寄存器的值。

- 件分支：ifeq、iflt、ifle、ifne、ifgt、ifge、ifnull、ifnonnull、if_icmpeq、if_icmpne、if_icmplt、if_icmpgt、if_icmple、if_icmpge、if_acmpeq和if_acmpne。
- 复合条件分支：tableswitch、lookupswitch。
- 无条件分支：goto、goto_w、jsr、jsr_w、ret。


<br/>

#### 方法调用和返回指令

- invokevirtual指令用于调用对象的实例方法，根据对象的实际类型进行分派（虚方法分派），这也是Java语言中最常见的方法分派方式。
- invokeinterface指令用于调用接口方法，它会在运行时搜索一个实现了这个接口方法的对象，找出适合的方法进行调用。
- invokespecial指令用于调用一些需要特殊处理的实例方法，包括实例初始化方法、私有方法和父类方法。
- invokestatic指令用于调用类方法（static方法）。
- invokedynamic指令用于在运行时动态解析出调用点限定符所引用的方法，并执行该方法，前面4条调用指令的分派逻辑都固化在Java虚拟机内部，而invokedynamic指令的分派逻辑是由用户所设定的引导方法决定的。

<br/>

#### 异常处理指令

在Java程序中**显式抛出异常**的操作（throw语句）都由athrow指令来实现，除了用throw语句显式抛出异常情况之外，Java虚拟机规范还规定了许多运行时异常会在其他Java虚拟机指令检测到异常状况时自动抛出。

而**处理异常**，采用异常表来完成。

<br>

#### 同步指令

方法级的同步和方法内部一段指令序列的同步，这两种同步结构都是使用管程（Monitor）来支持的。

**方法级的同步**，是隐式的，即无须通过字节码指令来控制，它实现在方法调用和返回操作之中。虚拟机可以从方法常量池的方法表结构中的ACC_SYNCHRONIZED访问标志得知一个方法是否声明为同步方法。

**同步一段指令集序列**，通常是由Java语言中的synchronized语句块来表示的，Java虚拟机的指令集中有monitorenter和monitorexit两条指令来支持synchronized关键字的语义。




## 虚拟机类加载机制

虚拟机把描述类的数据从Class文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这就是虚拟机的类加载机制。

与那些在编译时需要进行连接工作的语言不同，在Java语言里面，类型的加载、连接和初始化过程都是在程序运行期间完成的，这种策略虽然会令类加载时稍微增加一些性能开销，但是会为Java应用程序提供高度的灵活性，Java里天生可以动态扩展的语言特性就是依赖**运行期动态加载和动态连接**这个特点实现的。

<br/>

### 类加载的时机

类的生命周期：

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/jvm/jvm_class_life_cycle.png)

加载、验证、准备、初始化和卸载这5个阶段的顺序是确定的，类的加载过程必须按照这种顺序按部就班地开始，而解析阶段则不一定：它在某些情况下可以在初始化阶段之后再开始，这是为了支持Java语言的运行时绑定（也称为动态绑定或晚期绑定）。

#### 初始化

对于**初始化阶段**，虚拟机规范严格规定了有且只有5种情况必须立即对类进行“初始化”（而加载、验证、准备自然需要在此之前开始）。

- 遇到**new、getstatic、putstatic或invokestatic**这4条字节码指令时，如果类没有进行过初始化，则需要先触发其初始化。生成这4条指令的最常见的Java代码场景是：使用new关键字实例化对象的时候、读取或设置一个类的静态字段（被final修饰、已在编译期把结果放入常量池的静态字段除外）的时候，以及调用一个类的静态方法的时候。
- 使用java.lang.reflect包的方法对类进行**反射调用**的时候，如果类没有进行过初始化，则需要先触发其初始化。
- 当初始化一个类的时候，如果发现其**父类**还没有进行过初始化，则需要先触发其父类的初始化。
- 当虚拟机启动时，用户需要指定一个**要执行的主类**（包含main()方法的那个类），虚拟机会先初始化这个主类。
- 当使用JDK 1.7的动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果REF_getStatic、REF_putStatic、REF_invokeStatic的方法句柄，并且这个方法句柄所对应的类没有进行过初始化，则需要先触发其初始化。

以上这5种场景中的行为称为对一个类进行主动引用。除此之外，所有引用类的方式都不会触发初始化，称为被动引用。

对于静态字段，只有直接定义这个字段的类才会被初始化，因此通过其子类来引用父类中定义的静态字段，只会触发父类的初始化而不会触发子类的初始化。



<br/>

### 类加载的过程

#### 加载

在加载阶段，虚拟机需要完成以下3件事情：

- 通过一个类的全限定名来获取定义此类的二进制字节流。
- 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
- 在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口。


“通过一个类的全限定名来获取定义此类的二进制字节流”这条，它没有指明二进制字节流要从一个Class文件中获取，准确地说是根本没有指明要从哪里获取、怎样获取。


加载阶段完成后，虚拟机外部的二进制字节流就按照虚拟机所需的格式存储在方法区之中，然后在内存中实例化一个java.lang.Class类的对象，这个对象将作为程序访问方法区中的这些类型数据的外部接口。


#### 验证

目的是为了确保Class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。

包括：

- 文件格式验证
- 元数据验证
- 字节码验证
- 符号引用验证

#### 准备

准备阶段是正式为类变量分配内存并设置类变量初始值的阶段，这些变量所使用的内存都将在方法区中进行分配。

- 首先，这时候进行内存分配的仅包括类变量（被static修饰的变量），而不包括实例变量，实例变量将会在对象实例化时随着对象一起分配在Java堆中。
- 其次，这里所说的初始值“通常情况”下是数据类型的零值。因为这时候尚未开始执行任何Java方法，而把value赋值为真实值的putstatic指令是程序被编译后，存放于类构造器<clinit>()方法之中。

在“通常情况”下初始值是零值，那相对的会有一些“特殊情况”：如果类字段的字段属性表中存在ConstantValue属性，那在准备阶段变量value就会被初始化为ConstantValue属性所指定的值。



#### 解析

解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程。

- 符号引用（Symbolic References）：符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可。符号引用与虚拟机实现的内存布局无关，引用的目标并不一定已经加载到内存中。
- 直接引用（Direct References）：直接引用可以是直接指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄。直接引用是和虚拟机实现的内存布局相关的，同一个符号引用在不同虚拟机实例上翻译出来的直接引用一般不会相同。如果有了直接引用，那引用的目标必定已经在内存中存在。

对同一个符号引用进行多次解析请求是很常见的事情，除invokedynamic指令以外，虚拟机实现可以对第一次解析的结果进行缓存。

对于invokedynamic指令，必须等到程序实际运行到这条指令的时候，解析动作才能进行。


1. 类或接口的解析
2. 字段解析 
3. 类方法解析
4. 接口方法解析


#### 初始化

前面的类加载过程中，除了在加载阶段用户应用程序可以通过自定义类加载器参与之外，其余动作完全由虚拟机主导和控制。到了初始化阶段，才真正开始执行类中定义的Java程序代码（或者说是字节码）。

初始化阶段是执行**类构造器<clinit>()方法**的过程。


<clinit>()方法：

- <clinit>()方法是由**编译器自动收集**类中的所有类变量的赋值动作和静态语句块（static{}块）中的语句合并产生的，编译器收集的顺序是由语句在源文件中出现的顺序所决定的，静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句块可以赋值，但是不能访问。
- <clinit>()方法与类的构造函数（或者说实例构造器<init>()方法）不同，它不需要显式地调用父类构造器，虚拟机会保证**在子类的<clinit>()方法执行之前，父类的<clinit>()方法已经执行完毕**。因此在虚拟机中第一个被执行的<clinit>()方法的类肯定是java.lang.Object。
- 由于父类的<clinit>()方法先执行，也就意味着**父类中定义的静态语句块要优先于子类的变量赋值操作**。
- <clinit>()方法对于类或接口来说并不是必需的，如果一个类中没有静态语句块，也没有对变量的赋值操作，那么编译器可以不为这个类生成<clinit>()方法。
- 接口中不能使用静态语句块，但仍然有变量初始化的赋值操作，因此接口与类一样都会生成<clinit>()方法。但接口与类不同的是，执行接口的<clinit>()方法不需要先执行父接口的<clinit>()方法。只有当父接口中定义的变量使用时，父接口才会初始化。另外，接口的实现类在初始化时也一样不会执行接口的<clinit>()方法。
- 虚拟机会保证一个类的**<clinit>()方法在多线程环境中被正确地加锁、同步**，如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的<clinit>()方法，其他线程都需要阻塞等待，直到活动线程执行<clinit>()方法完毕。如果在一个类的<clinit>()方法中有耗时很长的操作，就可能造成多个进程阻塞[插图]，在实际应用中这种阻塞往往是很隐蔽的	

<br/>

### 类加载器

类加载器通过一个类的全限定名来获取描述此类的二进制字节流。

类加载器虽然只用于实现类的加载动作，但它在Java程序中起到的作用却远远不限于类加载阶段。对于任意一个类，都需要由加载它的类加载器和这个类本身一同确立其在Java虚拟机中的唯一性，**每一个类加载器，都拥有一个独立的类名称空间**。比较两个类是否“相等”，只有在这两个类是由同一个类加载器加载的前提下才有意义，否则，即使这两个类来源于同一个Class文件，被同一个虚拟机加载，只要加载它们的类加载器不同，那这两个类就必定不相等。


1. **启动类加载器（Bootstrap ClassLoader）**。这个类加载器负责将存放在<JAVA_HOME>\lib目录中的，或者被-Xbootclasspath参数所指定的路径中的，并且是虚拟机识别的（仅按照文件名识别，如rt.jar，名字不符合的类库即使放在lib目录中也不会被加载）类库加载到虚拟机内存中。
2. **扩展类加载器（Extension ClassLoader）**：这个加载器由sun.misc.Launcher$ExtClassLoader实现，它负责加载<JAVA_HOME>\lib\ext目录中的，或者被java.ext.dirs系统变量所指定的路径中的所有类库，开发者可以直接使用扩展类加载器。
3. **应用程序类加载器（Application ClassLoader）**：这个类加载器由sun.misc.Launcher$App-ClassLoader实现。由于这个类加载器是ClassLoader中的getSystemClassLoader()方法的返回值，所以一般也称它为系统类加载器。它负责加载用户类路径（ClassPath）上所指定的类库，开发者可以直接使用这个类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

#### 类加载器双亲委派模型

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/jvm/jvm_classloader_parent_delegation_model.png)

双亲委派模型要求除了顶层的启动类加载器外，其余的类加载器都应当有自己的父类加载器。

双亲委派模型的工作过程是：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去加载。

使用双亲委派模型来组织类加载器之间的关系，有一个显而易见的好处就是Java类随着它的类加载器一起具备了一种带有优先级的层次关系。

例如类java.lang.Object，它存放在rt.jar之中，无论哪一个类加载器要加载这个类，最终都是委派给处于模型最顶端的启动类加载器进行加载，因此Object类在程序的各种类加载器环境中都是同一个类。

#### 破坏双亲委派模型

双亲委派模型并不是一个强制性的约束模型，而是Java设计者推荐给开发者的类加载器实现方式。

破坏双亲委派模型的情况：

1. 在双亲委派模型出现之前——即JDK 1.2发布之前。
2. 基础类要调用回用户的代码的情况，如JNDI服务、JDBC等。
	- 一个典型的例子便是JNDI服务，JNDI现在已经是Java的标准服务，它的代码由启动类加载器去加载（在JDK 1.3时放进去的rt.jar），但JNDI的目的就是对资源进行集中管理和查找，它需要调用由独立厂商实现并部署在应用程序的ClassPath下的JNDI接口提供者（SPI，Service Provider Interface）的代码，但启动类加载器不可能“认识”这些代码啊！那该怎么办？
	- 因此引入**线程上下文类加载器（ThreadContext ClassLoader**），这个类加载器可以通过java.lang.Thread类的setContextClassLoaser()方法进行设置，如果创建线程时还未设置，它将会从父线程中继承一个，如果在应用程序的全局范围内都没有设置过的话，那这个类加载器默认就是应用程序类加载器。
3. 用户对程序动态性的追求而导致的破坏。
	- 这里动态性指代码热替换（HotSwap）、模块热部署（Hot Deployment）等。
	- OSGi已经成为了业界“事实上”的Java模块化标准，而OSGi实现模块化热部署的关键则是它自定义的类加载器机制的实现。每一个程序模块（OSGi中称为Bundle）都有一个自己的类加载器，当需要更换一个Bundle时，就把Bundle连同类加载器一起换掉以实现代码的热替换。

	
## 虚拟机字节码执行引擎

### 运行时栈帧结构

栈帧（Stack Frame）是用于**支持虚拟机进行方法调用和方法执行**的数据结构，它是虚拟机运行时数据区中的虚拟机栈（Virtual Machine Stack）的栈元素。栈帧存储了方法的局部变量表、操作数栈、动态连接和方法返回地址等信息。每一个方法从调用开始至执行完成的过程，都对应着一个栈帧在虚拟机栈里面从入栈到出栈的过程。

栈帧结构：

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/jvm/jvm_stack_frame_architecture.png)

在编译程序代码的时候，栈帧中需要多大的局部变量表，多深的操作数栈都已经完全确定了，并且写入到方法表的Code属性之中[插图]，因此一个栈帧需要分配多少内存，不会受到程序运行期变量数据的影响，而仅仅取决于具体的虚拟机实现。

一个线程中的方法调用链可能会很长，很多方法都同时处于执行状态。对于执行引擎来说，在活动线程中，只有位于栈顶的栈帧才是有效的，称为**当前栈帧**（Current Stack Frame），与这个栈帧相关联的方法称为当前方法（Current Method）。执行引擎运行的所有字节码指令都只针对当前栈帧进行操作。

#### 1. 局部变量表

局部变量表（Local Variable Table）是一组变量值存储空间，用于**存放方法参数和方法内部定义的局部变量**。在Java程序编译为Class文件时，就在方法的Code属性的max_locals数据项中确定了该方法所需要分配的局部变量表的最大容量。

局部变量表的容量以变量槽（Variable Slot，下称Slot）为最小单位。

虚拟机通过索引定位的方式使用局部变量表。

在方法执行时，虚拟机是使用局部变量表完成参数值到参数变量列表的传递过程的，如果执行的是实例方法（非static的方法），那局部变量表中第0位索引的Slot默认是用于**传递方法所属对象实例的引用**，在方法中可以通过关键字“this”来访问到这个隐含的参数。其余参数则按照参数表顺序排列，占用从1开始的局部变量Slot，参数表分配完毕后，再根据方法体内部定义的变量顺序和作用域分配其余的Slot。

可重用：

- 为了尽可能节省栈帧空间，局部变量表中的**Slot是可以重用的**，方法体中定义的变量，其作用域并不一定会覆盖整个方法体，如果当前字节码PC计数器的值已经超出了某个变量的作用域，那这个变量对应的Slot就可以交给其他变量使用。


局部变量表无默认初始化：

- 即使在初始化阶段程序员没有为类变量赋值也没有关系，类变量仍然具有一个确定的初始值。但局部变量就不一样，**如果一个局部变量定义了但没有赋初始值是不能使用的**，不要认为Java中任何情况下都存在诸如整型变量默认为0，布尔型变量默认为false等这样的默认值。
	
	
#### 2. 操作数栈

操作数栈（Operand Stack）也常称为操作栈，它是一个后入先出（Last In First Out, LIFO）栈。

当一个方法刚刚开始执行的时候，这个方法的操作数栈是空的，在方法的执行过程中，会有各种字节码指令往操作数栈中写入和提取内容，也就是出栈/入栈操作。例如，在做算术运算的时候是通过操作数栈来进行的，又或者在调用其他方法的时候是通过操作数栈来进行参数传递的。

操作数栈中元素的数据类型必须与字节码指令的序列严格匹配，在编译程序代码的时候，编译器要严格保证这一点，在类校验阶段的数据流分析中还要再次验证这一点。

栈帧重叠：

- 在大多虚拟机的实现里都会做一些优化处理，令两个栈帧出现一部分重叠。让下面栈帧的部分操作数栈与上面栈帧的部分局部变量表重叠在一起，这样在进行方法调用时就可以共用一部分数据，无须进行额外的参数复制传递	


#### 3. 动态连接

每个栈帧都包含一个指向运行时常量池[插图]中该栈帧所属方法的引用，持有这个引用是为了支持方法调用过程中的动态连接（Dynamic Linking）。

Class文件的常量池中存有大量的符号引用，字节码中的方法调用指令就以常量池中指向方法的符号引用作为参数。这些符号引用一部分会在类加载阶段或者第一次使用的时候就转化为直接引用，这种转化称为**静态解析**。另外一部分将在每一次运行期间转化为直接引用，这部分称为**动态连接**。


#### 4.方法返回地址

当一个方法开始执行后，只有两种方式可以退出这个方法：

- 执行引擎遇到任意一个方法返回的字节码指令，称为正常完成出口。
- 在方法执行过程中遇到了异常，并且这个异常没有在方法体内得到处理，导致方法退出，称为异常完成出口。一个方法使用异常完成出口的方式退出，是不会给它的上层调用者产生任何返回值的。


无论采用何种退出方式，在方法退出之后，都需要返回到方法被调用的位置，程序才能继续执行，方法返回时可能需要在栈帧中保存一些信息，用来帮助恢复它的上层方法的执行状态。一般来说，方法正常退出时，调用者的PC计数器的值可以作为返回地址，栈帧中很可能会保存这个计数器值。而方法异常退出时，返回地址是要通过异常处理器表来确定的，栈帧中一般不会保存这部分信息。

方法退出的过程实际上就等同于把当前栈帧出栈，因此**退出时可能执行的操作**有：恢复上层方法的局部变量表和操作数栈，把返回值（如果有的话）压入调用者栈帧的操作数栈中，调整PC计数器的值以指向方法调用指令后面的一条指令等。	


<br/>

### 方法调用

方法调用并不等同于方法执行，方法调用阶段唯一的任务就是**确定被调用方法的版本**（即调用哪一个方法），暂时还不涉及方法内部的具体运行过程。

Class文件的编译过程中不包含传统编译中的连接步骤，一切方法调用在Class文件里面存储的都只是符号引用，而不是方法在实际运行时内存布局中的入口地址，需要在类加载期间，甚至到运行期间才能确定目标方法的直接引用。


#### 解析

所有方法调用中的目标方法在Class文件里面都是一个常量池中的符号引用，在类加载的解析阶段，会将其中的一部分**符号引用转化为直接引用**，这种解析能成立的前提是：方法在程序真正运行之前就有一个可确定的调用版本，并且这个方法的调用版本在运行期是不可改变的。

在Java语言中符合“**编译期可知，运行期不可变**”这个要求的方法，主要包括静态方法和私有方法两大类。

只要能被invokestatic和invokespecial指令调用的方法，都可以在解析阶段中确定唯一的调用版本，符合这个条件的有静态方法、私有方法、实例构造器、父类方法4类，它们在类加载的时候就会把符号引用解析为该方法的直接引用。这些方法可以称为**非虚方法**，与之相反，其他方法称为**虚方法**。final方法也是一种非虚方法。

- 解析调用一定是个静态的过程，在编译期间就完全确定，在**类装载的解析阶段**就会把涉及的符号引用全部转变为可确定的直接引用，不会延迟到运行期再去完成。
- 分派（Dispatch）调用则可能是静态的也可能是动态的，根据分派依据的宗量数可分为单分派和多分派。

#### 分派

分派分为静态单分派、静态多分派、动态单分派、动态多分派。

1. 静态分派

静态分派的例子overload：

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/jvm/jvm_static_dispatch_example.png)

上面代码中的“Human”称为变量的**静态类型**（Static Type），或者叫做的外观类型（Apparent Type），后面的“Man”则称为变量的**实际类型**（Actual Typ）。静态类型和实际类型在程序中都可以发生一些变化，区别是静态类型的变化仅仅在使用时发生，变量本身的静态类型不会被改变，并且最终的静态类型是在编译期可知的；而实际类型变化的结果在运行期才可确定。

虚拟机（准确地说是编译器）在重载时是**通过参数的静态类型**而不是实际类型作为判定依据的。并且静态类型是编译期可知的，因此，在编译阶段，Javac编译器会根据参数的静态类型决定使用哪个重载版本，所以选择了sayHello(Human)作为调用目标，并把这个方法的符号引用写到main()方法里的两条invokevirtual指令的参数中。

运行结果：
```
hello,guy!
hello,guy!
```

所有依赖静态类型来定位方法执行版本的分派动作称为静态分派。静态分派的典型应用是方法重载。**静态分派发生在编译阶段**，因此确定静态分派的动作实际上不是由虚拟机来执行的。

编译器虽然能确定出方法的重载版本，但在很多情况下这个重载版本并不是“唯一的”，往往只能确定一个“更加合适的”版本。

解析与分派这两者之间的关系并不是二选一的排他关系，它们是在不同层次上去筛选、确定目标方法的过程。例如，静态方法会在类加载期就进行解析，而静态方法显然也是可以拥有重载版本的，选择重载版本的过程也是通过静态分派完成的。

2. 动态分派。在运行期根据实际类型确定方法执行版本的分派过程称为动态分派。

动态分派的例子重写Override：

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/jvm/jvm_dynamic_dispatch_example.png)

执行过程中，首先将调用对象（man/woman对象）放入操作数栈，然后使用invokevirtual指令执行。invokevirtual指令的运行时解析过程大致分为以下几个步骤：

- 找到操作数栈顶的第一个元素所指向的对象的实际类型，记作C。
- 如果在类型C中找到与常量中的描述符和简单名称都相符的方法，则进行访问权限校验，如果通过则返回这个方法的直接引用，查找过程结束；如果不通过，则返回java.lang. IllegalAccessError异常。
- 否则，按照继承关系从下往上依次对C的各个父类进行第2步的搜索和验证过程。
- 如果始终没有找到合适的方法，则抛出java.lang.AbstractMethodError异常。

3. 单分派与多分派

方法的接收者与方法的参数统称为方法的**宗量**。

据分派基于多少种宗量，可以将分派划分为单分派和多分派两种。单分派是根据一个宗量对目标方法进行选择，多分派则是根据多于一个宗量对目标方法进行选择。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/jvm/jvm_dispatch_example.png)

编译阶段编译器的选择过程，也就是静态分派的过程。这时选择目标方法的依据有两点：一是**静态类型**是Father还是Son，二是**方法参数**是QQ还是360。这次选择结果的最终产物是产生了两条invokevirtual指令，两条指令的参数分别为常量池中指向Father. hardChoice(360)及Father.hardChoice(QQ)方法的符号引用。因为是根据两个宗量进行选择，所以Java语言的静态分派属于多分派类型。

运行阶段虚拟机的选择，也就是动态分派的过程。在执行“son.hardChoice(new QQ())”这句代码时，更准确地说，是在执行这句代码所对应的invokevirtual指令时，由于编译期已经决定目标方法的签名必须为hardChoice(QQ)，虚拟机此时不会关心传递过来的参数“QQ”到底是“腾讯QQ”还是“奇瑞QQ”，因为这时参数的静态类型、实际类型都对方法的选择不会构成任何影响，唯一可以影响虚拟机选择的因素只有**此方法的接受者的实际类型**是Father还是Son。因为只有一个宗量作为选择依据，所以Java语言的动态分派属于单分派类型。

因此，Java是一个**静态多分派、动态单分派**的语言。

<br/>

**虚拟机动态分派的实现：**

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/jvm/jvm_method_table.png)

虚拟机为类在方法区中建立一个**虚方法表**（Vritual Method Table，也称为vtable，与此对应的，在invokeinterface执行时也会用到接口方法表——Inteface Method Table，简称itable），使用虚方法表索引来代替元数据查找以提高性能。

虚方法表中存放着各个方法的**实际入口地址**。如果某个方法在子类中没有被重写，那子类的虚方法表里面的地址入口和父类相同方法的地址入口是一致的，都指向父类的实现入口。如果子类中重写了这个方法，子类方法表中的地址将会替换为指向子类实现版本的入口地址。

方法表一般在类加载的连接阶段进行初始化，准备了类的变量初始值后，虚拟机会把该类的方法表也初始化完毕。


<br/>

### 动态语言支持

