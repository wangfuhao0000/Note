## 基本Annotation

在java.lang包下存在着5个基本的Annotation，其中有3个Annotation我们是非常常见的了。

1. **@Override：**重写注解，告诉编译器要检查**该方法是实现父类的**，例如在实现`equals()`方法的时候，把`euqals()`打错了，那么编译器就会发现该方法并不是实现父类的，与注解@Override冲突，于是就会给予错误。
2. **@Deprecated：**过时注解，Java在设计的时候，可能觉得某些方法设计得不好，**为了兼容以前的程序，是不能直接把它抛弃的，于是就设置它为过时**。
3. **@SuppressWarnings：**抑制编译器警告注解，当我们在使用集合的时候，如果没有指定泛型，那么会提示安全检查的警告。
4. @SafeVarargs：Java7堆污染警告，**当把一个不是泛型的集合赋值给一个带泛型的集合的时候**，这种情况就很容易发生堆污染。
5. **@FunctionalInterface**：显式指定该接口是一个**函数式接口**。

## 自定义注解基础

我们是可以自己来写注解，给方法或类注入信息。

### 标记Annotation

**没有任何成员变量的注解称作为标记注解**，`@Overried`就是一个标记注解

```java
//有点像定义一个接口一样，只不过它多了一个@
public @interface MyAnnotation {
}
```

### 元数据Annotation

我们自定义的注解是可以**带成员变量**的，定义带成员变量的注解叫做**元数据Annotation**。在注解中定义成员变量，语法类似于声明方法一样。

```java
public @interface MyAnnotation {
    //定义了两个成员变量
    String username();
    int age();
}
```

注意：在注解上定义的成员变量只能是**String、数组、Class、枚举类、注解**。为什么注解上还要定义注解成员变量？**注解的作用就是给类、方法注入信息**。那么我们经常使用XML文件，告诉程序怎么运行。XML经常会有嵌套的情况，那么当我们在使用注解的时候，也可能需要有嵌套的时候，所以就允许了注解上可以定义成员变量为注解。

## 使用自定义注解

### 常规使用

下面我有一个`add()`的方法，需要username和age参数，我们通过注解来让该方法拥有这两个变量！

```java
//注解拥有什么属性，在修饰的时候就要给出相对应的值
@MyAnnotation(username = "zhongfucheng", age = 20)
public void add(String username, int age) {
}
```

### 默认值

当然啦，我们可以在注解声明属性的时候，使用**default给出默认值**。那么在修饰的时候，就可以不用具体指定了。

```java
public @interface MyAnnotation {
    //定义了两个成员变量，使用default关键字指定默认值
    String username() default "zicheng";
    int age() default 23;
}
```

在修饰的时候就不需要给出具体的值了

```java
@MyAnnotation()
public void add(String username, int age) {
}
```

还有一种特殊的情况，如果**注解上只有一个属性，并且属性的名称为value**，那么在使用的时候，我们**可以不指定value属性，直接赋值给它就行**。

## 把自定义注解的基本信息注入到方法上

上面我们已经使用到了注解，但是目前为止**注解上的信息和方法上的信息是没有任何关联的**。我们自己写的自定义注解是需要我们自己来处理的，利用的是**反射技术**，步骤可分为三步：

1. **反射出该类的方法**
2. **通过方法得到注解上具体的信息**
3. **将注解上的信息注入到方法上**

```java
// 反射出该类的方法
Class aClass = Demo2.class;
Method method = aClass.getMethod("add", String.class, int.class);

// 通过该方法得到具体注解上的信息
MyAnnotation annotation = method.getAnnotation(MyAnnotation.class);
String username = annotation.username();
int age = annotation.age();

//将注解上的信息注入到方法上，调用并使用
Object o = aClass.newInstance();
method.invoke(o, username, age);
```

当我们执行的时候，我们发现会出现异常…

![不能直接使用](https://note.youdao.com/yws/public/resource/222eb81bcc53cbd8ecfff72917a55a53/xmlnote/20416DD66DD04F4691E409FE9CF68CF4/a086b97fa3b528489b42b88c12b40a0b/16038)

此时，我们需要在自定义注解上加入这样一句代码，**它的作用用来指明此注解的作用时机。**

```java
@Retention(RetentionPolicy.RUNTIME) //再次执行时，就可以通过注解来把信息注入到方法中了。
```

## JDK的元Annotation

在JDK中除了java.lang包下有Annotation，**在java.lang.annotation下也有几个常用的元Annotation，大多都是用于修饰其他的Annotation定义**。

### @Retention

@Retention只能用于修饰其他的Annotation，**用于指定被修饰的Annotation被保留多长时间。**

@Retention **包含了一个RetentionPolicy类型的value变量**，所以在使用它的时候，**必须要为value成员变量赋值**

value变量的值只有三个：

```java
public enum RetentionPolicy {
    /**
     * Annotations are to be discarded by the compiler.
     */
    SOURCE,

    /**
     * Annotations are to be recorded in the class file by the compiler
     * but need not be retained by the VM at run time.  This is the default
     * behavior.
     */
    CLASS,

    /**
     * Annotations are to be recorded in the class file by the compiler and
     * retained by the VM at run time, so they may be read reflectively.
     *
     * @see java.lang.reflect.AnnotatedElement
     */
    RUNTIME
}
```

java文件有三个时期：**编译,class,运行。@Retention默认是class**

前面我们是使用反射来得到注解上的信息的，**因为@Retention默认是class，而反射是在运行时期来获取信息的**。因此就获取不到Annotation的信息了。于是，就得在自定义注解上修改它的RetentionPolicy值

### @Target

@Target也是**只能用于修饰另外的Annotation**，**它用于指定被修饰的Annotation用于修饰哪些程序单元**

@Target是只有一个value成员变量的，该成员变量的值是以下的：

```java
public enum ElementType {
    /** Class, interface (including annotation type), or enum declaration */
    TYPE,

    /** Field declaration (includes enum constants) */
    FIELD,

    /** Method declaration */
    METHOD,

    /** Parameter declaration */
    PARAMETER,

    /** Constructor declaration */
    CONSTRUCTOR,

    /** Local variable declaration */
    LOCAL_VARIABLE,

    /** Annotation type declaration */
    ANNOTATION_TYPE,

    /** Package declaration */
    PACKAGE
}
```

如果@Target指定的是ElementType.ANNOTATION_TYPE，那么**该被修饰的Annotation只能修饰Annotaion**

### @Documented

@Documented用于指定**被该Annotation修饰的Annotation类将被javadoc工具提取成文档**。

该元Annotation用得挺少的….

### @Inherited

@Inherited也是用来修饰其他的Annotation的，被修饰过的Annotation将具有继承性。即若Annotation被@Inherited修饰后，**Annotation修饰的类的子类会自动的拥有Annotation。**

## 注入对象到方法或成员变量上

前面我们已经可以使用注解将基本的信息注入到方法上了，现在我们要使用的是**将对象注入到方法上**…..

6.1.2模拟场景：

- Person类，定义username和age属性，拥有uername和age的getter和setter方法

```
public class Person {
    private String username;
    private int age;
    //getter和setter
}
```

- PersonDao类，PersonDao类定义了Person对象，拥有person的setter和getter方法

```
public class PersonDao {
    private Person person;
    public Person getPerson() {
        return person;
    }
    public void setPerson(Person person) {
        this.person = person;
    }
}
```

- 现在我要做的就是：**使用注解将Person对象注入到setPerson()方法中，从而设置了PersonDao类的person属性**

```
public class PersonDao {

    private Person person;

    public Person getPerson() {
        return person;
    }

    //将username为zhongfucheng，age为20的Person对象注入到setPerson方法中
    @InjectPerson(username = "zhongfucheng",age = 20)
    public void setPerson(Person person) {
        this.person = person;
    }
}
```

**步骤：**

①： 自定义一个注解，属性是和JavaBean类一致的

```
//注入工具是通过反射来得到注解的信息的，于是保留域必须使用RunTime
@Retention(RetentionPolicy.RUNTIME)
public @interface InjectPerson {
    String username();
    int age();
}
```

②：编写注入工具

```
//1.使用内省【后边需要得到属性的写方法】，得到想要注入的属性
PropertyDescriptor descriptor = new PropertyDescriptor("person", PersonDao.class);

//2.得到要想注入属性的具体对象
Person person = (Person) descriptor.getPropertyType().newInstance();

//3.得到该属性的写方法【setPerson()】
Method method = descriptor.getWriteMethod();

//4.得到写方法的注解
Annotation annotation = method.getAnnotation(InjectPerson.class);

//5.得到注解上的信息【注解的成员变量就是用方法来定义的】
Method[] methods = annotation.getClass().getMethods();

//6.将注解上的信息填充到person对象上
for (Method m : methods) {
//得到注解上属性的名字【age或name】
    String name = m.getName();
    //看看Person对象有没有与之对应的方法【setAge(),setName()】
    try {
            //6.1这里假设：有与之对应的写方法，得到写方法
            PropertyDescriptor descriptor1 = new PropertyDescriptor(name, Person.class);
            Method method1 = descriptor1.getWriteMethod();//setAge(), setName()
            //得到注解中的值
            Object o = m.invoke(annotation, null);
            //调用Person对象的setter方法，将注解上的值设置进去
            method1.invoke(person, o);
    } catch (Exception e) {
            //6.2 Person对象没有与之对应的方法，会跳到catch来。我们要让它继续遍历注解就好了
            continue;
    }
}

//当程序遍历完之后，person对象已经填充完数据了

//7.将person对象赋给PersonDao【通过写方法】
PersonDao personDao = new PersonDao();
method.invoke(personDao, person);

System.out.println(personDao.getPerson().getUsername());
System.out.println(personDao.getPerson().getAge());
```

③：总结一下步骤

其实我们是这样把对象注入到方法中的：

- 得到想要类中注入的属性
- 得到该属性的对象
- 得到属性对应的写方法
- 通过写方法得到注解
- 获取注解详细的信息
- 将注解的信息注入到对象上
- 调用属性写方法，将已填充数据的对象注入到方法中

6.2把对象注入到成员变量

上面已经说了如何将对象注入到方法上了，那么注入到成员变量上也是非常简单的。

**步骤：**

①：**在成员变量上使用注解**

```
public class PersonDao {

    @InjectPerson(username = "zhongfucheng",age = 20) 
    private Person person;

    public Person getPerson() {
        return person;
    }

    public void setPerson(Person person) {
        this.person = person;
    }
}
```

②：编写注入工具

```
//1.得到想要注入的属性
Field field = PersonDao.class.getDeclaredField("person");
//2.得到属性的具体对象
Person person = (Person) field.getType().newInstance();
//3.得到属性上的注解
Annotation annotation = field.getAnnotation(InjectPerson.class);
//4.得到注解的属性【注解上的属性使用方法来表示的】
Method[] methods = annotation.getClass().getMethods();
//5.将注入的属性填充到person对象上
for (Method method : methods) {
    //5.1得到注解属性的名字
    String name = method.getName();
    //查看一下Person对象上有没有与之对应的写方法
    try {
          //如果有
          PropertyDescriptor descriptor = new PropertyDescriptor(name, Person.class);
          //得到Person对象上的写方法
          Method method1 = descriptor.getWriteMethod();
          //得到注解上的值
          Object o = method.invoke(annotation, null);
          //填充person对象
          method1.invoke(person, o);
     } catch (IntrospectionException e) {
          //如果没有想对应的属性，继续循环
          continue;
     }
}
//循环完之后，person就已经填充好数据了
//6.把person对象设置到PersonDao中
PersonDao personDao = new PersonDao();
field.setAccessible(true);
field.set(personDao, person);
System.out.println(personDao.getPerson().getUsername());
```

七、总结

①：注入对象的步骤：**得到想要注入的对象属性，通过属性得到注解的信息，通过属性的写方法将注解的信息注入到对象上，最后将对象赋给类**。

②：注解其实就是两个作用：

- **让编译器检查代码**
- **将数据注入到方法、成员变量、类上**

③:在JDK中注解分为了

- **基本Annotation：**在lang包下，用于常用于标记该方法，抑制编译器警告等
- **元Annotaion：**在annotaion包下，常用于修饰其他的Annotation定义