**1.介绍**

Atomic包里一共提供了13个类，属于4中类型的原子更新方式，分别是原子更新基本类型、原子更新数组、原子更新引用和原子更新属性（字段），这里面的类基本都是使用**Unsafe**实现的包装类。

**1.1 原子更新基本类型**

- AtomicBoolean：原子更新布尔类型
- AtomicInteger：整型
- AtomicLong：长整型

看一下其中的方法getAndIncrement是如何实现原子操作：

```
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}
// 在Unsafe中循环，这是JDK 8，JDK7则是在AtomicInteger中循环
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));
    return var5;
}
```

其中Unsafe基本上都是使用下面这三个方法来实现基本类型的原子操作，如下：

```
public final native boolean compareAndSwapObject(Object var1, long var2, Object var4, Object var5);
public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
public final native boolean compareAndSwapLong(Object var1, long var2, long var4, long var6);
```

**1.2 原子更新数组**

- AtomicIntegerArray：原子更新整形数组里的元素
- AtomicLongArray：原子更新长整形数组里的元素
- AtomicReferenceArray：原子更新引用类型数组里的元素

其实也是用Unsafe实现，且利用了Unsafe里的那三个方法中的一个。要注意的是若通过构造方法将一个数组传递给AtomicIntegerArray，则它会将传进去的数组复制一份，当AtomicIntegerArray对内部数组元素更改时，不影响传入的数组。构造函数代码如下：

```
public AtomicIntegerArray(int[] array) {
    // Visibility guaranteed by final field guarantees
    this.array = array.clone();
}
```

**1.3 原子更新引用类型**

- AtomicReference：原子更新引用类型
- AtomicReferenceFieldUpdater：原子更新引用类型里的字段
- AtomicMarkableReference：原子更新带有标记位的引用类型，可原子更新一个布尔类型的标记位和引用类型。

要注意这个方法的使用方法，一般若要原子更新一个对象（引用），则需要创建一个AtomicReference<class>对象，然后调用AtomicReference.compareAndSet(old, new);其中old和new分别是要更新的对象和更新后的对象。看一下代码：

```
public static AtomicReference<User> atomicUserRef = new AtomicReference<>();

public static void main(String[] args) {
    User user = new User("conan", 15);
    atomicUserRef.set(user);   // 引用老对象
    User updateUser = new User("shinichi", 17);
    atomicUserRef.compareAndSet(user, updateUser);  // 原子更新成新对象
}
// 内部实现
public final native boolean compareAndSwapObject(Object var1, long var2, Object var4, Object var5);
```

**1.4 原子更新字段类**

若需要原子的更新某个类里的某个字段时，就需要使用原子更新字段类：

- AtomicIntegerFieldUpdater：原子更新**整形字段**的更新器
- AtomicLongFieldUpdater：原子更新**长整形字段**的更新器
- **AtomicStampedReference**：原子更新带有版本号的引用类型。该类将整数值与引用关联起来，可用于原子的更新**数据**和数据的**版本号**，解决可能出现的**ABA**问题。

要想原子地更新字段类需要两步：

1. 因为原子更新字段类都是抽象类，每次使用须使用**静态方法**newUpdater()创建一个更新器，并设置想要更新的**类**和**属性**
2. 更新类的字段（属性）必须使用**public volatile**修饰符。

例如下面的代码：

```
public class AtomicIntegerFieldUpdaterTest {
    // 构造时传进了类和对应的属性
    private static AtomicIntegerFieldUpdater<User> a = AtomicIntegerFieldUpdater.newUpdater(User.class, "old");
    public static void main(String[] args) {
        User conan = new User("conan", 10);
        System.out.println(a.getAndIncrement(conan));
        System.out.println(a.get(conan));
    }
    static class User {
        private String name;
        public volatile int old;
        public User(String name, int old) {
            this.name = name;
            this.old = old;
        }
    }
}

// 内部实现，和原子更新引用类型一样
public final native boolean compareAndSwapObject(Object var1, long var2, Object var4, Object var5);
```