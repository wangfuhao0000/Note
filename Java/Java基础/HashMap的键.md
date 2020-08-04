大家都知道，HashMap的是key-value（键值对）组成的，这个key既可以是基本数据类型对象，如Integer，Float，同时也可以是自己编写的对象，那么问题来了，这个作为key的对象是否能够改变呢？或者说key能否是一个可变的对象？如果可以该HashMap会怎么样？

可变对象

　　**可变对象**是指创建后自身状态能改变的对象。换句话说，可变对象是**该对象**在创建后它的**哈希值**（由类的hashCode（）方法可以得出哈希值）**可能被改变**。

　　为了能直观的看出哈希值的改变，下面编写了一个类，同时重写了该类的hashCode（）方法和它的equals（）方法【至于为什么要重写equals方法可以看博客：http://www.cnblogs.com/0201zcr/p/4769108.html】，在查找和添加（put方法）的时候都会用到equals方法。

　　在下面的代码中，对象MutableKey的键在创建时变量 i=10 j=20，哈希值是1291。

　　然后我们改变实例的变量值，该对象的键 i 和 j 从10和20分别改变成30和40。现在Key的哈希值已经变成1931。

　　显然，这个对象的键在创建后发生了改变。所以类MutableKey是可变的。

　　让我们看看下面的示例代码：

```
public class MutableKey {
    private int i;
    private int j;
 
    public MutableKey(int i, int j) {
        this.i = i;
        this.j = j;
    }
 
    public final int getI() {
        return i;
    }
 
    public final void setI(int i) {
        this.i = i;
    }
 
    public final int getJ() {
        return j;
    }
 
    public final void setJ(int j) {
        this.j = j;
    }
 
    @Override
    public int hashCode() {
        final int prime = 31;
        int result = 1;
        result = prime * result + i;
        result = prime * result + j;
        return result;
    }
 
    @Override
    public boolean equals(Object obj) {
        if (this == obj) {
            return true;
        }
        if (obj == null) {
            return false;
        }
        if (!(obj instanceof MutableKey)) {
            return false;
        }
        MutableKey other = (MutableKey) obj;
        if (i != other.i) {
            return false;
        }
        if (j != other.j) {
            return false;
        }
        return true;
    }
}
```

**测试：**

```
public class MutableDemo {
 
    public static void main(String[] args) {
 
        // Object created
        MutableKey key = new MutableKey(10, 20);
        System.out.println("Hash code: " + key.hashCode());
 
        // Object State is changed after object creation.
        key.setI(30);
        key.setJ(40);
        System.out.println("Hash code: " + key.hashCode());
    }
}
```

结果:

```
Hash code: 1291
Hash code: 1931
```

 　只要MutableKey 对象的成员变量i或者j改变了，那么该对象的哈希值改变了，所以该对象是一个可变的对象。

HashMap如何存储键值对

　　HashMap底层是使用Entry对象数组存储的，而Entry是一个单项的链表。当调用一个put（）方法将一个键值对添加进来是，先使用hash（）函数获取该对象的hash值，然后调用indexFor方法查找到该对象在数组中应该存储的下标，假如该位置为空，就将value值插入，如果该下标出不为空，则要遍历该下标上面的对象，使用equals方法进行判断，如果遇到equals（）方法返回真的则进行替换，否则将其插入，源码详解可看：http://www.cnblogs.com/0201zcr/p/4769108.html。

　　查找时只需要查询通过key值获取获取hash值，然后找到其下标，遍历该下标下面的Entry对象即可查找到value。【具体看下面源码及其解释】

## 在HashMap中使用可变对象作为Key带来的问题

　　**如果HashMap Key的哈希值在存储键值对后发生改变，Map可能再也查找不到这个Entry了**。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public V get(Object key)   
{   
 // 如果 key 是 null，调用 getForNullKey 取出对应的 value   
 if (key == null)   
     return getForNullKey();   
 // 根据该 key 的 hashCode 值计算它的 hash 码  
 int hash = hash(key.hashCode());   
 // 直接取出 table 数组中指定索引处的值，  
 for (Entry<K,V> e = table[indexFor(hash, table.length)];   
     e != null;   
     // 搜索该 Entry 链的下一个 Entr   
     e = e.next)         // ①  
 {   
     Object k;   
     // 如果该 Entry 的 key 与被搜索 key 相同  
     if (e.hash == hash && ((k = e.key) == key   
         || key.equals(k)))   
         return e.value;   
 }   
 return null;   
}   
```

　　上面是HashMap的get（）方法源码，通过上面我们可以知道，如果 HashMap 的每个 bucket 里**只有一个 Entry** 时，**HashMap 可以根据索引、快速地取出该 bucket 里的 Entry**；在发生“Hash 冲突”的情况下，**单个 bucket 里存储的不是一个 Entry，而是一个 Entry 链**，**系统只能必须按顺序遍历每个 Entry，直到找到想搜索的 Entry 为止**——如果恰好要搜索的 Entry 位于该 Entry 链的最末端（该 Entry 是最早放入该 bucket 中），那系统必须循环到最后才能找到该元素。 

　　同时我们也看到，**判断是否找到该对象，我们还需要判断他的哈希值是否相同**，假如哈希值不相同，根本就找不到我们要找的值。

　　如果Key对象是可变的，那么Key的哈希值就可能改变。在HashMap中可变对象作为Key会造成数据丢失。

　　下面的例子将会向你展示HashMap中有可变对象作为Key带来的问题。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
import java.util.HashMap;
import java.util.Map;
 
public class MutableDemo1 {
 
    public static void main(String[] args) {
 
        // HashMap
        Map<MutableKey, String> map = new HashMap<>();
 
        // Object created
        MutableKey key = new MutableKey(10, 20);
 
        // Insert entry.
        map.put(key, "Robin");
 
        // This line will print 'Robin'
        System.out.println(map.get(key));
 
        // Object State is changed after object creation.
        // i.e. Object hash code will be changed.
        key.setI(30);
 
        // This line will print null as Map would be unable to retrieve the
        // entry.
        System.out.println(map.get(key));
    }
}
```

输出：

```
Robin
null
```

 

如何解决

　　在HashMap中使用不可变对象。在HashMap中，使用String、Integer等不可变类型用作Key是非常明智的。　

　　我们也**能定义属于自己的不可变类**。

　　如果可变对象在HashMap中被用作键，那就要小心在改变对象状态的时候，不要改变它的哈希值了。**我们只需要保证成员变量的改变能保证该对象的哈希值不变即可。**

　　在下面的Employee示例类中，哈希值是用实例变量id来计算的。**一旦Employee的对象被创建，id的值就不能再改变**。只有name可以改变，但name不能用来计算哈希值。所以，**一旦Employee对象被创建，它的哈希值不会改变**。所以Employee在HashMap中用作Key是安全的。

```
import java.util.HashMap;
import java.util.Map;
 
public class MutableSafeKeyDemo {
 
    public static void main(String[] args) {
        Employee emp = new Employee(2);
        emp.setName("Robin");
 
        // Put object in HashMap.
        Map<Employee, String> map = new HashMap<>();
        map.put(emp, "Showbasky");
 
        System.out.println(map.get(emp));
 
        // Change Employee name. Change in 'name' has no effect
        // on hash code.
        emp.setName("Lily");
        System.out.println(map.get(emp));
    }
}
 
class Employee {
    // It is specified while object creation.
    // Cannot be changed once object is created. No setter for this field.
    private int id;
    private String name;
 
    public Employee(final int id) {
        this.id = id;
    }
 
    public final String getName() {
        return name;
    }
 
    public final void setName(final String name) {
        this.name = name;
    }
 
    public int getId() {
        return id;
    }
 
    // Hash code depends only on 'id' which cannot be
    // changed once object is created. So hash code will not change
    // on object's state change
    @Override
    public int hashCode() {
        final int prime = 31;
        int result = 1;
        result = prime * result + id;
        return result;
    }
 
    @Override
    public boolean equals(Object obj) {
        if (this == obj)
            return true;
        if (obj == null)
            return false;
        if (getClass() != obj.getClass())
            return false;
        Employee other = (Employee) obj;
        if (id != other.id)
            return false;
        return true;
    }
}
```

输出

```
Showbasky
Showbasky
```