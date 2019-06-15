## 1.说说List、Set、Map三者的区别

1.List(对付顺序的好帮手)：List接口存储一组不唯一(可以有多个元素引用相同的对象)，有序的对象。

2.Set(注重独一无二的性质)：不允许重复的集合。不会有多个元素引用相同的对象。

3.Map(用Key来搜索的专家)：使用键值对存储。Map会维护与Key有关联的值。两个Key可以引用相同的对象，但Key不能重复，典型的Key是String类型，但也可以是任何对象。

## 2.ArrayList和LinkedList的区别

**1.是否保证线程安全：**ArrayList和LinkedList都是不同步的，也就是不保证线程安全。

**2.底层数据结构：**ArrayList底层使用的是Object数组；LinkedList底层使用的是双向链表数据结构(JDK1.6之前为循环链表，JDK1.7取消了循环。注意双向链表和双向循环链表的区别，下面有介绍到！)。

**3.插入和删除是否受元素位置的影响：**①**ArrayList采用数组存储，所以插入和删除元素的时间复杂度受到元素位置的影响。**比如：执行add(E e)方法的时候，ArrayList会默认在将指定的元素追加到此列表的末尾，这种情况时间复杂度就是O(1)。但是如果要在指定位置i插入和删除元素的话(add(int index, E e))时间复杂度就为O(n-i)。因为在进行上述操作的时候集合中第i和第i个元素之后的(n-i)个元素都要执行向后/向前移一位的操作。②**LinkedList采用链表存储，所以插入、删除元素的时间复杂度不受元素位置的影响，都是近似O(1)而数组为近似O(n)。**

**4.是否支持快速随机访问：**LinkedList不支持高效的随机元素访问，而ArrayList支持。快速随机访问就是通过元素序号快速获取元素对象(对应于get(int index)方法)。

**5.占内存空间：**ArrayList的空间浪费主要体现在在List列表的结尾会预留一定的容量空间，而LinkedList的空间花费则体现在它的每一个元素都需要消耗比ArrayList更多的空间(因为要存放直接后继和直接前驱以及数据)。

**下面再总结一下List的遍历方式选择：**

1.实现了RandomAccess接口的List，优先选择普通的for循环，其次foreach。

2.未实现RandomAccess接口的List，优先选择iterator遍历(foreach遍历底层也是通过iterator实现的)，大size的数据，千万不要使用普通for循环。

**补充内容：双向链表和双向循环链表**

双向链表：包含两个指针，一个prev指向前一个节点，一个next指向后一个节点。

![ååé¾è¡¨](https://camo.githubusercontent.com/b8d9b370f2fa2be5d375c6593bf88a56e542a2a9/68747470733a2f2f7773312e73696e61696d672e636e2f6c617267652f303036724e776f44677931673264703871693578696a3330666b30366964676a2e6a7067)

双向循环链表：最后一个节点的next指向head，而head的prev指向最后一个节点，构成一个环。

![img](https://camo.githubusercontent.com/21ead645f7eeead89fe6d2b194dc1e60be8d1ed8/68747470733a2f2f7773312e73696e61696d672e636e2f6c617267652f303036724e776f44677931673264703861316878656a3330657530367a676d642e6a7067)

## 3.HashMap和HashTable的区别

**HashMap功能上几乎可以等价于Hashtable，区别主要有以下几点**。

**1.线程是否安全：**HashMap是非线程安全的，HashTable是线程安全的；HashTable内部的方法基本都经过sychronized修饰。（如何你要保证线程安全的话就使用ConcurrentHashMap吧！）

**2.效率：**因为线程安全的问题，HashMap要比HashTable效率高一点。另外，HashTable基本被淘汰，不要在代码中使用它。

**3.对null key和null value的支持：**HashMap中，null可以作为键，这样的键只有一个，可以有一个或多个键所对应的值为null。但是在HashTable中put进的键值只要有一个null，直接抛出NullPointerException。

**4.初始容量大小和每次扩容容量大小的不同：**①创建时如果不指定容量初始值，HashTable默认的初始大小为11，之后每次扩充，容量变为原来的2n+1。HashMap默认的初始化大小为16。之后每次扩充，容量变为原来的2倍。②创建时如果给定了容量的初始值，那么HashTable会直接使用你给定的大小，而HashMap会将其扩充为2的幂次方大小，也就是说HashMap总是使用2的幂做为哈希表的大小，后面我们会介绍为什么是2的幂次方。

**5.底层数据结构：**JDK1.8以后的HashMap在解决哈希冲突时有了较大的变化，当链表长度大于阈值(默认为8)时，将链表转化为红黑树，以减少搜索时间。HashTable没有这种机制。

**HashMap中带有初始容量的构造函数：**

```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}
 public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
```

下面这个方法保证了HashMap总是使用2的幂作为哈希表的大小。

```java
static final int tableSizeFor(int cap) {
       int n = cap - 1;
       n |= n >>> 1;
       n |= n >>> 2;
       n |= n >>> 4;
       n |= n >>> 8;
       n |= n >>> 16;
       return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
   }
```

## 4.HashMap和HashSet的区别

如果你看过HashSet源码的话就应该知道：HashSet底层就是基于HashMap是实现的。(HashSet的源码非常少，因为除了clone()、writeObject()、readObject()是HashSet自己不得不实现之外，其他方法都是直接调用HashMap中的方法)。

| HashMap                          | HashSet                                                      |
| -------------------------------- | ------------------------------------------------------------ |
| 实现了Map接口                    | 实现Set接口                                                  |
| 存储键值对                       | 仅存储对象                                                   |
| 调用 `put（）`向map中添加元素    | 调用 `add（）`方法向Set中添加元素                            |
| HashMap使用键（Key）计算Hashcode | HashSet使用成员对象来计算hashcode值，对于两个对象来说hashcode可能相同，所以equals()方法用来判断对象的相等性， |

## 5.HashSet如何检查重复

当你把对象加入HashSet时，HashSet会先计算对象的hashcode值来判断对象加入的位置，同时也会与其他加入的对象的hashcode值作比较，如果没有符合的hashcode，HashSet会假设对象没有重复出现。但是如果发现有相同hashcode值的对象，这时会调用equals()方法来检查hashcode相等的对象是否真的相同。如果两者相同，HashSet就不会让加入操作成功。

**hashCode()与equals()的相关规定：**

1.如果两个对象相等，则hashcode一定也是相同的。

2.两个对象相等，对两个equals方法返回true。

3.两个对象有相同的hashcode值，它们也不一定是相等的。

4.综上，equals方法被覆盖过，则hashcode方法也必须被覆盖。

5.hashCode()的默认行为是对堆上的对象产生独特值。如果没有重写hashCode()，则该class的两个对象无论如何都不会相等(即使这两个对象指向相同的数据)。

**==与equals的区别**

1.==是判断两个变量或实例是不是指向同一个内存空间，equals()是判断两个变量或实例所指向的内存空间的值是不是相同。

2.==是指对内存地址进行比较，equals()是对字符串的内容进行比较。

3.==指引用是否相同，equals()指的是值是否相同。

## 6.HashMap的底层实现

### JDK1.8之前

JDK1.8之前HashMap底层是**数组和链表**结合在一起也就是**链表散列。HashMap通过key的hashCode经过扰动函数处理后得到hash值，然后通过(n-1)&hash判断当前元素存放的位置(这里的n指的是数组的长度)，如果当前位置存在元素的话，就判断该元素与要存入的元素的hash值以及key是否相同，如果相同的话，直接覆盖，不相同的话就通过拉链法解决冲突。**

**所谓的扰动函数指的就是HashMap的hash方法。使用hash方法也就是扰动函数是为了防止一些实现比较差的hashCode方法，换句话说就是使用扰动函数后可以减少碰撞。**

**JDK1.8HashMap的hash方法源码：**

JDK1.8的hash方法相比于JDK1.7hash方法更加简便，但是原理不变。

```java
static final int hash(Object key) {
   int h;
   // key.hashCode()：返回散列值也就是hashcode
   // ^ ：按位异或
   // >>>:无符号右移，忽略符号位，空位都以0补齐
   return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

对比一下JDK1.7的HashMap的hash方法源码

```java
static int hash(int h) {
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```

相比于 JDK1.8 的 hash 方法 ，JDK 1.7 的 hash 方法的性能会稍差一点点，因为毕竟扰动了 4 次。

所谓”**拉链法**“就是：将链表和数组相结合。也就是说创建一个链表数组，数组中每一格就是一个链表，若遇到哈希冲突，则将冲突的值加到链表中即可。

![jdk1.8ä¹åçåé¨ç»æ](https://camo.githubusercontent.com/eec1c575aa5ff57906dd9c9130ec7a82e212c96a/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f332f32302f313632343064626363333033643837323f773d33343826683d34323726663d706e6726733d3130393931)

### JDK1.8之后

相比之前的版本，JDK1.8之后在解决哈希冲突后有了较大的变化，当链表长度大于阈值(默认为8)时，将链表转化为红黑树，以减少搜索时间。

![JDK1.8ä¹åçHashMapåºå±æ°æ®ç»æ](https://camo.githubusercontent.com/20de7e465cac279842851258ec4d1ec1c4d3d7d1/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32322f36373233333736342e6a7067)

TreeMap、TreeSet以及JDK1.8之后的HashMap底层都用到了红黑树。红黑树就是为了解决二叉查找树的缺陷，因为二叉查找树在某些情况下会退化成一个线性结构。

## 7.HashMap的长度为什么是2的幂次方

为了能让HashMap存取高效，尽量减少碰撞，也就是要尽量把数据分配均匀。我们上面也讲到了过了，Hash值的范围-2147483648到2147483648，前后加起来大概40亿的映射空间，只要哈希函数映射的比较均匀松散，一般应用是很难出现碰撞的。但问题是一个40亿长度的数组，内存是放不下的。所以这个散列值是不能直接拿来用的。用之前还要先做对数组的长度取模运算，得到的余数才能用来作为要存放的位置也就是对应的数组下标。这个数组下标的计算方法是”(n-1)&hash“。n代表数组长度。这也就解释了HashMap的长度为什么是2的幂次方。

**这个算法应该如何设计呢？**

我们首先可能会想到采用%取余的操作来实现。但是，重点来了："**取余(%操作中如果除数是2的幂次则等价于与其除数减一的与(&)操作(也就是说 hash%length==hash&(length-1)的前提是 length 是2的 n 次方；)**"。"并且**采用二进制位操作&，相对于%能够提高运算效率，这也就解释了HashMap的长度为什么是2的幂次方**"。

## 8.ConcurrentHashMap线程安全的底层具体实现

### JDK1.7

首先将数据分为一段一段的存储，然后给每一段数据加一把锁，当一个线程占用锁访问其中一段数据时，其他段的数据也能被其他线程访问。

**ConcurrentHashMap是由Segment数据结构和HashEntry数组结构组成。**

Segment实现了ReentrantLock，所以Segment是一种可重入锁，扮演锁的角色。HashEntry用于存储键值对数据。

```java
static class Segment<K,V> extends ReentrantLock implements Serializable {
}
```

### JDK1.8

ConcurrentHashMap取消了Segment分段锁，采用了CAS和synchronized来保证并发安全。数据结构跟HashMap1.8的结构类似，数组+链表/红黑树。Java8在链表长度超过一定阈值(8)时将链表转换为红黑树。

synchronized只锁定当前链表或红黑二叉树的首节点，这样只要hash不冲突，就不会产生并发，效率又提升N倍。

## 9.comparable和Comparator的区别

1.comparable接口实际上是出自java.lang包，它有一个compareTo(Object obj)方法用来排序。

2.comparator接口实际上是出自java.util包它有一个compare(Object obj1, Object obj2)方法用来排序。

一般我们需要对一个集合使用自定义排序时，我们就要重写`compareTo()`方法或`compare()`方法，当我们需要对某一个集合实现两种排序方式，比如一个song对象中的歌名和歌手分别采用一种排序方法的话，我们可以重写`compareTo()`方法和使用自制的Comparator方法或者以两个Comparator来实现歌名排序和歌星名排序，第二种代表我们只能使用两个参数版的 `Collections.sort()`。

### Comparator定制排序

```java
ArrayList<Integer> arrayList = new ArrayList<Integer>();
arrayList.add(-1);
arrayList.add(3);
arrayList.add(3);
arrayList.add(-5);
arrayList.add(7);
arrayList.add(4);
arrayList.add(-9);
arrayList.add(-7);
System.out.println("原始数组:");
System.out.println(arrayList);
// void reverse(List list)：反转
Collections.reverse(arrayList);
System.out.println("Collections.reverse(arrayList):");
System.out.println(arrayList);

// void sort(List list),按自然排序的升序排序
Collections.sort(arrayList);
System.out.println("Collections.sort(arrayList):");
System.out.println(arrayList);
// 定制排序的用法
Collections.sort(arrayList, new Comparator<Integer>() {

    @Override
    public int compare(Integer o1, Integer o2) {
        return o2.compareTo(o1);
    }
});
System.out.println("定制排序后：");
System.out.println(arrayList);
```

Output:

```java
原始数组:
[-1, 3, 3, -5, 7, 4, -9, -7]
Collections.reverse(arrayList):
[-7, -9, 4, 7, -5, 3, 3, -1]
Collections.sort(arrayList):
[-9, -7, -5, -1, 3, 3, 4, 7]
定制排序后：
[7, 4, 3, 3, -1, -5, -7, -9]
```

### 重写compareTo方法实现按年龄来排序

```java
// person对象没有实现Comparable接口，所以必须实现，这样才不会出错，才可以使treemap中的数据按顺序排列
// 前面一个例子的String类已经默认实现了Comparable接口，详细可以查看String类的API文档，另外其他
// 像Integer类等都已经实现了Comparable接口，所以不需要另外实现了

public  class Person implements Comparable<Person> {
    private String name;
    private int age;

    public Person(String name, int age) {
        super();
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    /**
     * TODO重写compareTo方法实现按年龄来排序
     */
    @Override
    public int compareTo(Person o) {
        if (this.age > o.getAge()) {
            return 1;
        } else if (this.age < o.getAge()) {
            return -1;
        }
        return age;
    }
}
public static void main(String[] args) {
    TreeMap<Person, String> pdata = new TreeMap<Person, String>();
    pdata.put(new Person("张三", 30), "zhangsan");
    pdata.put(new Person("李四", 20), "lisi");
    pdata.put(new Person("王五", 10), "wangwu");
    pdata.put(new Person("小红", 5), "xiaohong");
    // 得到key的值的同时得到key所对应的值
    Set<Person> keys = pdata.keySet();
    for (Person key : keys) {
        System.out.println(key.getAge() + "-" + key.getName());

    }
}
```

Output：

```java
5-小红
10-王五
20-李四
30-张三
```

## 10.集合框架底层数据结构总结

### Collection

**1.List**

- **Arraylist：** Object数组
- **Vector：** Object数组
- **LinkedList：** 双向链表(JDK1.6之前为循环链表，JDK1.7取消了循环) 详细可阅读

**2.Set**

- **HashSet（无序，唯一）:** 基于 HashMap 实现的，底层采用 HashMap 来保存元素
- **LinkedHashSet：** LinkedHashSet 继承与 HashSet，并且其内部是通过 LinkedHashMap 来实现的。有点类似于我们之前说的LinkedHashMap 其内部是基于 Hashmap 实现一样，不过还是有一点点区别的。
- **TreeSet（有序，唯一）：** 红黑树(自平衡的排序二叉树。)

### Map

- **HashMap：** JDK1.8之前HashMap由数组+链表组成的，数组是HashMap的主体，链表则是主要为了解决哈希冲突而存在的（“拉链法”解决冲突）。JDK1.8以后在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为8）时，将链表转化为红黑树，以减少搜索时间
- **LinkedHashMap：** LinkedHashMap 继承自 HashMap，所以它的底层仍然是基于拉链式散列结构即由数组和链表或红黑树组成。另外，LinkedHashMap 在上面结构的基础上，增加了一条双向链表，使得上面的结构可以保持键值对的插入顺序。同时通过对链表进行相应的操作，实现了访问顺序相关逻辑。
- **Hashtable：** 数组+链表组成的，数组是 HashMap 的主体，链表则是主要为了解决哈希冲突而存在的
- **TreeMap：** 红黑树（自平衡的排序二叉树）



## 11.如何选用集合？

主要根据集合的特点选用，比如我们需要根据键值获取到元素值时就选用Map接口下的集合，需要排序时选择TreeMap，不需要排序时就选择HashMap，需要保证线程安全就选用ConcurrentHashMap。当我们只需要存放元素值时，就选择实现Collection接口的集合，需要保证元素唯一时选择实现Set接口的集合比如TreeSet或HashSet，不需要就选择实现List接口的比如ArrayList或LinkedList，然后再根据实现这些接口的集合的特点来选用。