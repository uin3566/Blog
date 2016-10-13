#### 为啥要写这篇文章
起初我看Handler相关源码，看到Looper里面有个ThreadLocal，如下，而这个ThreadLocal是理解Looper的关键之一，遂又入了ThreadLocal这个坑。
```
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```
之前也看过一些ThreadLocal的博客，但还是很懵逼，还是要自己看源码才能更深入地理解啊，这里把自己的理解写下来。

#### Thread、ThreadLocal、ThreadLocal.Values的关系
ThreadLocal的作用是，为一个线程保存一个线程本地变量，该变量对本线程全局可见，各个线程之间互不干扰，也互不可见。这是如何做到的呢，下面先大概看一下Thread、ThreadLocal、ThreadLocal.Values的关系。

在Thread类中，有这样一个成员：
```
    /**
     * Normal thread local values.
     */
    ThreadLocal.Values localValues;
```
ThreadLocal.Values是ThreadLocal的静态内部类。在ThreadLocal.Values中有一个table成员，如下：
```
    /**
     * Map entries. Contains alternating keys (ThreadLocal) and values.
     * The length is always a power of 2.
     */
    private Object[] table;
```
而这个table就以key，value的形式存储了线程的本地变量。key是ThreadLocal<T>类型的对象的弱引用，而Value则是线程需要保存的线程本地变量T。这个table的结构如下图所示：
![ThreadLocal.Value.table.png](http://upload-images.jianshu.io/upload_images/2630820-531d629854334123.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

ok，弄明白了这三者的关系，下面深入看下ThreadLocal的源码。

####  ThreadLocal源码解析
######ThreadLocal在Looper中的应用
先来看下ThreadLocal的用法，然后根据api往下理解，我们知道，在android中，每个线程都可以调用Looper.prepare()来生成一个Looper对象来处理线程内消息循环，Looper是线程相关的，每个线程的Looper互不干扰。我们来看看ThreadLocal在Looper中的使用，在Looper类中，有一个ThreadLocal<Looper>对象。
```
    // sThreadLocal.get() will return null unless you've called prepare().
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```
如下所示，在Looper中，使用了ThreadLocal的get和set方法。在prepare中，调用set将Looper存入所在线程的本地变量中，从而将Looper和Thread关联了起来，此后，需要获得所在线程Looper时，只需调用Looper.myLooper()方法，便可获得先前set进去的Looper。因此，我们从ThreadLocal的set，get方法着手分析。
```
     /** Initialize the current thread as a looper.
      * This gives you a chance to create handlers that then reference
      * this looper, before actually starting the loop. Be sure to call
      * {@link #loop()} after calling this method, and end it by calling
      * {@link #quit()}.
      */
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
    
    ...

    /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```

###### ThreadLocal源码分析
先看set方法
```
    /**
     * Sets the value of this variable for the current thread. If set to
     * {@code null}, the value will be set to null and the underlying entry will
     * still be present.
     *
     * @param value the new value of the variable for the caller thread.
     */
    public void set(T value) {
        Thread currentThread = Thread.currentThread();
        Values values = values(currentThread);//标注1
        if (values == null) {
            values = initializeValues(currentThread);//标注2
        }
        values.put(this, value);//标注3
    }
```
先看标注1，调用的是ThreadLocal.Values中的方法。
```
    /**
     * Gets Values instance for this thread and variable type.
     */
    Values values(Thread current) {
        return current.localValues;
    }
```
好吧，其实就是取得currentThread的成员变量localValues，如果此时localValues为null，则调用标注2构造一个Values并复制给localValues成员变量。
```
    /**
     * Creates Values instance for this thread and variable type.
     */
    Values initializeValues(Thread current) {
        return current.localValues = new Values();
    }
```
再看标注3，调用了ThreadLocal.Values的put方法，ok，在此打住，先不急着看，Values留到后面分析，因为Values才是核心。我们来看ThreadLocal的get方法。
```
    /**
     * Returns the value of this variable for the current thread. If an entry
     * doesn't yet exist for this variable on this thread, this method will
     * create an entry, populating the value with the result of
     * {@link #initialValue()}.
     *
     * @return the current value of the variable for the calling thread.
     */
    @SuppressWarnings("unchecked")
    public T get() {
        // Optimized for the fast path.
        Thread currentThread = Thread.currentThread();
        Values values = values(currentThread);
        if (values != null) {
            Object[] table = values.table;
            int index = hash & values.mask;
            if (this.reference == table[index]) {
                return (T) table[index + 1];
            }
        } else {
            values = initializeValues(currentThread);
        }

        return (T) values.getAfterMiss(this);//标注
    }
```
思路与set大体一样，大致意思是：获取currentThread的Values成员，若为空，则初始化该成员，并调用标注代码。若不为空，取得Values的成员table，使用ThreadLocal的hash与values.mask取得this在table中的索引，若table[index] == this.reference，则返回该处索引的值，若不满足此条件，调用标注代码。这里留两个坑：1、hash & values.mask是什么卵意思，怎么就能得到index了，2、什么情况下this.reference ！= table[index]。

set和get都看完了，可以看到原理还是在ThreadLocal.Values中，下面继续剖析ThreadLocal.Values的原理。先看下Values的成员变量。
```
        /**
         * Size must always be a power of 2.
         */
        private static final int INITIAL_SIZE = 16;

        /**
         * Placeholder for deleted entries.
         * 这个单词的意思是墓碑，在被回收的key处起占位的作用
         */
        private static final Object TOMBSTONE = new Object();

        /**
         * Map entries. Contains alternating keys (ThreadLocal) and values.
         * The length is always a power of 2.
         * 存放线程本地变量的数组，示意图已在上文画出，长度一定要是2的倍数
         */
        private Object[] table;

        /** Used to turn hashes into indices.
         * 值为table.length - 1
         */
        private int mask;

        /** Number of live entries. 
         * 没被gc回收的key的个数
         */
        private int size;

        /** Number of tombstones. 
         * 已被回收的key的个数  
         */
        private int tombstones;

        /** Maximum number of live entries and tombstones. 
         * 当size+tombstones >= maximumLoad时，将触发rehash，扩容table
         */
        private int maximumLoad;

        /** Points to the next cell to clean up. 
         * 记录清除被回收的key时的开始清除索引。
         */
        private int clean;
```
上面提到了被回收的key，在此解释一下，table中，偶数位放key，基数位放value，key就是ThreadLocal的reference成员，如下，它是ThreadLocal对象本身的弱引用。在被回收的index处，将执行table[index] = TOMBSTONE，table[index+1] = null
```
    /** Weak reference to this thread local instance. */
    private final Reference<ThreadLocal<T>> reference
            = new WeakReference<ThreadLocal<T>>(this);
```
了解了各成员变量的含义之后，先来看下Values中的put方法(在ThreadLocal的set中调用)
```
        /**
         * Sets entry for given ThreadLocal to given value, creating an
         * entry if necessary.
         */
        void put(ThreadLocal<?> key, Object value) {
            //标注1
            cleanUp();

            // Keep track of first tombstone. That's where we want to go back
            // and add an entry if necessary.
            int firstTombstone = -1;

            //标注2
            for (int index = key.hash & mask;; index = next(index)) {
                Object k = table[index];

                //step1
                if (k == key.reference) {
                    // Replace existing entry.
                    table[index + 1] = value;
                    return;
                }

                //step2
                if (k == null) {
                    if (firstTombstone == -1) {
                        // Fill in null slot.
                        table[index] = key.reference;
                        table[index + 1] = value;
                        size++;
                        return;
                    }

                    // Go back and replace first tombstone.
                    table[firstTombstone] = key.reference;
                    table[firstTombstone + 1] = value;
                    tombstones--;
                    size++;
                    return;
                }

                 //step3
                // Remember first tombstone.
                if (firstTombstone == -1 && k == TOMBSTONE) {
                    firstTombstone = index;
                }
            }
        }
```
标注1调用了cleanUp方法，这里大概讲下cleanUp方法的用途，就不贴代码了，cleanUp的作用是，检测现在需不需要rehash，如果需要，则rehash之后直接return，rehash是将oldTable中的还存活的值拷贝到newTable中，因此，rehash之后，newTable中不存在TOMBSTONE以及被回收了的key，若无需rehash，则在被回收了的key所在的index处，执行table[index]=TOMBSTONE，table[index+1]=null，释放不用了的value。

标注2循环遍历table中的index，在循环体中干了三件事。step1、若index处已存在key，则用新值value替换旧值，并return；step2、若index处取得的key为null，判断firstTombstone是否有效，若无效，将key，value放入Index，index+1中，并return，若有效，则将key，value放入firstTombstone中，并return；step3、若index处的key为TOMBSTONE且此时firstTombstone无效，则将index赋值给firstTombstone，继续循环。

好吧，这里出现了上文所留的两个坑，第一个坑很明显，for循环中index的初值就是坑一，至于第二个坑，和这里为啥要用循环是一个意思。坑先留着，往下看。

看完put，再来看get。
```
        /**
         * Gets value for given ThreadLocal after not finding it in the first
         * slot.
         */
        Object getAfterMiss(ThreadLocal<?> key) {
            Object[] table = this.table;
            int index = key.hash & mask;

            // step1
            // If the first slot is empty, the search is over.
            if (table[index] == null) {
                Object value = key.initialValue();

                // If the table is still the same and the slot is still empty...
                if (this.table == table && table[index] == null) {
                    table[index] = key.reference;
                    table[index + 1] = value;
                    size++;

                    cleanUp();
                    return value;
                }

                // The table changed during initialValue().
                put(key, value);
                return value;
            }

            // Keep track of first tombstone. That's where we want to go back
            // and add an entry if necessary.
            int firstTombstone = -1;

            // step2
            // Continue search.
            for (index = next(index);; index = next(index)) {
                Object reference = table[index];
                if (reference == key.reference) {
                    return table[index + 1];
                }

                // If no entry was found...
                if (reference == null) {
                    Object value = key.initialValue();

                    // If the table is still the same...
                    if (this.table == table) {
                        // If we passed a tombstone and that slot still
                        // contains a tombstone...
                        if (firstTombstone > -1
                                && table[firstTombstone] == TOMBSTONE) {
                            table[firstTombstone] = key.reference;
                            table[firstTombstone + 1] = value;
                            tombstones--;
                            size++;

                            // No need to clean up here. We aren't filling
                            // in a null slot.
                            return value;
                        }

                        // If this slot is still empty...
                        if (table[index] == null) {
                            table[index] = key.reference;
                            table[index + 1] = value;
                            size++;

                            cleanUp();
                            return value;
                        }
                    }

                    // The table changed during initialValue().
                    put(key, value);
                    return value;
                }

                if (firstTombstone == -1 && reference == TOMBSTONE) {
                    // Keep track of this tombstone so we can overwrite it.
                    firstTombstone = index;
                }
            }
        }
```
getAfterMiss方法由ThreadLocal的get方法调用，当第一次取table中index处的key没有取成功时（坑2）或者此时线程的Values成员还没初始化时，会进入该方法。该方法大体分成两步：

step1，判断table[index] == null是否为true，若为true，则调用key.initialValue获得初始值并赋值给value，好了，下面有个条件语句if (this.table == table && table[index] == null)，刚看到这个条件我也是懵逼的，这个函数的第一句话不就是Object[] table = this.table么，为毛这里还要判断这个条件。这里解释一下，因为这中间调用了key.initialValue()，这个函数是这样定义的：
```
    /**
     * Provides the initial value of this variable for the current thread.
     * The default implementation returns {@code null}.
     *
     * @return the initial value of the variable.
     */
    protected T initialValue() {
        return null;
    }
```
用户可以自定义这个函数实现，因此用户完全可以在这个函数里改变table。弄明白了这点，step1应该就明白了。

step2， 若table[index] == null为false，则进入循环体，遍历所有索引，继续搜索，若满足table[index] == key.reference，则返回对应得value，若满足table[index] == null，则调用key.initialValue()获取初始化的value，然后又要判断table是否已被改变，此处逻辑与step1一致，其中多了一个逻辑：若table[firstTombstone] == TOMBSTONE，那么将key,value放入firstTombstone,firstTombstone + 1的位置并返回value。

到这里，这个函数分析完了，不过仍有一点疑惑，当把key，value放在一个table[index]为null的时候，为啥要调用cleanUp？！我在rehash函数的注释中看到了这样一句话：We must rehash every time we fill a null slot; we depend on the presence of null slots to end searches (otherwise, we'll infinitely loop)。而调用rehash的地方只有cleanUp一处，这便解释了这个问题。
那这句注释又怎么理解呢，对于We must rehash every time we fill a null slot，每当我们在null slot中填入一个key，那么table的size就要增加1，当size+tombstones >= maximumLoad时需要扩容table，因此需要调用rehash。对于we depend on the presence of null slots to end searches (otherwise, we'll infinitely loop)，由于扩容会增加null slot的数量，回到getAfterMiss函数，试想，若null slot数量越多，那么table[index] == null条件便越容易满足，getAfterMiss便越早return，也就提高了效率。若不调用rehash，一旦table满了，且table中没有对应得key，将导致无限循环，那就呵呵哒了。


到此，已将核心函数讲解完了。还记得上面挖的两个坑么，现在来一个个的填。

坑一：hash & values.mask是什么卵意思，怎么就能得到index了。我们已经知道了table的结构，偶数位放key，基数位放value，通过hash & values.mask总能得到一个偶数位的index，且能尽可能地避免冲突。hash的定义如下
```
/** Hash counter. */
    private static AtomicInteger hashCounter = new AtomicInteger(0);

    /**
     * Internal hash. We deliberately don't bother with #hashCode().
     * Hashes must be even. This ensures that the result of
     * (hash & (table.length - 1)) points to a key and not a value.
     *
     * We increment by Doug Lea's Magic Number(TM) (*2 since keys are in
     * every other bucket) to help prevent clustering.
     */
    private final int hash = hashCounter.getAndAdd(0x61c88647 * 2);
```
它用final声明了，一旦得到ThreadLocal对象，hash就被赋值了，且不可变，hashCounter被声明为static，由所有ThreadLocal对象共享，每构造一个ThreadLocal对象，hash增加0x61c88647 * 2，这里有个魔数0x61c88647 ，关于这个数字，有很多博客都有所介绍了，反正涉及到了比较深奥的数学原理，斐波拉契散列。。我就不去研究这种学术问题了。。但还是需要对此做个实验的，看下这个魔数的神奇之处。这里写个demo，
```
    public class main {    
        private static void testHash(int tableLength) {        
            final int mask = tableLength - 1;        
            final int add = 0x61c88647 * 2;        
            int hash = 0;        
            for (int i = 0; i < tableLength; i++) {            
                hash += add;            
                int index = hash & mask;            
                System.out.print(index + " ");       
            }    
        }    

        public static void main(String[] args) {
            testHash(32);    
        }
    }
```
执行之后输出为
```
    14 28 10 24 6 20 2 16 30 12 26 8 22 4 18 0 14 28 10 24 6 20 2 16 30 12 26 8 22 4 18 0
```
这里tableLength是32，那么key的个数就是16，可以看到index的值均是偶数，且没有冲突，就是这么牛逼。

坑2：什么情况下this.reference ！= table[index]。为了方便分析，我们再贴一次ThreadLocal的set和get函数
```
    public void set(T value) {
        Thread currentThread = Thread.currentThread();
        Values values = values(currentThread);
        if (values == null) {
            values = initializeValues(currentThread);
        }
        values.put(this, value);
    }

    public T get() {
        // Optimized for the fast path.
        Thread currentThread = Thread.currentThread();
        Values values = values(currentThread);
        if (values != null) {
            Object[] table = values.table;
            int index = hash & values.mask;
            if (this.reference == table[index]) {
                return (T) table[index + 1];
            }
        } else {
            values = initializeValues(currentThread);
        }

        return (T) values.getAfterMiss(this);
    }
```
显然，若没有调用过set，那么this.reference ！= table[index]，那如果调用过set呢，还记得Values的put函数中有个for循环吗，for (int index = key.hash & mask;; index = next(index))，也就是说，在存放ThreadLocal对象时，并不一定是存在了table的index = key.hash & mask处，可能存在其他index了，因此，当调用get的时候，this.reference ！= table[index]是有可能成立的。那么，什么情况下put时，ThreadLocal对象不会存在key.hash & mask处呢，简单贴下这个for循环的代码：
```
            for (int index = key.hash & mask;; index = next(index)) {
                Object k = table[index];

                //step1
                if (k == key.reference) {
                    ...
                    return;
                }

                //step2
                if (k == null) {
                   ...
                   return;
                }
              }
```
若满足step1，step2处的条件，便会return，结束循环。也就是说，k要么是TOMBSTONE，要么是其他的ThreadLocal reference对象。这说明此处的index曾经被其他ThreadLocal对象占用过，为什么呢，我们来看两个细节，先看第一个，
```
    /** Hash counter. */
    private static AtomicInteger hashCounter = new AtomicInteger(0);
```
hashCounter 声明为static了，说明无论是哪个线程的ThreadLocal对象，都公用这一个AtomicInteger，每构造一个ThreadLocal对象，其hash字段相比于上一个构造的ThreadLocal对象增加了0x61c88647 * 2，而不论上一个ThreadLocal对象是否属于本线程。
再看第二个细节，
```
    14 28 10 24 6 20 2 16 30 12 26 8 22 4 18 0 14 28 10 24 6 20 2 16 30 12 26 8 22 4 18 0
```
这是上面demo的输出，可以看到这是一个循环的序列。嗯，就是这样，让我们来假想一下，有两个线程，线程A和线程B，它们的table大小均为32，线程A先构造8个ThreadLocal对象放入自己的table，然后线程B也构建8个ThreadLocal对象放入自己的table，此时线程A再构建一个ThreadLocal对象，欲存入自己的table，而index = hash & values.mask，由上面的序列得知，此时index=14，而index=14处已经存在了线程A第一个放入的ThreadLocal对象，那么this.reference ！= table[index]便顺利成章的成立了。那就只能继续循环咯，index = next(index)，即index += 2，由于这个index序列分的足够散，很快for循环便结束了，效率很高。不得不说0x61c88647这个数实在是神奇。

#### ThreadLocal之static
回到Looper代码中，可以看到ThreadLocal成员被声明成了static
```
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```
那我不禁要想，为毛线要声明成static，我是这么理解的。

1、一般来说ThreadLocal这个key所对应的value应该是一个线程全局可见的，那么要获得这个变量，用static方法会方便些，就像Looper中的myLooper函数.
```
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```
那么sThreadLocal顺利成章被声明成static。

2、我注意到ThreadLocal文件开头有个注释
```
/**
 * Implements a thread-local storage, that is, a variable for which each thread
 * has its own value. All threads share the same {@code ThreadLocal} object,
 * but each sees a different value when accessing it, and changes made by one
 * thread do not affect the other threads. The implementation supports
 * {@code null} values.
 *
 * @see java.lang.Thread
 * @author Bob Lee
 */
public class ThreadLocal<T> {
    ...
}
```
其中有句话，意思是所有线程共享同一个ThreadLocal对象，但是当他们访问ThreadLocal所对应的对象时，都能看到不同的值，改变这个值不会影响到其他线程。冲着这句话，我们使用ThreadLocal时，也应该声明成static不是？想想也有道理，只要能达到每个线程访问自身变量而不影响其他线程的目的就可以了，如果不声明成static，那每个线程都得创建一个ThreadLocal，耗内存。

既然用了static，那好，我看到static就想到内存泄露，ThreadLocal会不会导致内存泄露呢。关于这点，网上有很多讨论，看了很多，觉得还是没能完全吃透，就不在这瞎BB了，但有一点需要明确，就是记得调用remove，记得调用remove，记得调用remove。这样应该就万无一失了。

#### 总结
讲到这里，ThreadLocal应该了解的八九不离十了，但仍有两点疑惑。
1、关于内存泄露问题，线程条件下，线程池条件下，ThreadLocal不声明成static条件下的内存情况，看了挺多文章，感觉都没说清楚。
2、用WeakHashMap代替table是否可行，为什么要重新写一个类似于HashMap功能的table？
不好意思，又挖坑了，谁知道的能否留下评论，求教育，在下实在是感激不尽。

本来只是想写下自己的理解加深下印象的，顺便练习下表述能力，不知不觉写了这么多。第一次写这种技术文章，写得不好的地方希望大家多吐槽，哪里不对的一定要指出来，在下万分感激。
