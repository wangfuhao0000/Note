**一、简单聊聊JVM**

1.1编译过程

.java文件是由Java源码编译器(上述所说的**javac.exe**)来完成，流程图如下所示：

![img](https://note.youdao.com/yws/public/resource/b4ef39bbc01d43d64e41afc9dabd4843/xmlnote/WEBc94676bfb9ed9e082b8365f5bd71bbca/76B94B56813E492EB1786A8E79F3DFC7/16312)

Java源码编译由以下**三个**过程组成：

- 分析和输入到符号表
- 注解处理
- 语义分析和生成class文件

![img](https://note.youdao.com/yws/public/resource/b4ef39bbc01d43d64e41afc9dabd4843/xmlnote/WEBc94676bfb9ed9e082b8365f5bd71bbca/A82D2292318B4E9E84557E8AE444EB2A/16311)

1.2.1编译时期-语法糖

语法糖可以看做是**编译器实现的一些“小把戏”**，这些“小把戏”可能会使得**效率“大提升”。**

最值得说明的就是**泛型**了，这个语法糖可以说我们是经常会使用到的！

- 泛型只会在Java源码中存在，**编译过后**会被替换为原来的原生类型（Raw Type，也称为裸类型）了。这个过程也被称为：**泛型擦除**。

有了泛型这颗语法糖以后：

- 代码更加简洁【不用强制转换】
- 程序更加健壮【只要编译时期没有警告，那么运行时期就不会出现ClassCastException异常】
- 可读性和稳定性【在编写集合的时候，就限定了类型】

1.4class文件和JVM的恩怨情仇

1.4.1类的加载时机**（重要）**

现在我们例子中生成的两个 .class文件**都会直接被加载到JVM中吗**？？

虚拟机规范则是严格规定了有且只有5种情况必须**立即对类进行“初始化”**(class文件加载到JVM中)：

- 创建类的实例(new 的方式)。访问某个类或接口的静态变量，或者对该静态变量赋值，调用类的静态方法
- 反射的方式
- 初始化某个类的子类，则其父类也会被初始化
- Java虚拟机启动时被标明为启动类的类，直接使用java.exe命令来运行某个主类（包含main方法的那个类）
- 当使用JDK1.7的动态语言支持时(....)

所以说：

- Java类的加载是动态的，它并不会一次性将所有类全部加载后再运行，而是保证程序运行的基础类(像是基类)完全加载到jvm中，至于其他类，**则在需要的时候才加载**。这当然就是为了**节省内存开销**。

1.4.2如何将类加载到jvm

class文件是通过**类的加载器**装载到jvm中的！

Java**默认有三种类加载器**：

![img](https://note.youdao.com/yws/public/resource/b4ef39bbc01d43d64e41afc9dabd4843/xmlnote/WEBc94676bfb9ed9e082b8365f5bd71bbca/E80A3E8EC64C456BA2AD3285D16B1681/16322)

各个加载器的工作责任：

- 1）Bootstrap ClassLoader：负责加载$JAVA_HOME中jre/lib/**rt.jar**里所有的class，由C++实现，不是ClassLoader子类
- 2）Extension ClassLoader：负责加载java平台中**扩展功能**的一些jar包，包括$JAVA_HOME中jre/lib/ext/*.jar或-Djava.ext.dirs指定目录下的jar包
- 3）App ClassLoader：负责记载**classpath**中指定的jar包及目录中class

其实这就是所谓的**双亲委派模型**。简单来说：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把**请求委托给父加载器去完成，依次向上**。

好处：

- **防止内存中出现多份同样的字节码**(安全性角度)

特别说明：

- 类加载器在成功加载某个类之后，会把得到的 java.lang.Class类的实例缓存起来。下次再请求加载该类的时候，类加载器会直接使用缓存的类的实例，而**不会尝试再次加载**。

1.4.2类加载详细过程（重要）

加载器加载到jvm中，接下来其实又分了**好几个步骤**：

- 加载，查找并加载类的二进制数据，在Java堆中也**创建一个java.lang.Class类的对象**。
- 连接，连接又包含三块内容：验证、准备、初始化。 - 1）验证，文件格式、元数据、字节码、符号引用验证； - 2）准备，为类的静态变量分配内存，并将其初始化为默认值； - 3）解析，把类中的符号引用转换为直接引用
- 初始化，为类的静态变量赋予正确的初始值。

![img](https://note.youdao.com/yws/public/resource/b4ef39bbc01d43d64e41afc9dabd4843/xmlnote/WEBc94676bfb9ed9e082b8365f5bd71bbca/6EC0C9A89C80412B81C938D7F994BDFB/16326)

1.4.3JIT即时编辑器

一般我们可能会想：JVM在加载了这些class文件以后，针对这些字节码，**逐条取出，逐条执行**-->解析器解析。

但如果是这样的话，那就**太慢**了！

我们的JVM是这样实现的：

- 就是把这些Java字节码**重新编译优化**，生成机器码，让CPU直接执行。这样编出来的代码效率会更高。
- 编译也是要花费时间的，我们一般对**热点代码**做编译，**非热点代码直接解析**就好了。

热点代码解释：一、多次调用的方法。二、多次执行的循环体

使用热点探测来**检测是否为热点代码**，热点探测有两种方式：

- 采样
- 计数器

目前HotSpot使用的是**计数器的方式**，它为每个方法准备了两类计数器：

- 方法调用计数器（Invocation Counter）
- 回边计数器（Back EdgeCounter）。
- 在确定虚拟机运行参数的前提下，这两个计数器都有一个确定的阈值，**当计数器超过阈值溢出了，就会触发JIT编译**。

![img](https://note.youdao.com/yws/public/resource/b4ef39bbc01d43d64e41afc9dabd4843/xmlnote/WEBc94676bfb9ed9e082b8365f5bd71bbca/021F2E2403DC49FA9F4A4CB0D64A3AA5/16313)

1.5类加载完以后JVM干了什么？

在类加载检查通过后，接下来虚拟机**将为新生对象分配内存**。

1.5.1JVM的内存结构

简单看了一下内存结构，简单看看每个区域究竟存储的是什么(干的是什么)：

- 堆：**存放对象实例**，几乎所有的对象实例都在这里分配内存
- 虚拟机栈：虚拟机栈描述的是**Java方法执行的内存结构**：每个方法被执行的时候都会同时创建一个**栈帧**（Stack Frame）用于存储局部变量表、操作栈、动态链接、方法出口等信息
- 本地方法栈：本地方法栈则是为虚拟机使用到的**Native方法服务**。
- 方法区：存储已**被虚拟机加载的类元数据信息**(元空间)
- 程序计数器：当前线程所执行的字节码的**行号指示器**

1.5.2例子中的流程

![img](https://note.youdao.com/yws/public/resource/b4ef39bbc01d43d64e41afc9dabd4843/xmlnote/WEBc94676bfb9ed9e082b8365f5bd71bbca/762FD04B89E14D7994A8734C5F7C83BD/16314)

我来**宏观简述**一下我们的例子中的工作流程：

- 1、通过 java.exe运行 Java3yTest.class，随后被加载到JVM中，**元空间存储着类的信息**(包括类的名称、方法信息、字段信息..)。
- 2、然后JVM找到Java3yTest的主函数入口(main)，为main函数创建栈帧，开始执行main函数
- 3、main函数的第一条命令是 Java3yjava3y=newJava3y();就是让JVM创建一个Java3y对象，但是这时候方法区中没有Java3y类的信息，所以JVM马上加载Java3y类，把Java3y类的类型信息放到方法区中(元空间)
- 4、加载完Java3y类之后，Java虚拟机做的第一件事情就是在堆区中为一个新的Java3y实例分配内存, 然后调用构造函数初始化Java3y实例，这个**Java3y实例持有着指向方法区的Java3y类的类型信息**（其中包含有方法表，java动态绑定的底层实现）的引用
- 5、当使用 java3y.setName("Java3y");的时候，JVM**根据java3y引用找到Java3y对象**，然后根据Java3y对象持有的引用定位到方法区中Java3y类的类型信息的**方法表**，获得 setName()函数的字节码的地址
- 6、为 setName()函数创建栈帧，开始运行 setName()函数

从微观上其实还做了很多东西，正如上面所说的**类加载过程**（加载-->连接(验证，准备，解析)-->初始化)，在类加载完之后jvm**为其分配内存**(分配内存中也做了非常多的事)。由于这些步骤并不是一步一步往下走，会有很多的“混沌bootstrap”的过程，所以很难描述清楚。

1.6简单聊聊各种常量池

在写这篇文章的时候，原本以为我对 Strings="aaa";类似这些题目已经是不成问题了，直到我遇到了 String.intern()这样的方法与诸如 Strings1=newString("1")+newString("2");混合一起用的时候

- 我发现，我还是太年轻了。

首先我是先阅读了美团技术团队的这篇文章：

- 深入解析String#intern

- - https://tech.meituan.com/indepthunderstandingstringintern.html

嗯，然后就懵逼了。我摘抄一下他的例子：

```
public static void main(String[] args) {
    String s = new String("1");
    s.intern();
    String s2 = "1";
    System.out.println(s == s2);

    String s3 = new String("1") + new String("1");
    s3.intern();
    String s4 = "11";
    System.out.println(s3 == s4);
}
```

打印结果是

- jdk7,8下false true

调换一下位置后：

```
public static void main(String[] args) {

    String s = new String("1");
    String s2 = "1";
    s.intern();
    System.out.println(s == s2);

    String s3 = new String("1") + new String("1");
    String s4 = "11";
    s3.intern();
    System.out.println(s3 == s4);
}
```

打印结果为：

- jdk7,8下false false

1.6.1各个常量池的情况

针对于jdk1.7之后：

- 运行时常量池位于**堆中**
- 字符串常量池位于**堆中**

常量池存储的是：

- 字面量(Literal)：文本字符串等---->用双引号引起来的字符串字面量都会进这里面
- 符号引用(Symbolic References)
- 类和接口的全限定名(Full Qualified Name)
- 字段的名称和描述符(Descriptor)
- 方法的名称和描述符

常量池（Constant Pool Table），用于存放编译期生成的各种字面量和符号引用，这部分内容将在**类加载后进入方法区的运行时常量池中存放**--->来源：深入理解Java虚拟机 JVM高级特性与最佳实践（第二版）

现在我们的运行时常量池只是换了一个位置(原本来方法区，现在在堆中),但可以明确的是：**类加载后，常量池中的数据会在运行时常量池中存放**！

别人总结的常量池：

它是Class文件中的内容，还不是运行时的内容，不要理解它是个池子，其实就是Class文件中的字节码指令

字符串常量池：

HotSpot VM里，记录interned string的一个全局表叫做StringTable，它本质上就是个HashSet。注意**它只存储对java.lang.String实例的引用，而不存储String对象的内容**

**字符串常量池只存储引用，不存储内容**！

再来看一下我们的intern方法：

```
 * When the intern method is invoked, if the pool already contains a
 * string equal to this {@code String} object as determined by
 * the {@link #equals(Object)} method, then the string from the pool is
 * returned. Otherwise, this {@code String} object is added to the
 * pool and a reference to this {@code String} object is returned.
```

- **如果常量池中存在当前字符串，那么直接返回常量池中它的引用**。
- **如果常量池中没有此字符串, 会将此字符串引用保存到常量池中后, 再直接返回该字符串的引用**！

1.6.2解析题目

本来打算写注释的方式来解释的，但好像挺难说清楚的。我还是画图吧...

1. public static void main(String[] args) {
2. 
3. 
4. String s = new String("1");
5. 
6. s.intern();
7. 
8. 
9. String s2 = "1";
10. 
11. System.out.println(s == s2);// false
12. System.out.println("-----------关注公众号：Java3y-------------");
13. }

第一句： Strings=newString("1");

![img](https://note.youdao.com/yws/public/resource/b4ef39bbc01d43d64e41afc9dabd4843/xmlnote/WEBc94676bfb9ed9e082b8365f5bd71bbca/0612101E0E344E6E9785F00A7E31FC0F/16308)

第二句： s.intern();发现字符串常量池中已经存在"1"字符串对象，直接**返回字符串常量池中对堆的引用(但没有接收)**-->此时s引用还是指向着堆中的对象

![img](https://note.youdao.com/yws/public/resource/b4ef39bbc01d43d64e41afc9dabd4843/xmlnote/WEBc94676bfb9ed9e082b8365f5bd71bbca/8AE02ED8858D4D9591F1F18F0C5A29D3/16331)

第三句： Strings2="1";发现字符串常量池**已经保存了该对象的引用**了，直接返回字符串常量池对堆中字符串的引用

![img](https://note.youdao.com/yws/public/resource/b4ef39bbc01d43d64e41afc9dabd4843/xmlnote/WEBc94676bfb9ed9e082b8365f5bd71bbca/3CB5E2F5B8C64FC7B37863C9EE81BCBC/16327)

很容易看到，**两条引用是不一样的！所以返回false**。

1. public static void main(String[] args) {
2. 
3. System.out.println("-----------关注公众号：Java3y-------------");
4. 
5. String s3 = new String("1") + new String("1");
6. 
7. 
8. s3.intern();
9. 
10. 
11. String s4 = "11";
12. System.out.println(s3 == s4); // true
13. }

第一句： Strings3=newString("1")+newString("1");注意：此时**"11"对象并没有在字符串常量池中保存引用**。

![img](https://note.youdao.com/yws/public/resource/b4ef39bbc01d43d64e41afc9dabd4843/xmlnote/WEBc94676bfb9ed9e082b8365f5bd71bbca/5102FA1123CE471CB94A591831949D6C/16319)

第二句： s3.intern();发现"11"对象**并没有在字符串常量池中**，于是将"11"对象在字符串常量池中**保存当前字符串的引用**，并**返回**当前字符串的引用(但没有接收)

![img](https://note.youdao.com/yws/public/resource/b4ef39bbc01d43d64e41afc9dabd4843/xmlnote/WEBc94676bfb9ed9e082b8365f5bd71bbca/C55791EEF6D94324821C43ABBEA83B2C/16309)

第三句： Strings4="11";发现字符串常量池已经存在引用了，直接返回(**拿到的也是与s3相同指向的引用**)

![img](https://note.youdao.com/yws/public/resource/b4ef39bbc01d43d64e41afc9dabd4843/xmlnote/WEBc94676bfb9ed9e082b8365f5bd71bbca/54464AB646BB4E8FA5C0B07277929FAD/16321)

根据上述所说的：最后会返回true~~~

如果还是不太清楚的同学，可以试着接收一下 intern()方法的返回值，再看看上述的图，应该就可以理解了。

下面的就由各位来做做，看是不是掌握了：

1. public static void main(String[] args) {
2. 
3. String s = new String("1");
4. String s2 = "1";
5. s.intern();
6. System.out.println(s == s2);//false
7. 
8. String s3 = new String("1") + new String("1");
9. String s4 = "11";
10. s3.intern();
11. System.out.println(s3 == s4);//false
12. }

还有：

1. public static void main(String[] args) {
2. String s1 = new String("he") + new String("llo");
3. String s2 = new String("h") + new String("ello");
4. String s3 = s1.intern();
5. String s4 = s2.intern();
6. System.out.println(s1 == s3);// true
7. System.out.println(s1 == s4);// true
8. }

1.7GC垃圾回收

可以说GC垃圾回收是JVM中一个非常重要的知识点，应该非常详细去讲解的。但在我学习的途中，我已经发现了有很好的文章去讲解垃圾回收的了。

所以，这里我只简单介绍一下垃圾回收的东西，详细的可以到下面的面试题中查阅和最后给出相关的资料阅 读吧~

1.7.1JVM垃圾回收简单介绍

首先，JVM回收的是**垃圾**，垃圾就是我们程序中已经是不需要的了。垃圾收集器在对堆进行回收前，第一件事情就是要确定这些对象之中哪些还“存活”着，**哪些已经“死去”**。判断哪些对象“死去”常用有两种方式：

- **引用计数法-**->这种难以解决对象之间的循环引用的问题
- **可达性分析算法**-->主流的JVM采用的是这种方式

![img](https://note.youdao.com/yws/public/resource/b4ef39bbc01d43d64e41afc9dabd4843/xmlnote/WEBc94676bfb9ed9e082b8365f5bd71bbca/BD629E55419947F3B6AADDABCDF7E93A/16329)

现在已经可以判断哪些对象已经“死去”了，我们现在要对这些“死去”的对象进行回收，回收也有好几种算法：

- 标记-清除算法
- 复制算法
- 标记-整理算法
- 分代收集算法

无论是可达性分析算法，还是垃圾回收算法，JVM使用的都是**准确式GC**。JVM是使用一组称为**OopMap**的数据结构，来存储所有的对象引用(这样就不用遍历整个内存去查找了，空间换时间)。 并且不会将所有的指令都生成OopMap，只会在**安全点**上生成OopMap，在**安全区域**上开始GC。

- 在OopMap的协助下，HotSpot可以**快速且准确地**完成GC Roots枚举（可达性分析）。

上面所讲的垃圾收集算法只能算是**方法论**，落地实现的是**垃圾收集器**：

- Serial收集器
- ParNew收集器
- Parallel Scavenge收集器
- Serial Old收集器
- Parallel Old收集器
- CMS收集器
- G1收集器

上面这些收集器大部分是可以互相**组合使用**的

![img](https://note.youdao.com/yws/public/resource/b4ef39bbc01d43d64e41afc9dabd4843/xmlnote/WEBc94676bfb9ed9e082b8365f5bd71bbca/4DBB8347CB2941A6BB95C22DC5C9DAC0/16334)

二、JVM面试题

拿些常见的JVM面试题来做做，**加深一下理解和查缺补漏**：

- 1、详细jvm内存结构
- 2、讲讲什么情况下回出现内存溢出，内存泄漏？
- 3、说说Java线程栈
- 4、JVM 年轻代到年老代的晋升过程的判断条件是什么呢？
- 5、JVM 出现 fullGC 很频繁，怎么去线上排查问题？
- 6、类加载为什么要使用双亲委派模式，有没有什么场景是打破了这个模式？
- 7、类的实例化顺序
- 8、JVM垃圾回收机制，何时触发MinorGC等操作
- 9、JVM 中一次完整的 GC 流程（从 ygc 到 fgc）是怎样的
- 10、各种回收器，各自优缺点，重点CMS、G1
- 11、各种回收算法
- 12、OOM错误，stackoverflow错误，permgen space错误

2.1详细jvm内存结构

根据 JVM 规范，JVM 内存共分为虚拟机栈、堆、方法区、程序计数器、本地方法栈五个部分。

![img](https://note.youdao.com/yws/public/resource/b4ef39bbc01d43d64e41afc9dabd4843/xmlnote/WEBc94676bfb9ed9e082b8365f5bd71bbca/CC49657582A54C52A9A2AB96549F1342/16333)

具体**可能会**聊聊jdk1.7以前的PermGen（永久代），替换成Metaspace（元空间）

- 原本永久代存储的数据：符号引用(Symbols)转移到了native heap；字面量(interned strings)转移到了java heap；类的静态变量(class statics)转移到了java heap
- Metaspace（元空间）存储的是类的元数据信息（metadata）
- 元空间的本质和永久代类似，都是**对JVM规范中方法区的实现**。不过元空间与永久代之间最大的区别在于：**元空间并不在虚拟机中，而是使用本地内存**。
- **替换的好处**：一、字符串存在永久代中，容易出现性能问题和内存溢出。二、永久代会为 GC 带来不必要的复杂度，并且回收效率偏低

![img](https://note.youdao.com/yws/public/resource/b4ef39bbc01d43d64e41afc9dabd4843/xmlnote/WEBc94676bfb9ed9e082b8365f5bd71bbca/0800AB6F0EA5456A94322F09EEB9456F/16330)

2.2讲讲什么情况下回出现内存溢出，内存泄漏？

内存泄漏的原因很简单：

- **对象是可达的**(一直被引用)
- 但是对象**不会被使用**

常见的内存泄漏例子：

```
 public static void main(String[] args) {
        Set set = new HashSet();
        for (int i = 0; i < 10; i++) {
            Object object = new Object();
            set.add(object);
            // 设置为空，这对象我不再用了
            object = null;
        }
        // 但是set集合中还维护这obj的引用，gc不会回收object对象
        System.out.println(set);
    }
```

解决这个内存泄漏问题也很简单，将set设置为null，那就可以避免**上诉**内存泄漏问题了。其他内存泄漏得一步一步分析了。

内存溢出的原因：

- 内存泄露导致堆栈内存不断增大，从而引发内存溢出。
- 大量的jar，class文件加载，装载类的空间不够，溢出
- 操作大量的对象导致堆内存空间已经用满了，溢出
- nio直接操作内存，内存过大导致溢出

解决：

- 查看程序是否存在内存泄漏的问题
- 设置参数加大空间
- 代码中是否存在死循环或循环产生过多重复的对象实体、
- 查看是否使用了nio直接操作内存。

2.3说说线程栈

这里的线程栈应该指的是虚拟机栈吧...

JVM规范让**每个Java线程**拥有自己的**独立的JVM栈**，也就是Java方法的调用栈。

当方法调用的时候，会生成一个**栈帧**。栈帧是保存在虚拟机栈中的，栈帧存储了方法的**局部变量表、操作数栈**、动态连接和方法返回地址等信息

线程运行过程中，**只有一个栈帧是处于活跃状态**，称为“当前活跃栈帧”，当前活动栈帧始终是虚拟机栈的**栈顶元素**。

通过**jstack**工具查看线程状态

2.4JVM 年轻代到年老代的晋升过程的判断条件是什么呢？

1. 部分对象会在From和To区域中复制来复制去,**如此交换15次**(由JVM参数MaxTenuringThreshold决定,这个参数默认是15),最终如果还是存活,就存入到老年代。
2. 如果**对象的大小大于Eden的二分之一会直接分配在old**，如果old也分配不下，会做一次majorGC，如果小于eden的一半但是没有足够的空间，就进行minorgc也就是新生代GC。
3. minor gc后，survivor仍然放不下，则放到老年代
4. 动态年龄判断 ，大于等于某个年龄的对象超过了survivor空间一半 ，大于等于某个年龄的对象直接进入老年代

2.5JVM 出现 fullGC 很频繁，怎么去线上排查问题

这题就依据full GC的触发条件来做：

- 如果有perm gen的话(jdk1.8就没了)，**要给perm gen分配空间，但没有足够的空间时**，会触发full gc。 - 所以看看是不是perm gen区的值设置得太小了。
- System.gc()方法的调用 - 这个一般没人去调用吧~~~
- 当**统计**得到的Minor GC晋升到旧生代的平均大小**大于老年代的剩余空间**，则会触发full gc(这就可以从多个角度上看了) - 是不是**频繁创建了大对象(也有可能eden区设置过小)**(大对象直接分配在老年代中，导致老年代空间不足--->从而频繁gc) - 是不是**老年代的空间设置过小了**(Minor GC几个对象就大于老年代的剩余空间了)

![img](https://note.youdao.com/yws/public/resource/b4ef39bbc01d43d64e41afc9dabd4843/xmlnote/WEBc94676bfb9ed9e082b8365f5bd71bbca/EEF0B8AD5DF24E55BFAB60D31F136228/16316)

2.6类加载为什么要使用双亲委派模式？

双亲委托模型的重要用途是为了解决类载入过程中的**安全性问题**。

- 假设有一个开发者自己编写了一个名为 java.lang.Object的类，想借此欺骗JVM。现在他要使用自定义 ClassLoader来加载自己编写的 java.lang.Object类。
- 然而幸运的是，双亲委托模型不会让他成功。因为JVM会优先在 BootstrapClassLoader的路径下找到 java.lang.Object类，并载入它

Java的类加载是否一定遵循双亲委托模型？

2.7类的实例化顺序

- 1． 父类静态成员和静态初始化块 ，按在代码中出现的顺序依次执行
- 2． 子类静态成员和静态初始化块 ，按在代码中出现的顺序依次执行
- 3． 父类实例成员和实例初始化块 ，按在代码中出现的顺序依次执行
- 4． 父类构造方法
- 5． 子类实例成员和实例初始化块 ，按在代码中出现的顺序依次执行
- 6． 子类构造方法

检验一下是不是真懂了：

```
class Dervied extends Base {
    private String name = "Java3y";
    public Dervied() {
        tellName();
        printName();
    }
    public void tellName() {
        System.out.println("Dervied tell name: " + name);
    }
    public void printName() {
        System.out.println("Dervied print name: " + name);
    }
    public static void main(String[] args) 
        new Dervied();
    }
}

class Base {
    private String name = "公众号";
    public Base() {
        tellName();
        printName();
    }
    public void tellName() {
        System.out.println("Base tell name: " + name);
    }
    public void printName() {
        System.out.println("Base print name: " + name);
    }
}
```

输出数据：

1. Dervied tell name: null
2. Dervied print name: null
3. Dervied tell name: Java3y
4. Dervied print name: Java3y

2.8JVM垃圾回收机制，何时触发MinorGC等操作

当young gen中的eden区分配满的时候触发MinorGC(新生代的空间不够放的时候).

2.9JVM 中一次完整的 GC 流程（从 ygc 到 fgc）是怎样的

YGC和FGC是什么 

- YGC ：**对新生代堆进行gc**。频率比较高，因为大部分对象的存活寿命较短，在新生代里被回收。性能耗费较小。
- FGC ：**全堆范围的gc**。默认堆空间使用到达80%(可调整)的时候会触发fgc。以我们生产环境为例，一般比较少会触发fgc，有时10天或一周左右会有一次。

什么时候执行YGC和FGC

- a.eden空间不足,执行 young gc
- b.old空间不足，perm空间不足，调用方法 System.gc() ，ygc时的悲观策略, dump live的内存信息时(jmap –dump:live)，都会执行full gc

2.10各种回收算法

- 标记-清除算法，“标记-清除”（Mark-Sweep）算法，如它的名字一样，算法分为“标记”和“清除”两个阶段：首先标记出所有需要回收的对象，在标记完成后统一回收掉所有被标记的对象。
- 复制算法，“复制”（Copying）的收集算法，它将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。
- 标记-整理算法，标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存
- 分代收集算法，“分代收集”（Generational Collection）算法，把Java堆分为新生代和老年代，这样就可以根据各个年代的特点采用最适当的收集算法。