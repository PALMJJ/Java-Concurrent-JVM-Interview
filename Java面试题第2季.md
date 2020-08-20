## 1 volatile

**1.1 volatile定义**

- Java虚拟机提供的轻量级的同步机制。

**1.2 volatile性质**

- 保证可见性：线程将主内存中的数据拷贝到自己的工作内存中进行操作，操作后再将数据写回主内存，一个线程对于数据的改动其他线程立即可感知到；例如开启两个线程，一个线程对于加了volatile的变量进行改变，之后第二个线程会感知到此变量已被改变。

- 不保证原子性：原子性即完整性，某个线程在做某个业务时，中间不可以被分割，要么同时成功，要么同时失败；例如20个线程，每个线程循环1000次，每次将count值加1，结束后很可能不等于20000；如何解决不保证原子性：使用JUC下的AtomicInteger的getAndIncrement方法，底层原理是CAS。

- 禁止指令重排：单线程环境中确保程序最终执行结果和代码顺序执行结果一致，多线程环境线程交替执行，由于编译器优化重排存在，两个线程中使用的变量能否保证一致无法确定，结果无法预测；例如如下所示，语句可以按1234顺序执行也可以按1324执行，但不可以按4123执行，因为存在数据依赖性。

  ```java
  int x = 11;
  int y = 12;
  x = x + 5;
  y = x * x;
  ```

## 2 单例模式

**2.1 单例模式代码**

- 下述代码在单线程下只会产生一个对象，但多线程模式下可能产生多个对象。

```java
class Singleton {
    private static Singleton singleton = null;

    private Singleton() {
        System.out.println("我是构造方法Singleton");
    }

    public static Singleton getInstance() {
        if (singleton == null) {
            singleton = new Singleton();
        }
        return singleton;
    }

    public static void main(String[] args) {
        System.out.println(Singleton.getInstance() == Singleton.getInstance());
    }
}
```

**2.2 DCL模式**

- 即Double Check Lock，双端检锁，加锁的前和后都进行检查。

- 双端检锁不一定是线程安全的，因为有指令重排，加volatile可以禁止指令重排。

```java
class Singleton {
    private static volatile Singleton singleton = null;

    private Singleton() {
        System.out.println("我是构造方法Singleton");
    }

    public static Singleton getInstance() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }

    public static void main(String[] args) {
        for (int i = 1; i <=10; i++) {
            new Thread(() -> {
                Singleton.getInstance();
            }, String.valueOf(i)).start();
        }
    }
}
```

## 3 CAS

**3.1 CAS：即比较交换**

- compareAndSet方法两个参数为期望值和更新值，意思是现在期望值如果与自己内存值相同，此时情况下修改为更新值。

- 功能是判断内存中某个位置是否为期望值，如果是则更改为新值，这个过程是原子的。

```java
import java.util.concurrent.atomic.AtomicInteger;

class CAS {
    public static void main(String[] args) {
        AtomicInteger atomicInteger = new AtomicInteger(5);
        System.out.println(atomicInteger.compareAndSet(5, 2019) + "\t current data " + atomicInteger.get());
        System.out.println(atomicInteger.compareAndSet(5, 2014) + "\t current data " + atomicInteger.get());
        //true	 current data 2019
		//false	 current data 2019
    }
}
```

**3.2 Unsafe类：CAS核心类**

- 由于Java方法无法直接访问底层系统，需要通过本地（native）方法来访问，Unsafe相当于一个后门，基于该类可以直接操作特定内存的数据。
- 存在于sun.misc包中，其内部方法可以像C指针一样直接操作内存，因为Java中的CAS操作执行依赖于Unsafe类的方法。

**3.3 CAS底层原理**

- 假设两个线程同时执行getAndAddInt方法，两个线程同时拿到主内存一个变量存入自己内存作为副本并进行修改；
- 线程1通过getVolatile方法获取值并被挂起；
- 线程2执行compareAndSwapInt方法比较内存值为自己存的值并将变量修改并存回主内存；
- 线程1此时读取到的值和本身存的值不相同，因为变量被volatile修饰，其他线程对此变量的修改它总是能感知到，必须从主内存重新读值并进行修改才可。

**3.4 CAS缺点**

- 循环时间长，开销大（do...while死循环）；
- 只能保证一个共享变量的原子操作；
- 引出ABA问题。

## 4 ABA问题

**4.1 狸猫换太子**

- 线程T1和线程T2同时拿到主内存数据A，线程T2等待只需2秒，线程T1需10秒，此时线程T2将A修改为B并存回主内存，由于线程T1较慢，线程T2将主内存B又修改回A。
- 时间差会导致数据的变化。

**4.2  解决ABA**

- 原子引用：AtomicReference<V>。

```java
import java.util.concurrent.atomic.AtomicReference;

class User {
    private String userName;
    private int age;

    public User(String userName, int age) {
        this.userName = userName;
        this.age = age;
    }
}

public class AtomicReferenceDemo {
    public static void main(String[] args) {
        User zhangsan = new User("zhangsan", 22);
        User lisi = new User("lisi", 23);
        AtomicReference<User> atomicReference = new AtomicReference<>();
        atomicReference.set(zhangsan);
        System.out.println(atomicReference.compareAndSet(zhangsan, lisi) + "\t" + atomicReference.get().toString()); // true
        System.out.println(atomicReference.compareAndSet(zhangsan, lisi) + "\t" + atomicReference.get().toString()); // false
    }
}
```

**4.3 时间戳的原子引用**

- 新增一种机制，就是修改版本号（类似时间戳），每次修改都增加一次版本号。

**4.4 ABA问题解决**

- AtomicStampedReference<V>。

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;
import java.util.concurrent.atomic.AtomicStampedReference;

public class Test {
    static AtomicReference<Integer> atomicReference = new AtomicReference<>(100);
    static AtomicStampedReference<Integer> atomicStampedReference = new AtomicStampedReference<>(100, 1);

    public static void main(String[] args) {
        System.out.println("============以下是ABA问题的产生============");
        new Thread(() -> {
            atomicReference.compareAndSet(100, 101);
            atomicReference.compareAndSet(101, 100);
        }, "t1").start();
        new Thread(() -> {
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(atomicReference.compareAndSet(100, 2019) + "\t" + atomicReference.get());
        }, "t2").start();

        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("============以下是ABA问题的解决============");
        new Thread(() -> {
            int stamp = atomicStampedReference.getStamp();
            System.out.println(Thread.currentThread().getName() + "\t第一次版本号：" + stamp);
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            atomicStampedReference.compareAndSet(100, 101, atomicStampedReference.getStamp(), atomicStampedReference.getStamp() + 1);
            System.out.println(Thread.currentThread().getName() + "\t第二次版本号：" + atomicStampedReference.getStamp());
            atomicStampedReference.compareAndSet(101, 100, atomicStampedReference.getStamp(), atomicStampedReference.getStamp() + 1);
            System.out.println(Thread.currentThread().getName() + "\t第三次版本号：" + atomicStampedReference.getStamp());
        }, "t3").start();
        new Thread(() -> {
            int stamp = atomicStampedReference.getStamp();
            System.out.println(Thread.currentThread().getName() + "\t第一次版本号：" + stamp);
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            boolean result = atomicStampedReference.compareAndSet(100, 2019, stamp, stamp + 1);
            System.out.println(Thread.currentThread().getName() + "\t修改成功否：" + result + "\t当前最新版本号：" + atomicStampedReference.getStamp());
            System.out.println(Thread.currentThread().getName() + "\t当前最新值：" + atomicStampedReference.getReference());
        }, "t4").start();
    }
}
//============以下是ABA问题的产生============
//true	2019
//============以下是ABA问题的解决============
//t3	第一次版本号：1
//t4	第一次版本号：1
//t3	第二次版本号：2
//t3	第三次版本号：3
//t4	修改成功否：false	当前最新版本号：3
//t4	当前最新值：100
```

## 5 集合类不安全之并发修改异常

**5.1 故障现象：java.util.ConcurrentModificationException**

```java
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

public class Test {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        for (int i = 1; i <= 30; i++) {
            new Thread(() -> {
                list.add(UUID.randomUUID().toString().substring(0, 8));
                System.out.println(list);
            }, String.valueOf(i)).start();
        }
    }
}
```

**5.2 导致原因**

- 并发争抢修改导致，参考花名册签名情况；一个人正在写，另外一个同学过来抢夺，导致数据不一致异常，也就是并发修改异常。

**5.3 解决方案**

- vector类加锁，虽线程安全，但效率急剧下降；ArrayList直接不加锁。

- ```java
  List<String> list = Collections.synchronizedList(new ArrayList<>());
  ```

- ```java
  List<String> list = new CopyOnWriteArrayList<>(); // 写时复制，读写分离
  ```

## 6 集合类不安全之Set

**6.1 故障现象：java.util.ConcurrentModificationException**

```java
import java.util.HashSet;
import java.util.Set;
import java.util.UUID;

public class Test {
    public static void main(String[] args) {
        Set<String> set = new HashSet<>();
        for (int i = 1; i <= 30; i++) {
            new Thread(() -> {
                set.add(UUID.randomUUID().toString().substring(0, 8));
                System.out.println(set);
            }, String.valueOf(i)).start();
        }
    }
}
```

**6.2 解决方案**

- ```java
  Set<String> set = Collections.synchronizedSet(new HashSet<>());
  ```

- ```java
  Set<String> set = new CopyOnWriteArraySet<>(); // 底层就是CopyOnWriteArrayList
  ```

**6.3 HashSet底层就是HashMap**

- HashSet调用add方法时就传递一个key参数，而HashMap需要key-value键值对，此时的value为PRESNET的Object常量。

## 7 集合不安全之Map

**7.1 解决方案**

- ```java
  Map<String> map = new ConcurrentHashMap<>();
  ```

## 8 公平锁与非公平锁

**8.1 是什么？**

- 公平锁：指多个线程按照申请锁的顺序来获取锁，类似排队打饭，先来后到。
- 非公平锁（synchronized和ReentrantLock默认值）：指多个线程获取锁的顺序并不是按照申请顺序来，后申请的线程可能比先申请的线程优先获得锁，在高并发情况下，有可能会造成优先级反转或者饥饿现象。

## 9 可重入锁（递归锁）

**9.1 含义**

- 指同一线程外层函数获得锁之后，内层递归函数仍然能获取该锁；在同一线程外层方法获取锁的时候，进入内层方法会自动获取锁。
- synchronized和ReentrantLock就是典型的可重入锁。

**9.2 代码**

```java
public class Test {
    public static void main(String[] args) {
        Phone phone = new Phone();
        new Thread(() -> {
            try {
                phone.sendSMS();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }, "t1").start();
        new Thread(() -> {
            try {
                phone.sendSMS();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }, "t2").start();
    }
}

class Phone {
    public synchronized void sendSMS() throws Exception {
        System.out.println(Thread.currentThread().getId() + "\t invoked sendSMS()");
        sendEmail();
    }

    public synchronized void sendEmail() throws Exception {
        System.out.println(Thread.currentThread().getId() + "\t invoked sendEmail()");
    }
}
//11	 invoked sendSMS()
//11	 invoked sendEmail()
//12	 invoked sendSMS()
//12	 invoked sendEmail()
```

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class Test {
    public static void main(String[] args) {
        Phone phone = new Phone();
        Thread t3 = new Thread(phone, "t3");
        Thread t4 = new Thread(phone, "t4");
        t3.start();
        t4.start();
    }
}

class Phone implements Runnable {
    public synchronized void sendSMS() throws Exception {
        System.out.println(Thread.currentThread().getId() + "\t invoked sendSMS()");
        sendEmail();
    }

    public synchronized void sendEmail() throws Exception {
        System.out.println(Thread.currentThread().getId() + "\t invoked sendEmail()");
    }

    Lock lock = new ReentrantLock();
    @Override
    public void run() {
        get();
    }

    public void get() {
        lock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + "\t invoked get()");
            set();
        } finally {
            lock.unlock();
        }
    }

    public void set() {
        lock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + "\t invoked set()");
        } finally {
            lock.unlock();
        }
    }
}
//t3	 invoked get()
//t3	 invoked set()
//t4	 invoked get()
//t4	 invoked set()
```

**9.3 锁之间要两两配对，加几次锁就要解几次锁**

## 10 自旋锁（spinlock）

**10.1 含义**

- 指尝试获取锁的线程不会立即阻塞，而是采用循环的方式去尝试获取锁，好处是减少线程上下文切换的损耗，缺点是会消耗CPU。

**10.2 代码**

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;

public class Test {
    // 原子引用线程
    AtomicReference<Thread> atomicReference = new AtomicReference<>();

    public void myLock() {
        Thread thread = Thread.currentThread();
        System.out.println(Thread.currentThread().getName() + "\t come in");;
        while (!atomicReference.compareAndSet(null, thread)) {

        }
    }

    public void myUnLock() {
        Thread thread = Thread.currentThread();
        atomicReference.compareAndSet(thread, null);
        System.out.println(Thread.currentThread().getName() + "\t invoked myUnLock()");
    }

    public static void main(String[] args) {
        Test test = new Test();
        new Thread(() -> {
            test.myLock();
            try {
                TimeUnit.SECONDS.sleep(5);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            test.myUnLock();
        }, "AA").start();

        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        new Thread(() -> {
            test.myLock();
            try {
                TimeUnit.SECONDS.sleep(5);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            test.myUnLock();
        }, "BB").start();
    }
}
//AA	 come in
//BB	 come in
//AA	 invoked myUnLock()
//BB	 invoked myUnLock()
```

## 11 独占锁（写锁）和共享锁（读锁）和互斥锁

**11.1 含义**

- 独占锁：指该锁一次只能被一个线程持有，对ReentrantLock和synchronized而言都是独占锁；
- 共享锁：指该锁可被多个线程所持有；
- 对ReentrantReadWriteLock其读锁是共享锁，写锁是独占锁。

**11.2 代码**

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class Test {
    public static void main(String[] args) {
        MyCache myCache = new MyCache();
        for (int i = 1; i <= 5; i++) {
            final int tempInt = i;
            new Thread(() -> {
               myCache.put(tempInt + "", tempInt + "");
            }, String.valueOf(i)).start();
        }
        for (int i = 1; i <= 5; i++) {
            final int tempInt = i;
            new Thread(() -> {
                myCache.get(tempInt + "");
            }, String.valueOf(i)).start();
        }
    }
}

class MyCache {
    private volatile Map<String, Object> map = new HashMap<>();
    private ReentrantReadWriteLock rwlock = new ReentrantReadWriteLock();

    // 写操作：原子 + 独占
    public void put(String key, Object value) {
        rwlock.writeLock().lock();
        try {
            System.out.println(Thread.currentThread().getName() + "\t 正在写入：" + key);
            try {
                TimeUnit.MILLISECONDS.sleep(300);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            map.put(key, value);
            System.out.println(Thread.currentThread().getName() + "\t写入完成");
        } finally {
            rwlock.writeLock().unlock();
        }
    }

    public void get(String key) {
        rwlock.readLock().lock();
        try {
            System.out.println(Thread.currentThread().getName() + "\t 正在读取：");
            try {
                TimeUnit.MILLISECONDS.sleep(300);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            Object result = map.get(key);
            System.out.println(Thread.currentThread().getName() + "\t读取完成");
        } finally {
            rwlock.readLock().unlock();
        }
    }
}
//1	 正在写入：1
//1	写入完成
//2	 正在写入：2
//2	写入完成
//3	 正在写入：3
//3	写入完成
//4	 正在写入：4
//4	写入完成
//5	 正在写入：5
//5	写入完成
//3	 正在读取：
//1	 正在读取：
//4	 正在读取：
//2	 正在读取：
//5	 正在读取：
//3	读取完成
//1	读取完成
//4	读取完成
//2	读取完成
//5	读取完成
```

## 12 CountDownLatch

**12.1 代码**

```java
import java.util.concurrent.CountDownLatch;

public class Test {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(6);
        for (int i = 1; i <= 6; i++) {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + "\t上完自习，离开教室");
                countDownLatch.countDown();
            }, String.valueOf(i)).start();
        }
        countDownLatch.await();
        System.out.println(Thread.currentThread().getName() + "\t班长最后关门走人");
    }
}
//1	上完自习，离开教室
//2	上完自习，离开教室
//3	上完自习，离开教室
//5	上完自习，离开教室
//6	上完自习，离开教室
//4	上完自习，离开教室
//main	班长最后关门走人
```

```java
public enum CountryEnum {
    ONE(1, "齐"), TWO(2, "楚"), THREE(3, "燕"), FOUR(4, "赵"), FIVE(5, "魏"), SIX(6, "韩");

    private Integer retCode;
    private String retMessage;

    public Integer getRetCode() {
        return retCode;
    }

    public String getRetMessage() {
        return retMessage;
    }

    CountryEnum(Integer retCode, String retMessage) {
        this.retCode = retCode;
        this.retMessage = retMessage;
    }
}

import java.util.concurrent.CountDownLatch;

public class Test {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(6);
        for (int i = 1; i <= 6; i++) {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + "\t国被灭");
                countDownLatch.countDown();
            }, CountryEnum.forEach_CountryEnum(i).getRetMessage()).start();
        }
        countDownLatch.await();
        System.out.println(Thread.currentThread().getName() + "\t秦国统一天下");
    }
}
```

## 13 CyclicBarrier

**13.1 人到齐了才能开会（做加法）**

- 让一组线程到达一个屏障（也叫做同步点）被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续干活，线程进入屏障通过CyclicBarrier的await()方法。

**13.2 代码**

```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class Test {
    public static void main(String[] args) throws InterruptedException {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(7, ()-> {
            System.out.println("************召唤神龙************");
        });
        for (int i = 1; i <= 7; i++) {
            final int tempInt = i;
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + "\t 收集到" + tempInt + "龙珠");
                try {
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }, String.valueOf(i)).start();
        }
    }
}
//1	 收集到1龙珠
//2	 收集到2龙珠
//3	 收集到3龙珠
//4	 收集到4龙珠
//5	 收集到5龙珠
//6	 收集到6龙珠
//7	 收集到7龙珠
//************召唤神龙************
```

## 14 Semaphore（信号量）

**14.1 多个线程抢多个资源**

- 信号量：用于多个共享资源互斥使用；用于并发线程数的控制。

**14.2 代码**

```java
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class Main {
    public static void main(String[] args) {
        Semaphore semaphore = new Semaphore(3); // 模拟3个停车位
        for (int i = 1; i <= 6; i++) { // 模拟6部汽车
            new Thread(() -> {
                try {
                    semaphore.acquire();
                    System.out.println(Thread.currentThread().getName() + "\t抢到车位");
                    TimeUnit.SECONDS.sleep(3);
                    System.out.println(Thread.currentThread().getName() + "\t停车3秒后离开车位");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    semaphore.release();
                }
            }, String.valueOf(i)).start();
        }
    }
}
//1	抢到车位
//2	抢到车位
//3	抢到车位
//2	停车3秒后离开车位
//3	停车3秒后离开车位
//1	停车3秒后离开车位
//4	抢到车位
//5	抢到车位
//6	抢到车位
//4	停车3秒后离开车位
//6	停车3秒后离开车位
//5	停车3秒后离开车位
```

## 15 阻塞队列

**15.1 BlockingQueue**

- 当阻塞队列为空，队列中获取元素会被阻塞；队列满时，往队列添加元素会被阻塞。
- 好处：不需要关心什么时候需要阻塞线程，什么时候需要唤醒线程，因为这一切BlockingQueue都会给你一手操办。

**15.2 BlockingQueue架构**

- 和List都是Collection的子接口；
- ArrayBlockingQueue：由数组结构组成的有界阻塞队列；
- LinkedBlockingQueue：由链表结构组成的有界阻塞队列（但大小默认为Integer.MAX_VALUE）；
- SynchronousQueue：不存储元素的阻塞队列，也即单个元素的队列。

**15.3 BlockingQueue的核心方法**

| 方法类型 | 抛出异常  | 特殊值   | 阻塞   | 超时               |
| -------- | --------- | -------- | ------ | ------------------ |
| 插入     | add(e)    | offer(e) | put(e) | offer(e,time,unit) |
| 移除     | remove()  | poll()   | take() | poll(time,unit)    |
| 检查     | element() | peek()   | 不可用 | 不可用             |

**15.4 代码**

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeUnit;

public class Main {
    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<String> blockingQueue = new ArrayBlockingQueue<>(3);
        blockingQueue.add("a");
        blockingQueue.add("b");
        blockingQueue.add("c");
        // blockingQueue.add("x"); // 队列满，抛出异常

        System.out.println(blockingQueue.element());

        System.out.println(blockingQueue.remove());
        System.out.println(blockingQueue.remove());
        System.out.println(blockingQueue.remove());
        // System.out.println(blockingQueue.remove()); // 队列空，抛出异常

        blockingQueue.offer("a");
        blockingQueue.offer("b");
        blockingQueue.offer("c");
        // blockingQueue.offer("x"); // false

        System.out.println(blockingQueue.peek());

        System.out.println(blockingQueue.poll());
        System.out.println(blockingQueue.poll());
        System.out.println(blockingQueue.poll());
        // System.out.println(blockingQueue.poll()); // null

        blockingQueue.put("a");
        blockingQueue.put("b");
        blockingQueue.put("c");
        // blockingQueue.put("x"); // 一直阻塞等待有空

        blockingQueue.take();
        blockingQueue.take();
        blockingQueue.take();
        // blockingQueue.take();

        System.out.println(blockingQueue.offer("a", 2L, TimeUnit.SECONDS));
        System.out.println(blockingQueue.offer("b", 2L, TimeUnit.SECONDS));
        System.out.println(blockingQueue.offer("c", 2L, TimeUnit.SECONDS));
        // System.out.println(blockingQueue.offer("x", 2L, TimeUnit.SECONDS)); // 阻塞2秒后停止
    }
}
```

**15.5 阻塞队列之同步SynchronousQueue**

- 不存储元素的BlockingQueue；
- 每个put必须等待一个take操作，否则不继续添加元素，反之亦然。

```java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.SynchronousQueue;
import java.util.concurrent.TimeUnit;

public class Main {
    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<String> blockingQueue = new SynchronousQueue<>();
        new Thread(() -> {
            try {
                System.out.println(Thread.currentThread().getName() + "\t put 1");
                blockingQueue.put("1");
                System.out.println(Thread.currentThread().getName() + "\t put 2");
                blockingQueue.put("2");
                System.out.println(Thread.currentThread().getName() + "\t put 3");
                blockingQueue.put("3");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "AAA").start();

        new Thread(() -> {
            try {
                try {
                    TimeUnit.SECONDS.sleep(2);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(blockingQueue.take());

                try {
                    TimeUnit.SECONDS.sleep(2);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(blockingQueue.take());

                try {
                    TimeUnit.SECONDS.sleep(2);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(blockingQueue.take());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "BBB").start();
    }
}
```

## 16 生产者消费者模式

**16.1 代码**

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class Main {
    public static void main(String[] args) throws InterruptedException {
        ShareData shareData = new ShareData();
        new Thread(() -> {
           for (int i = 1; i <= 5; i++) {
               try {
                   shareData.increment();
               } catch (Exception e) {
                   e.printStackTrace();
               }
           }
        }, "AA").start();

        new Thread(() -> {
            for (int i = 1; i <= 5; i++) {
                try {
                    shareData.decrement();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }, "BB").start();
    }
}

// 资源类
class ShareData {
    private int number = 0;
    private Lock lock = new ReentrantLock();
    private Condition condition = lock.newCondition();

    public void increment() throws Exception {
        lock.lock();
        try {
            // 判断
            while (number != 0) {
                condition.await();
            }
            // 干活
            number++;
            System.out.println(Thread.currentThread().getName() + "\t" + number);
            // 通知唤醒
            condition.signalAll();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void decrement() throws Exception {
        lock.lock();
        try {
            // 判断
            while (number == 0) {
                condition.await();
            }
            // 干活
            number--;
            System.out.println(Thread.currentThread().getName() + "\t" + number);
            // 通知唤醒
            condition.signalAll();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

**16.2 Synchronized和Lock的区别**

- 原始构成：synchronized是Java关键字，属于JVM层面；Lock是具体类。
- 使用方法：synchronized不需要用户手动释放锁；ReentrantLock需要手动释放，没有主动释放可能出现死锁。
- 等待是否可中断：synchronized不可中断，除非抛出异常或者正常退出；ReentrantLock设置超时方法tryLock方法，interrupt方法。
- 加锁是否公平：synchronized非公平锁；ReentrantLock两者皆可，默认非公平。
- 锁绑定多个条件Condition：synchronized没有；ReentrantLock用来实现分组唤醒需要唤醒的线程，可以精确唤醒，而不是像synchronized要么随机唤醒一个要么唤醒全部线程。

**16.3 锁绑定多个条件Condition**

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class Main {
    public static void main(String[] args) throws InterruptedException {
        ShareResource shareResource = new ShareResource();
        new Thread(() -> {
           for (int i = 1; i <= 5; i++) {
               shareResource.print5();
           }
        }, "A").start();
        new Thread(() -> {
            for (int i = 1; i <= 10; i++) {
                shareResource.print10();
            }
        }, "B").start();
        new Thread(() -> {
            for (int i = 1; i <= 15; i++) {
                shareResource.print15();
            }
        }, "C").start();
    }
}

// A打印5次，B打印10次，C打印15次
class ShareResource {
    private int number = 1; // A:1, B:2, C:3
    private Lock lock = new ReentrantLock();
    private Condition c1 = lock.newCondition();
    private Condition c2 = lock.newCondition();
    private Condition c3 = lock.newCondition();

    public void print5() {
        lock.lock();
        try {
            // 判断
            while (number != 1) {
                c1.await();
            }
            // 干活
            for (int i = 1; i <= 5; i++) {
                System.out.println(Thread.currentThread().getName() + "\t" + i);
            }
            // 通知
            number = 2;
            c2.signal();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void print10() {
        lock.lock();
        try {
            // 判断
            while (number != 2) {
                c2.await();
            }
            // 干活
            for (int i = 1; i <= 10; i++) {
                System.out.println(Thread.currentThread().getName() + "\t" + i);
            }
            // 通知
            number = 3;
            c3.signal();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void print15() {
        lock.lock();
        try {
            // 判断
            while (number != 3) {
                c3.await();
            }
            // 干活
            for (int i = 1; i <= 15; i++) {
                System.out.println(Thread.currentThread().getName() + "\t" + i);
            }
            // 通知
            number = 1;
            c1.signal();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

## 17 线程通信之生产者消费者阻塞队列版 

17.1 