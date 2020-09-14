# 并发学习
参照《Java并发编程实战》、《Java并发编程之美》  
## 为什么会有并发
速度对比：CPU>内存>IO设备  
为了平衡三者速度差异性  
1. CPU设计增加了L1/L2/L3 Cache，以均衡CPU与内存之间的速度差异
2. 操作系统增加了进程、线程，以便应用程序在进行IO时可以让出CPU使用权，其他程序可以分配到CPU时间片得以执行
3. 分时操作系统分配给每一个正在运行的进程或线程一段CPU时间片，这个时间片基本是ms级别的，所以给我们的感觉就是好像所有的程序都在同时运行，其实是轮番运行的，只是切换的时间特别短，用户肉眼无法感知。
4. 编译器也会对指令进行优化，使得缓存能得到充分利用（例如公共子表达式消除：```int d = b * c + a * b * c会被优化为E = b * c;int d = E + a * E```）

### CPU缓存带来的可见性问题
同样一个共享变量``name="张三"``，多个线程在单核CPU上执行，即使有缓存线程之间读到的也是最新的数据。但如果是多个线程在多核CPU上执行，一个线程在对name进行了修改``name="李四"``，另一个线程所在CPU是对修改内容不感知的，因为它的缓存还是原本的值``name="张三"``，这就是CPU缓存带来的可见性问题

### 线程切换带来的原子性问题
``i++``自增这个操作实际上不是原子性操作，它是分三步指令实现的
1. 先通过一个tmp临时变量读到i变量的值，把这个i的值读取到寄存器中
2. 对i变量``i=i+1``的操作，这一步是在寄存器中对i进行加1操作的
3. 再把tmp临时变量返回，此时i会被写入到缓存中或者内存中

执行下面代码会有概率打印出<=2000的数
```
private static int i = 0;

public static void main(String[] args) throws InterruptedException {
    Thread thread1 = new Thread(new Plus());
    Thread thread2 = new Thread(new Plus());

    thread1.start();
    thread2.start();

    thread1.join();
    thread2.join();

    System.out.println(i);
}

static class Plus implements Runnable {
    @Override
    public void run() {
        for (int j = 0; j < 1000; j++) {
            i++;
        }
    }
}
```
所以说i++这步操作对于CPU来说并不是一条原子性的指令，而是多条指令的集合。可以说多个线程所在CPU同时读到了同一个值做自增操作，也可以说读到了所在CPU的缓存中的值而造成的线程不安全的问题

### 编译优化带来的有序性问题
在学习单例模式时会有一种DCL写法的懒汉式单例模式
```
public class Singleton {

    private Singleton() { }

    private static volatile Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```
一开始看到这个例子时我们可以分析出double check null的含义，线程A和线程B同时进入``synchronized``临界区，假设线程A进入了临界区，那线程B就会被阻塞，此时线程A完成了``new``指令初始化操作后，``instance``就是一个not null的对象，线程A退出临界区后线程B也会进入临界区，此时``instance``已经是非空状态，所以线程B就不需要对其进行再次初始化操作，也就是进行二次null判断。至于``instance``这个引用为什么加``volatile``修饰，一开始我也是疑惑的，直到学习了``volatile``关键词的语义后才知道它在这里的作用只是为了禁止指令重排序，``instance = new Singleton()``这步操作是分三步指令实现的
1. 堆上分配一块内存空间M
2. 在内存空间M上执行初始化Singleton对象
3. 将JVM栈上的instance reference指向内存空间M

但实际指令的执行情况有可能2在3后边执行，这就会造成instance引用指向了内存空间M，已经是非空状态了，但实际上Singleton对象还并未初始化完成，如果此时其他线程获取到该对象再使用就会出现问题
