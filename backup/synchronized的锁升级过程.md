#简介
在多线程中解决线程安全的问题时常用到Synchronized，现在的synchronized相对于早期的synchronized做出了优化，从以前的加锁就是重量级锁优化成了有一个锁升级的过程(偏向锁->轻量级锁->重量级锁)。
##CAS
   cas的全称是compare and swap，从名称上可以看出它是先比较再进行设置，它是一种在多线程环境下实现同步功能的机制。
下面这段代码是在ReentrantLock类中复制的一段关于CAS操作的代码
     
   `protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
     }`

compareAndSwapInt的参数，这里的参数一和参数二现在把他理解成是一个参数**unsafe.compareAndSwapInt(curr, expect, update)**;所以这一个Cas操作里面需要三个参数
参数一：当前值
参数二：期望值
参数三：需要修改成的值
只有在当前值和期望值一致的时候才会将当前值修改成参数三所传入的值。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/32f59a07ab7841acb7d6aee1f8c57e66~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

CAS在JUC包中应用很广泛，比如在AtomicXXX类中使用到了大量的CAS操作，
CAS不是很难理解，有个概念就好。

##markWord
如果了解对象的内存布局的可以略过此段。这个对象的内存布局是和JVM的实现有关，本章所说的是**HotSpot**的实现。
当一个对象被创建出来后它在内存中的布局如下，由四部分组成：

8个字节的markword，（markword里面包含了其它的东西，比如GC标记，锁类型）
4个字节的ClassPoint(此指针指向的Class)，默认是开启指针压缩所以是四个字节，关闭指针压缩后是八个字节
实例对象中的成员属性大小
字节填充(有的JVM需要8字节对齐，如果上面的字节相加后不能被8整除，则需要在此补齐)
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/32f59a07ab7841acb7d6aee1f8c57e66~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)


看到上面的图，应该可以大概的看出来synchronized加锁，其实就是修改的对像头里面的markword的数据。所以synchronized可以对任何一个对象加锁

现在有一个Java类T,将它new出来之后它的对象的内存布局是什么样子的呢？
`class T{
    Integer age;
}`


可以通过一个小工具来查看下这个T类在内存中的对象布局

`    <dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.9</version>
    <scope>compile</scope>
</dependency>
`
通过下面的程序来打印下T对象的布局是什么样子的。
 `    public static void main(String[] args) {
        T o = new T();
    System.out.println(ClassLayout.parseInstance(o).toPrintable());
}`


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7db4205bc492471ca59a9b8b6cd42145~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)
这张图是一个没有加锁的对象的对象布局。

通过synchronized后的对象布局是什么样子的呢？这次再修改下T类，目的是让它存在字节填充
`class T{
    Integer age;
    Integer age1;
}`

`public static void main(String[] args) {
    T o = new T();
    synchronized (o){
        System.out.println(ClassLayout.parseInstance(o).toPrintable());
    }
}`


到这里可能有些小伙伴有疑问，这里为啥是轻量级锁，不应该先是偏向锁吗？原因如下：
因为偏向锁是有4秒的延迟的，所以如果想要看到效果可以在代码里加上sleep(4100)就可以了。或者是通过jvm参数-**XX:BiasedLockingStartupDelay**=0将延迟设置成0


看完这里也对markword有了些了解了，因为在synchronized中加锁就是通过cas的方式修改的markword中的锁状态

#**Synchronized的锁升级**
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ea33f029d65f434e8a3da5a133050331~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)
上图大概就是Synchronized加锁后的一个锁升级的过程。从早期的重量级锁优化到了现在一个轻量级锁。

##**偏向锁**
上面的重量级锁说到重量级锁想要申请一把锁需要用户态到内核态的一个转换，到了后期的JDK版本中，加锁不用在去向OS去申请锁了，只需要在用户态就可以完成加锁。
从名字上可以看出偏向锁它就是偏向某一个线程，把这个锁加到这个线程上，在加锁的时候如果发现当前锁的竞争线程只有一个线程的话，那么这个锁直接偏向这个线程。直接上锁，不存在竞争。并在线程栈中创建一个LR(锁记录)并将markword拷贝到LR中，同时锁中的markwrod中的指针也会指向当前持有锁线程的LR

这里的LR是有什么作用？
首先synchronized是一个可重入锁，它即然是一个可重入锁它就得有一个东西用来记录重入的次数(加锁几次必须解锁几次)。在解锁时LR在栈中弹出一个就表示解锁一次。

当有多个线程竞争的时候会升级成轻量级锁(自旋锁)
通过下图来看下偏向锁是怎么一回事。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/927188829d2641029706d9c0a3382d60~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

当大呆需要上WC时，只有它自已要上ＷＣ，此时并没有其它的人需要上ＷＣ，那么这时这个ＷＣ可以直接给大呆使用，并且大呆把可以标识自已身份的ID贴到门上，表示此时大呆占用了这个WC。
当又有一个线程来抢占锁时发现当前锁已被占用，此时锁会从偏向锁升级成轻量级锁。
匿名偏向
在执行的时候将偏向锁的延迟设置成0-XX:BiasedLockingStartupDelay=0
` public static void main(String[] args) throws InterruptedException {
     T o = new T();
     System.out.println(ClassLayout.parseInstance(o).toPrintable());
 }`


可以看这个程序的执行结果，当前的锁状态是偏向锁，而有意思是的锁存在，但是他并没有指向线程的指针，
这种情况称为匿名偏向。
##**轻量级锁**
说到轻量级锁可能需要在两种情况下来说它，一是在升级成轻量锁之前有偏向锁，另一种是在升级轻量锁之前没有偏向锁，这里说完第一种第二种不用解释各位也会明白是怎么一回事。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7269a5ee402743eb83b5bd8104c14461~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)
还是用上面这个图来解释，此时当前的WC被大呆所占用，这时二呆来了也要使用WC。这时大呆和二呆就要通过CAS的方式来抢占WC。
因为此时锁的状态是偏向锁的状态，二呆来了也要使用WC(这时有两个人同时要使用WC，这时就要将偏向锁升级成轻量级锁)，在升级轻量锁之前首先需要将WC上的标识大呆身份的ID撕下来(这一步叫做偏向锁的撤销)，然后能过自旋+CAS的方式两个人来抢锁。当其中一个线程抢锁成功后，会将LR贴到WC的门上，表示WC当前被某个线程占用，然后另一个没有抢到锁的线程就一直自旋，当自旋一定次数后升级成重量级锁。
如果在升级轻量锁之前没有偏向锁，此时两个线程直接自旋+CAS的方式来抢锁。
##**重量级锁**
在了解重量级锁之前，我想应该先说下用户态与内核态：
对于系统而言，它可以做的一些事情，普通的应用程序是无法完成的，比如系统可以干掉硬盘，如果普通的程序想要干掉硬盘它必须向操作系统去申请，由此操作系统中的指令分了级别，操作系统级别可以访问所有的指令，在用户态下只能访问用户能访问的指令，如果用户态要访问内核态可以执行的指令必须去向操作系统去申请，请操作系统调用。

在JDK早期，上锁只能上重量级锁。因为，所谓的JVM其实它也是工作在用户态的一个进程，如果想要对一个对象进行上锁，那它必须去向系统去申请锁。申请锁成功后，还需要将这把锁从内核态返回到用户态，它称为重量级锁的原因就是在锁申请的时候都要有一个在用户态到内核态的转换。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/705506c724bd4a8b86aef618e1dee226~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)
当抢占到锁后，markword里面记录的不再是LR的指针，而是指向的是一个C++的对象ObjectMonitor，
如果当前线程自旋一段时间后没有抢到锁就会升级成重量级锁，并将当前的线程存入EntryList队列中阻塞，持有锁的线程执行完成后，在唤醒EntryList队列中的线程去抢占锁。
#**总结**
synchronized的锁升级过程，

偏向锁未启动，创建出来的是普通对象， 如果有一个线程来抢占锁，该锁偏向此线程，这时升级为偏向锁。

在偏向锁的基础上又来一个线程抢占锁此时升级为轻量级锁

当一个线程没有抢占到锁，并且自旋了一定时间后还没有抢到锁，就会升级成重量级锁。


在偏向锁的基础上如果出现了重度竞争就会直接升级成重量级锁


偏向锁已启动，创建出来的对象匿名偏向，后面的锁升级和上面写的一样。

