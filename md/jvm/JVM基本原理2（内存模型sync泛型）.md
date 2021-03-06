##  Java内存模型

- happens-before 

  - happens-before 关系是用来描述两个操作的内存可见性的。如果操作 X happens-before 操作 Y，那么 X 的结果对于 Y 可见。

  - 实际上，如果后者没有观测前者的运行结果，即后者没有数据依赖于前者，那么它们可能会被重排序。

  - 在同一个线程中，字节码的先后顺序（program order）也暗含了 happens-before 关系

  - 线程间的 happens-before关系

    - 解锁操作 happens-before 之后（这里指时钟顺序先后）对同一把锁的加锁操作
    - volatile 字段的写操作 happens-before 之后（这里指时钟顺序先后）对同一字段的读操作
    - 线程的启动操作（即 Thread.starts()） happens-before 该线程的第一个操作
    - 线程的最后一个操作 happens-before 它的终止事件
      - （即其他线程通过 Thread.isAlive() 或Thread.join() 判断该线程是否中止）。
    - 线程对其他线程的中断操作 happens-before 被中断线程所收到的中断事件
      - 即被中断线程的InterruptedException 异常，或者第三个线程针对被中断线程的 Thread.interrupted 或者Thread.isInterrupted 调用

  - happens-before 关系还具备传递性

    - ```java
      // 如何解决这个问题呢？答案是，将 a 或者 b 设置为 volatile 字段
      
      int a=0, b=0;
      
      public void method1() {
       	int r2 = a;
       	b = 1;
      }
      
      public void method2() {
       	int r1 = b;
       	a = 2;
      }
      ```

    - 解决这种数据竞争问题的关键在于构造一个跨线程的 happens-before 关系 ：操作X happens-before 操作 Y，使得操作 X 之前的字节码的结果对操作 Y 之后的字节码可见。

- Java 内存模型的底层实现
  - Java 内存模型是通过内存屏障来禁止重排序的。对于即时编译器来说，内存屏障将限制它所能做的重排序优化。对于处理器来说，内存屏障会导致缓存的刷新操作。
    - 对于即时编译器来说，它会针对前面提到的每一个 happens-before 关系，向正在编译的目标方法中插入相应的读读、读写、写读以及写写内存屏障。
    - X86_64 架构上读读、读写以及写写内存屏障是空操作（no-op）
    - 这些内存屏障会限制即时编译器的重排序操作
      - 以 volatile 字段访问为例，所插入的内存屏障将不允许 volatile 字段写操作之前的内存访问被重排序至其之后；也将不允许 volatile 字段读操作之后的内存访问被重排序至其之前（写前读后）
    - 即时编译器将根据具体的底层体系架构，将这些内存屏障替换成具体的 CPU 指令
  - X86_64 架构的处理器不能将读操作重排序至写操作之后
  -  只有 volatile 字段写操作之后的写读内存屏障需要用具体指令来替代
    - 该具体指令的效果，可以简单理解为强制刷新处理器的写缓存
    - 强制刷新写缓存，将使得当前线程写入 volatile 字段的值（以及写缓存中已有的其他内存修改），同步至主内存之中。
- 锁，volatile 字段，fnal 字段与安全发布
  - 在解锁时，Java 虚拟机同样需要强制刷新缓存，使得当前线程所修改的内存对其他线程可见。
    - 锁操作的 happens-before 规则的关键字是同一把锁。也就意味着，如果编译器能够（通过逃逸分析）证明某把锁仅被同一线程持有，那么它可以移除相应的加锁解锁操作。
    - 因此也就不再强制刷新缓存。举个例子，即时编译后的 synchronized (new Object()) {}，可能等同于空操作，而不会强制刷新缓存。
  - volatile 字段可以看成一种轻量级的、不保证原子性的同步，其性能往往优于锁操作。
    - 频繁地访问 volatile 字段也会因为不断地强制刷新缓存而严重影响程序的性能。
      - volatile 字段的另一个特性是即时编译器无法将其分配到寄存器里。换句话说，volatile 字段的每次访问均需要直接从内存中读写。
      - 所谓的分配到寄存器中，可以理解为编译器将内存中的值缓存在寄存器中，之后一直用访问寄存器来代表对这个内存的访问的。
        - 遍历一个数组，数组的长度是内存中的值。由于我们每次循环都要比较一次，因此编译器决定把它放在寄存器中，免得每次比较都要读一次内存
        - 对于会更改的内存值，编译器也可以先缓存至寄存器，最后更新回内存即可。
      - 非volatile的
        - jvm不保证何时变量的值会写回内存。假如另一个线程加锁访问这个变量，jvm也不保证它能拿到最新数据。
        - 如果即时编译器把那个变量放在寄存器里维护，那么另一个线程也没办法
      - Volatile会禁止上述优化。
  - fnal 实例字段则涉及新建对象的发布问题。当一个对象包含 fnal 实例字段时，我们希望其他线程只能看到已初始化的 fnal 实例字段。
    - 即时编译器会在 fnal 字段的写操作后插入一个写写屏障，以防某些优化将新建对象的发布重排序至 fnal 字段的写操作之前。





## Java虚拟机是怎么实现 synchronized的

- 当声明 synchronized 代码块时

  - 编译而成的字节码将包含 monitorenter 和 monitorexit 指令。这两种指令均会消耗操作数栈上的一个 synchronized 关键字括号里的引用( synchronized (lock))，作为所要加锁解锁的锁对象。
  - 字节码中包含一个 monitorenter 指令以及多个 monitorexit 指令。这是因为 Java虚拟机需要确保所获得的锁在正常执行路径，以及异常执行路径上都能够被解锁。

- 当用 synchronized 标记方法时

  - 字节码中方法的访问标记包括 ACC_SYNCHRONIZED
  - 表示在进入该方法时，Java 虚拟机需要进行 monitorenter 操作。而在退出该方法时，不管是正常返回，还是向调用者抛异常，Java 虚拟机均需要进行 monitorexit 操作。
  - 这里 monitorenter 和 monitorexit 操作所对应的锁对象是隐式的。对于实例方法来说，这两个操作对应的锁对象是 this；对于静态方法来说，这两个操作对应的锁对象是则是所在类的 Class 实例。

- 关于 monitorenter 和 monitorexit 的作用，我们可以抽象地理解为每个锁对象拥有一个锁计数器和一个指向持有该锁的线程的指针。

  - 当执行 monitorenter 时，如果目标锁对象的计数器为 0，Java 虚拟机会将该锁对象的持有线程设置为当前线程，并且将其计数器加 1。
  - 不为 0 ，如果锁对象的持有线程是当前线程，那么 Java 虚拟机可以将其计数器加 1，否则需要等待，直至持有线程释放该锁
  - 之所以采用这种计数器的方式，是为了允许同一个线程重复获取同一把锁
  - 举个例子，如果一个Java 类中拥有多个 synchronized 方法，那么这些方法之间的相互调用，不管是直接的还是间接的，都会涉及对同一把锁的重复加锁操作。因此，我们需要设计这么一个可重入的特性，来避免编程里的隐式约束。

- 重量级锁

  - Java 线程的阻塞以及唤醒，都是依靠操作系统来完成的
    - 这些操作将涉及系统调用，需要从操作系统的用户态切换至内核态，其开销非常之大
  - 为了尽量避免昂贵的线程阻塞、唤醒操作，Java 虚拟机会在线程进入阻塞状态之前，以及被唤醒后竞争不到锁的情况下，进入自旋状态，在处理器上空跑并且轮询锁是否被释放
    - Java 虚拟机给出的方案是自适应自旋，根据以往自旋等待时是否能够获得锁，来动态调整自旋的时间（循环数目）
    - 自旋状态还带来另外一个副作用，那便是不公平的锁机制。处于阻塞状态的线程，并没有办法立刻竞争被释放的锁。然而，处于自旋状态的线程，则很有可能优先获得这把锁。

- 轻量级锁

  - 多个线程在不同的时间段请求同一把锁，也就是说没有锁竞争。针对这种情形，Java 虚拟机采用了轻量级锁，来避免重量级锁的阻塞以及唤醒。

-  Java 虚拟机是怎么区分轻量级锁和重量级锁的。

  - 对象头中的标记字段（mark word）。它的最后两位便被用来表示该对象的锁状态。
    - 00 代表轻量级锁，01 代表无锁（或偏向锁），10 代表重量级锁，11 则跟垃圾回收算法的标记有关。
  - 加锁操作时
    - Java 虚拟机会判断是否已经是重量级锁。如果不是，它会在当前线程的当前栈桢中划出一块空间，作为该锁的锁记录，并且将锁对象的标记字段复制到该锁记录中（假设当前锁对象的标记字段为 X…XYZ）
    - 然后，Java 虚拟机会尝试用 CAS操作替换锁对象的标记字段
      - 如果是X…X01，则替换为刚才分配的锁记录的地址。由于内存对齐的缘故，它的最后两位为 00。此时，该线程已成功获得这把锁，可以继续执行了。
      - 如果不是 X…X01，那么有两种可能。
        - 第一，该线程重复获取同一把锁。此时，Java 虚拟机会将锁记录清零，以代表该锁被重复获取
        - 第二，其他线程持有该锁。此时，Java 虚拟机会将这把锁膨胀为重量级锁，并且阻塞当前线程。
  - 解锁操作时
    - 如果当前锁记录，值为 0，则代表重复进入同一把锁，直接返回即可。
    - 否则，Java 虚拟机会尝试用 CAS 操作，比较锁对象的标记字段的值是否为当前锁记录的地址
      - 如果是，则替换为锁记录中的值，也就是锁对象原本的标记字段。此时，该线程已经成功释放这把锁。
      - 如果不是，则意味着这把锁已经被膨胀为重量级锁。此时，Java 虚拟机会进入重量级锁的释放过程，唤醒因竞争该锁而被阻塞了的线程。

- 偏向锁

  - 偏向锁针对的情况则更加乐观：从始至终只有一个线程请求某一把锁。
    - 具体来说，在线程进行加锁时，如果该锁对象支持偏向锁，那么 Java 虚拟机会通过 CAS 操作，将当前线程的地址记录在锁对象的标记字段之中，并且将标记字段的最后三位设置为 101
    - 在接下来的运行过程中，每当有线程请求这把锁，Java 虚拟机只需判断锁对象标记字段中：最后三位是否为 101，是否包含当前线程的地址，以及 epoch 值是否和锁对象的类的 epoch 值相同。如果都满足，那么当前线程持有该偏向锁，可以直接返回
  -  epoch 值
    - 偏向锁的撤销
      - 当请求加锁的线程和锁对象标记字段保持的线程地址不匹配时，Java 虚拟机需要撤销该偏向锁。这个撤销过程非常麻烦，它要求持有偏向锁的线程到达安全点，再将偏向锁替换成轻量锁。
      - 总撤销数超过了一个阈值 20，Java 虚拟机会宣布这个类的偏向锁失效	
      - 如果总撤销数超过另一个阈值40， Java 虚拟机会认为这个类已经不再适合偏向锁，之后的加锁过程中直接为该类实例设置轻量级锁
    - 每个类中维护一个 epoch 值，你可以理解为第几代偏向锁，当设置偏向锁时，Java 虚拟机需要将该 epoch 值复制到锁对象的标记字段中。
    - 在宣布某个类的偏向锁失效时，Java 虚拟机实则将该类的 epoch 值加 1，表示之前那一代的偏向锁已经失效。而新设置的偏向锁则需要复制新的 epoch 值。

  

  

  

## Java语法糖与Java编译器（泛型相关）

-  Java 语法和 Java 字节码的差异之处。这些差异之处都是通过Java 编译器来协调的。

- 自动装箱与自动拆箱

  - 之所以需要包装类型，是因为许多 Java 核心类库的 API 都是面向对象的。举个例子，Java 核心类库中的容器类，就只支持引用类型
    - 调用 Integer.valueOf 方法，将 int 类型的值转换为 Integer 类型，再存储至容器类中
    - 调用 Integer.intValue 方法。返回 Integer 对象所存储的 int 值
  - 对于基本类型的数值来说，我们需要先将其转换为对应的包装类，再存入容器之中。这个转换可以是显式，也可以是隐式的，后者正是 Java 中的自动装箱
  -  ArrayList 取出元素时，我们得到的实际上也是 Integer 对象。如果应用程序期待的是一个 int 值，那么就会发生自动拆箱

- 类型擦除

  - 在前面例子生成的字节码中，往 ArrayList 中添加元素的 add 方法，所接受的参数类型是 Object；而从 ArrayList 中获取元素的 get 方法，其返回类型同样也是 Object。

  - 之所以会出现这种情况，是因为 Java 泛型的类型擦除。Java 程序里的泛型信息，在 Java 虚拟机里全部都丢失了。

  - Java 编译器将选取该泛型所能指代的所有类中层次最高的那个，作为替换泛型的类。

  - ```java
    // foo 方法的方法描述符所接收参数的类型以及返回类型都为 Number。
    // 方法描述符是 Java 虚拟机识别方法调用目标方法的关键。
    class GenericTes<T extends Number> {
     T foo(T t) {
     return t;
     }
    }
    ```

- 桥接方法

  - 泛型的类型擦除带来了不少问题。其中一个便是方法重写

  - 为了保证编译而成的 Java 字节码能够保留重写的语义，Java 编译器额外添加了一个桥接方法。该桥接方法在字节码层面重写了父类的方法，并将调用子类的方法。

  - ```java
    // VIPOnlyMerchant 类将包含一个桥接方法 actionPrice(Customer)，它重写了父类的同名同方法描述符的方法。该桥接方法将传入的 Customer 参数强制转换为 VIP 类型，再调用原本的 actionPrice(VIP) 方法。
    
    class Merchant<T extends Cusomer> {
     public double actionPrice(T cusomer) {
     return 0.0d;
     }
    }
    class VIPOnlyMerchant extends Merchant<VIP> {
     @Override
     public double actionPrice(VIP cusomer) {
     return 0.0d;
     }
    }
    ```



































