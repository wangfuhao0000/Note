# 早期优化

Java语言的“编译期”其实是一段“不确定”的操作过程，它可能包含三类：

- **前端编译器**：把*.java文件转变为*.class文件的过程，如Sun的Javac、Eclipse JDT中的增量式编译器（ECJ）。
- **后端运行期编译器（JIT）**：把字节码转变为机器码，如HotSpot VM的C1、C2编译器。
- **静态提前编译器（AOT）**：直接把*.java文件编译为本地机器代码的过程，如GNU Compiler for the Java（GCJ）、Excelsior JET。

当然我们可能认知较清楚的是第一种，我们这次也是主要介绍它。Javac这类编译器对代码的运行效率几乎没有任何优化措施，虚拟机设计团队把优化的性能集中到了后端的即时编译器中，这样可以让那些不是由Javac产生的Class文件也同样能享受到编译器优化所带来的的好处。但前端编译器优化主要会对**Java程序编码**有一些优化，如一些语法糖等，而即时编译器优化对**程序的运行**更加重要。

------

### 1. Javac编译器

javac编译器本身就是一个由Java语言编写的程序，它的编译过程主要分为3个步骤：

1. 解析与填充符号表过程
2. 插入式注解处理器的注解处理过程
3. 分析与字节码生成过程

三者直接的关系和交互顺序如图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191117131419736.png)

###### 1.1 解析与填充符号表

------

解析步骤由parseFiles()方法完成，包括经典程序编译原理中的**词法分析**和**语法分析**。

**词法、语法分析**

**词法分析**是将源代码中的字符流转变为**标记（Token）集合**，单个字符是程序编写过程的最小元素，而标记则是编译过程的最小元素，关键字、变量名、字面量、运算符都可以成为标记。例如`int a=b+2`就包含int、a、=、b、+、2共六个标记，此过程是由com.sun.tools.java.parser.Scanner类实现。
**语法分析**是根据Token序列构造**抽象语法树**的过程，抽象语法树（AST）是一种用来描述程序代码语法结构的树形表示方式，语法树的每一个节点都代表着程序代码中的一个语法结构，例如包、类型、修饰符、运算符…都可以是一个语法结构。这个阶段产出的抽象语法树由com.sun.tools.javac.tree.JCTree类标识，经过这个步骤后，编译器基本就不会再对源码文件进行操作了，后续的操作都建立在抽象语法树上。

------

**填充符号表**

它是在完成了词法分析和语法分析之后要做的事。符号表是由一组**符号地址**和**符号信息**构成的表格，可以把它想象成哈希表中K-V值对的形式。符号表中所登记的信息在编译的不同阶段都要用到：在语义分析中，符号表所登记的内容将用于语义检查（如检查一个名字的使用和原先的说明是否一致）和产生中间代码；在目标代码生成阶段，当对符号名进行地址分配时，符号表是地址分配的依据。填充符号表的过程由com.sun.tools.javac.comp.Enter类实现，此过程的出口是一个待处理列表，包含了每一个编译单元的抽象语法树的顶级节点。

###### 1.2 注解处理器

JDK1.5中Java语言提供了对注解（Annotation）的支持，JDK1.6中提供了一组插入式注解处理器的标准API，它在编译期间对注解进行处理。可以把它们看作是一组编译器的插件，在这些插件里面可以读取、修改、添加抽象语法树中的任意元素。如果这些插件在处理注解期间对语法树进行了修改，编译器将回到解析及填充符号表的过程重新处理，直到所有插入式注解处理器都没有在对语法树进行修改为止，每一次循环称为一个Round，也就是最开始的图所展示的循环。

###### 1.3 语义分析与字节码生成

语法分析后我们能得到程序代码的抽象语法树表示，语法树能表示一个结构正确的源程序的抽象，但无法保证源程序是符合逻辑的。而语义分析的主要任务就是对结构上正确的源程序进行上下文有关性质的审查，如进行类型检查。比如说下面这段代码，三个赋值语句都能构成结构正确的语法树，但是只有第一种在语义上没问题，能通过编译，其余两种无法编译。

```java
int a = 1;
boolean b = false;
char c = 2;

int d = a + c;
int d = b + c;
char d = a + c;
1234567
```

**标注检查**

Javac编译过程中，语义分析过程分为**标注检查**以及**数据及控制流分析**两个步骤。标注检查步骤检查的内容包括诸如变量使用前是否已被声明、变量与赋值之间的数据类型是否能够匹配等。标注检查步骤中还有一个重要动作是**常量折叠**，比如语句`int a = 1 + 2;`，在生成语法树后能看到字面量“1”、“2”和操作符“+”，但是经过语义分析过程的常量折叠后就会在语法树上变为3，所以代码中定义`a = 1+2`和`a = 3`没什么不同，不会增加指令的运算量。

**数据及控制流分析**
数据及控制流分析是对程序上下文逻辑更进一步的验证，它可以检查出诸如程序局部变量在使用前是否有赋值、方法的每条路径是否都有返回值、是否所有的受查异常都能被正确处理等问题。编译时期的数据及控制流分析与类加载时的数据及控制流分析的目的基本上是一致的，只是校验范围可能有所区别。有一些校验项可能只在编译期或运行期才能进行。看一下下面的例子：

```java
public void foo(final int arg) {
	final int var = 0;
	...
}

public void foo(int arg) {
	int var = 0;
	...
}
123456789
```

这两个方法中第一个方法的参数和局部变量都使用了final修饰符，第二种没有，但是它们编译出来的Class文件没有任何区别。我们知道局部变量在常量池中没有CONSTAN_Fieldref_info的符号引用，也就没有访问标志等信息，自然在Class文件中不会知道一个变量是不是声明为final的。因此，将局部变量声明为final，对运行期没有任何影响，变量的不变形仅仅由编译器在编译期间保障，即编译期的语义检查部分会对final变量进行检查，看它是否更改过，如果是则不会通过编译。

**解语法糖**

语义分析过程中也会对Java中的语法糖进行解析，其实语法糖只是编码需要，Java虚拟机运行时不支持这些语法，它们在编译阶段还原回简单的基础语法结构，此过程称为解语法糖，下面会讲到这一部分。

**字节码生成**

字节码生成是Javac编译过程的最后一个阶段，在Javac源码里面由com.sun.tools.javac.jvm.Gen类完成。字节码生成阶段不仅仅是把前面各个步骤所生成的信息（语法树、符号表）转化成字节码写到磁盘中，编译器还进行了少量的代码添加和转化工作。

例如，实例构造器<init>()和类构造器<clinit>()方法就是在这个阶段添加到语法树之中的（注意此处的实例构造器不是默认的那个，默认的构造器添加过程已经在填充符号表阶段就完成了）。这两个构造器的产生过程实际上是一个代码收敛的过程，编译器会把**语句块**（实例构造器—“{}”块、类构造器—“static {}”块）、**变量初始化**（实例变量、类变量）、**调用父类的实例构造器**（仅仅是实例构造器，<clinit>()方法中无需调用父类的<clinit>()方法，虚拟机会保证）等操作收敛到<init>()和<clinit>()方法之中，并且保证一定是按先父类的实例构造器，然后初始化变量，最后执行语句块的顺序进行。除了生成构造器外，还有其他的一些代码替换工作用于优化程序的实现逻辑，如把字符串的家操作替换为StringBuffer或StringBuilder的append()操作等。

完成了对语法树的遍历和调整后，就会把填充了所有所需信息的符号表交给com.sun.tools.javac.jvm.ClassWriter类，由这个类的writeClass()方法输出字节码，生成最终的Class文件，然后编译阶段结束。

### 2. Java语法糖

###### 2.1 泛型与类型擦除

泛型是JDK1.5的一项新增特性，它的本质是参数化类型的应用，也就是说所操作的数据类型被指定为一个参数。泛型技术在C#和Java之中的使用方式类似，但实现上有根本性的分歧，C#里面泛型无论在程序源码中、编译后的IL（中间语言）中，或者是运行期的CLR中，都是切实存在的，List<int>与List<String>就是两个不同的类型，他们在系统运行期生成，有自己的虚方法表和类型数据，这种实现称为**类型膨胀**，基于这种方法实现的泛型称为真实泛型。

Java中的泛型则不一样，它只是在程序源码中存在，在编译后的字节码文件中，就已经替换为原来的原生类型了，并在相应的地方插入了强制转型代码。所以对于运行期的Java语言来说，ArrayList<int>与ArrayList<String>就是同一个类，所以泛型技术实际上是Java语言的一颗语法糖，Java语言中的泛型实现方法称为**类型擦除**，基于这种方法实现的泛型称为伪泛型。例如下面的代码：

```java
    public static void main(String[] args) {
        Map<String, String> map = new HashMap<>();
        map.put("hello", "你好");
        map.put("how are you?", "吃了没");
        System.out.println(map.get("hello"));
        System.out.println(map.get("how are you?"));
    }
1234567
```

当把这段代码编译为Class文件后，使用idea直接看时发现泛型都不见了，变为了原生类型：

```java
    public static void main(String[] args) {
        Map<String, String> map = new HashMap();	//这个地方不知道为什么还有
        map.put("hello", "你好");
        map.put("how are you?", "吃了没");
        System.out.println((String)map.get("hello"));
        System.out.println((String)map.get("how are you?"));
    }
1234567
```

但是Java泛型在某些地方是有一些不足的，比如下面的代码不能被编译，因为参数List<Integer>和List<Integer>编译之后都被擦除了，变成了一样的原生类型List<E>，擦除动作导致这两种方法的特征前面变得一模一样。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191130194502997.png)
但是看如下代码，两个方法参数没有变，但返回值改变了，它是“有可能”被编译的（和版本有关），因为方法重载要求方法具备不同的特征签名，返回值不包含在方法特征签名中，但是在Class文件格式中，只要描述符不是完全一致的两个方法就可以共存。就是说两个方法即使有相同的名称和特征签名，但返回值不同，也是可以合法地共存于一个Class文件中的。

```java
    public static String method(List<String> list) {
        System.out.println("invoke method(List<String> list)");
        return "";
    }

    public static int method(List<Integer> list) {
        System.out.println("invoke method(List<Integer> list)");
        return 0;
    }
123456789
```

###### 2.2 自动装箱、拆箱与遍历循环

这些语法糖和上面的泛型相比难度和深度都有较大差距，但是用到的很多。看一下下面的代码：

```java
    public static void main(String[] args) {
        List<Integer> list = Arrays.asList(1,2,3,4);
        int sum = 0;
        for (int i : list)
            sum += i;
        System.out.println(sum);
    }
1234567
```

编译后的字节码如下，但不知为什么没有看到自动装箱和自动拆箱对应的方法Integer.valueOf和Integer.intValue()，但是能看到遍历循环把代码还原成了迭代器的实现，这也是为何遍历循环需要被遍历的类实现Iterable接口的原因。

```java
    public static void main(String[] args) {
        List<Integer> list = Arrays.asList(1, 2, 3, 4);
        int sum = 0;

        int i;
        for(Iterator var3 = list.iterator(); var3.hasNext(); sum += i) {
            i = (Integer)var3.next();
        }

        System.out.println(sum);
    }
1234567891011
```

###### 2.3 条件编译

Java语言也可以进行**条件编译**，方法就是使用条件为常量的if语句。如下代码所示：

```java
public static void main(String[] args) {
	if (true) {
		System.out.println("block 1");
	} else {
		System.out.println("block 2");
	}
}
1234567
```

编译后的Class文件字节码如下：

```java
    public static void main(String[] args) {
        System.out.println("block 1");
    }
123
```

只能使用条件为常量的if语句才能达到上述效果，如果使用常量与其他带有条件判断能力的语句搭配，则可能在控制流分析中提示错误，被拒绝编译，比如下面的代码：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191130202736363.png)
Java语言中条件编译的实现，也是Java语言的一颗语法糖，根据布尔常量的真假，编译器将会把分支中不成立的代码块消除掉，这一工作将在编译器接触语法糖阶段完成。也正是使用了if语句来实现，因此只能实现语句基本块级别的条件编译，没有办法实现根据条件调整整个Java类的结构。

除了上述说的泛型、自动装箱、自动拆箱、遍历循环、编程参数和条件编译之外，Java还有很多其他的语法糖，如**内部类、枚举类、断言语句、对枚举和字符串的switch支持、try语句中定义和关闭资源**等，我们可通过跟踪Javac源码、反编译Class文件等方式了解他们的本质实现。

# 晚期优化

**1. 概述**

当虚拟机分析某个方法或代码块运行特别频繁时，会把这些代码认定为**“热点代码”**，为提高热点代码执行效率，运行时虚拟机会把这些代码编译成与本地平台相关的机器码，并进行优化，完成此任务的编译器称为**即时编译器**。

**2. HotSpot虚拟机内的即时编译器**

我们要解决以下问题：

- 

**2.1 解释器与编译器**

解释器与编译器两者各有优势：当程序需要迅速启动和执行时，解释器可先发挥作用，省去编译时间立即执行；程序运行后，随着时间推移编译器逐渐发挥作用，把越来越多的代码编译成本地代码后，可获得更高的执行效率。程序运行环境中内存资源限制较大，可使用解释执行节约内存，反正可使用编译执行来提升效率。

HotSpot虚拟机中内置了两个即时编译器：分别称为Client Compiler和Server Compiler，简称C1编译器和C2编译器（Opto编译器）。主流虚拟机中默认采用解释器和其中一个编译器直接配合的方式工作，选择哪个编译器取决于虚拟机运行的模式。这称为混合模式

即时编译器编译本地代码需要占用程序运行时间，为了在程序启动响应速度与运行效率之间达到最佳平衡，HotSpot虚拟机会逐渐启用**分层编译**。分层编译根据编译器编译、优化的规模与耗时，划分出不同的编译层次，其中包括：

- 第0层，程序解释执行，解释器不开启性能监控功能，可触发第1层编译
- 第1层，也称C1编译，将字节码编译为本地代码，进行简单、可靠的优化，若有必要将加入性能监控的逻辑
- 第2层，也称C2编译，也是将字节码编译为本地代码，但是会启用一些编译耗时较长的优化，甚至会根据性能监控信息进行一些不可靠的激进优化。

实施分层编译后，Client Compiler和Server Compiler会同时工作，许多代码都可能会被多次编译，用Client Compiler获取更高的编译速度，用Server Compiler来获取更好的编译质量，在解释执行时无须再承担收集性能监控信息的任务。

**2.2 编译对象与触发条件**

运行过程中被即时编译器编译的“热点代码”有两类，即：

- **被多次调用的****方法**
- **被多次执行的****循环体**

对第一种情况，由于是方法调用触发的编译，因此编译器会以整个方法作为编译对象，这种编译也是虚拟机中标准的JIT编译方式。而对于后一种情况，尽管编译动作是由循环体触发，但编译器任然会以整个方法（而不是单独的循环体）作为编译对象。这种编译方式因为编译发生在方法执行过程中，因此形象地称之为栈上替换（OSR编译，即方法栈帧还在栈上，方法就被替换了）。

判断一段代码是否是热点代码，是不是需要触发即时编译，此行为称为**热点探测**。目前热点探测判定方式有两种，分别如下：

- **基于****采样****的热点探测**：虚拟机周期性地检查各个线程的栈顶，若发现某个方法经常出现在栈顶，那此方法就是“热点方法”。它的好处是简单高效，很容易获取方法调用关系，缺点是很难精确一个方法的热点，易受到线程阻塞或别的外界因素影响而扰乱热点探测。
- **基于****计数器****的热点探测**：虚拟机会为每个方法建立计数器，统计方法的执行次数，若执行次数超过一定阈值就认为它是“热点方法”。这种方式实现麻烦，需为每个方法建立并维护计数器，且不能直接获取到方法调用关系，但统计结果更精确。

HotSpot中使用第二种，它为每个方法准备了两类计数器：方法调用计数器和回边计数器。他们都有一个确定的阈值，计数器超过阈值溢出了，就会触发JIT编译。

**方法调用计数器**

它用于统计方法被调用的次数，当一个方法被调用时，会检查该方法是否存在被JIT编译过的版本，若存在则优先使用编译后的本地代码执行；弱不存在则将此方法的调用计数器加1，然后判断方法调用计数器值是否超过了阈值，若超过则将向即时编译器提交一个该方法的代码编译请求。

若不做任何设置，执行引擎不会同步等待编译请求完成，而是继续进入解释器按照解释方式执行字节码，直到提交的请求被编译器编译完成。当编译工作完成后，此方法的调用入口地址就会被系统自动改写成新的，下次调用该方法时就会使用已编译的版本。

若不做任何设置，方法调用计数器统计的不是该方法被调用的绝对次数，而是个相对执行频率，即一段时间内方法被调用的次数。当超过一定时间限度，若方法的调用次数仍然不足以让它提交给即时编译器，那此方法的调用计数器就会被减少一半，此过程称为**方法调用计数器热度的衰减**，这段时间就称为此方法统计的**半衰周期**。进行热度衰减的动作是在虚拟机进行垃圾收集时顺便进行的。

![img](https://note.youdao.com/yws/public/resource/b4ef39bbc01d43d64e41afc9dabd4843/xmlnote/65435673D66F461CAC358DA377A48FD4/B1BC6D19A3C54AE098528DDCF86D8A52/15135)

**回边计数器**

它的作用是统计一个方法中循环体代码执行的次数，在字节码中遇到控制流向后跳转的指令称为“回边”。建立回边计数器目的是为了触发OSR编译。

当解释器遇到一条回边指令时，会查找将要执行的 代码片段是否有已经编译好的版本，若有则优先执行已编译的代码，否则就把回边计数器的值加1，然后判断方法调用计数器与回边计数器值之和是否超过回边计数器阈值。超过阈值将会提交一个OSR编译请求，并且把回边计数器的值降低一些，以便继续在解释器中循环等待，等待编译器输出编译结果。

回边计数器没有计数热度衰减过程，因此此计数器统计的就是该方法执行的绝对次数。当计数器溢出时，他还会把方法计数器的值调整到溢出状态，这样下次再进入该方法时就会执行标准编译过程。

![img](https://note.youdao.com/yws/public/resource/b4ef39bbc01d43d64e41afc9dabd4843/xmlnote/65435673D66F461CAC358DA377A48FD4/8492EE0A78334A378679C393ED48E9AF/15138)

**2.3 编译过程**

虚拟机在代码编译器还未完成之前，仍然按解释方式继续进行，而编译动作则在后台的编译线程中进行。后台执行编译 过程中，Server Compiler和Client Compiler两个编译器编译过程不一样。

**Client Compiler**

对于Client Compiler来说，它是个简单快速的三段式编译器，主要关注点在于局部性的优化，而放弃了许多耗时较长的全局优化手段。

1. 第一个阶段，一个平台独立的前端将字节码构成一种高级中间代码表示（HIR）。在此之前编译器会在字节码上完成一部分基础优化，如方法内联、常量传播等优化将会在字节码被构成HIR之前完成。
2. 第二个阶段，一个平台相关的后端从HIR中产生低级中间代码表示（LIR），而在此之前会在HIR上完成另外一些优化，如空值检查消除、范围检查消除等，以便让HIR达到更高效的代码表示形式。
3. 最后阶段是在平台相关的后端使用线性扫描算法在LIR上分配寄存器，并在LIR上做窥孔优化，然后产生机器代码。

![img](https://note.youdao.com/yws/public/resource/b4ef39bbc01d43d64e41afc9dabd4843/xmlnote/65435673D66F461CAC358DA377A48FD4/10490751F0A94FA1BA3B83B2FFD44883/15140)

**Server Compiler**

它优化强度很高，会执行所有经典的优化动作，如无用代码消除、循环展开、循环表达式外提、消除公共子表达式、常量传播、基本块重排序等，还会实施一些与Java语言特性相关的优化技术，如范围检查消除、空值检查消除等。另外还可能根据解释器或Client Compiler提供的性能监控信息，进行一些不稳定的激进优化，如守护内联、分支频率预测等

当然它是比较缓慢的，但编译速度远远超过传统的静态优化编译器，且相对于Client Compiler编译输出的代码质量有所提高，可减少本地代码执行时间，从而抵消额外的编译时间开销。

**3. 编译优化技术**

以编译方式执行本地代码比解释方式更快，不只是因为虚拟机解释执行字节码时会消耗额外的时间，还有一个原因是虚拟机设计团队几乎把对代码的所有优化措施都集中在了即时编译器之中。因此一般来说，编译器产生的本地代码会比Javac产生的字节码更加优秀。

**3.1 公共子表达式消除**

它的含义是：若一个表达式E已经计算过了，且从先前的计算到现在E中所有变量的值都未发生变化，那么E的这次出现就成了公共子表达式。对于这种表达式没必要花时间计算，直接使用前面的计算结果就好了。若这种优化权限仅限于程序的基本块内，便称为局部公共子表达式消除，若优化范围涵盖了多个基本块，那就称为全局公共子表达式消除。