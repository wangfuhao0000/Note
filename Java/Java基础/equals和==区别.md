##### ==：

- **对于基本类型，比较的是两个变量的值是否相等**
- 对于引用类型变量，比较的是变量(栈)内存中存放的对象的(堆)内存地址，用来判断**两个对象的地址是否相同**，即是否是指相同一个对象。

##### equals：

　　equals用来比较的是两个对象的内容是否相等，由于所有的类都是继承自java.lang.Object类的，所以适用于所有对象，如果没有对该方法进行覆盖的话，调用的仍然是Object类中的方法，**而Object中的equals方法返回的却是==的判断**，如下：

```java
public boolean equals(Object obj) {
    return (this == obj);
}
```

##### 区别

由equals的源码可以看出这里定义的equals与==是等效的（Object类中的equals没什么区别），不同的原因就在于有些类（像String、Integer等类）**对equals进行了重写**，但是没有对equals进行重写的类（比如我们自己写的类）就只能从Object类中继承equals方法，其equals方法与==就也是等效的，除非我们在此类中重写equals。例如String对其重写，它实际的就是**循环比较两个字符串里面的字符**

```java
public boolean equals(Object anObject) {
    if (this == anObject) {
        return true;
    }
    if (anObject instanceof String) {
        String anotherString = (String)anObject;
        int n = value.length;
        if (n == anotherString.value.length) {
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            while (n-- != 0) {   //比较的是字符串里面的字符数组values
                if (v1[i] != v2[i])
                    return false;
                i++;
            }
            return true;
        }
    }
    return false;
}
```

> 另外，"=="比"equals"运行速度快,因为"=="只是比较引用。

##### instanceof

```java
实例对象 instanceof class(interface)
```

java中，instanceof运算符的前一个操作符是一个引用变量，后一个操作数通常是一个类（可以是接口），用于判断**前面的对象是否是后面的类，或者其子类、实现类的实例**。如果是返回true，否则返回false。也就是说：**使用instanceof关键字做判断时， instanceof 操作符的左右操作数必须有继承或实现关系**。

