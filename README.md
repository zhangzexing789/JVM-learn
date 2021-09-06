# JVM-learn

## 元数据空间 Metaspace
- 元空间(Metaspace)登上舞台，方法区存在于元空间(Metaspace)。同时，元空间不再与堆连续，而且是存在于本地内存（Native memory）。
- 本地内存（Native memory），也称为C-Heap，是供JVM自身进程使用的。当Java Heap空间不足时会触发GC，但Native memory空间不够却不会触发GC,而且元空间存在于本地内存，意味着只要本地内存足够，它不会出现像永久代中“java.lang.OutOfMemoryError: PermGen space”这种错误。
    - XX:MetaspaceSize，class metadata的初始空间配额，以bytes为单位，达到该值就会触发垃圾收集进行类型卸载，同时GC会对该值进行调整：如果释放了大量的空间，就适当的降低该值；如果释放了很少的空间，那么在不超过MaxMetaspaceSize（如果设置了的话），适当的提高该值。
    - XX：MaxMetaspaceSize，可以为class metadata分配的最大空间。默认是没有限制的。
    - XX：MinMetaspaceFreeRatio,在GC之后，最小的Metaspace剩余空间容量的百分比，减少为class metadata分配空间导致的垃圾收集。
    - XX:MaxMetaspaceFreeRatio,在GC之后，最大的Metaspace剩余空间容量的百分比，减少为class metadata释放空间导致的垃圾收集。

## 堆 heap
### OOM 问题集锦
- java.lang.OutOfMemoryError：java heap space
    - 堆空间溢出。老年代区域剩余的内存，已经无法满足将要晋升到老年代区域的对象大小，会报此错。一般都是大量对象占据了堆空间，而这些对象持有强引用，导致无法回收，当对象大小之和大于由xmx参数指定的堆空间大小时，溢出错误就发生了。
    - 发生的原因：
        - 设置的堆内存太小.
        - 内存泄露。关注系统稳定运行期，full gc每次gc后的可用内存值是否一直在增大。
        - 由于设计原因导致系统需要过多的内存，如系统中过多地缓存了数据库中的数据，这属于设计问题，需要通过设计减少内存的使用,可以将其放在系统缓存。
        - 长生命周期的对象引用短生命周期对象。
    - 解决方法：
        - 看代码是否有问题；.eg:List.add(" ")在一个死循环中不断的调用add却没有remove。
        - 对于长生命周期对象（如I/O操作）对象后续不用了，objecft=null可以辅助GC，一旦方法脱离了作用域，相应的局部变量应用就会被注销。
        - 代码无法优化后，考虑增加heap size，但是过大将可能会导致Full GC时间过长。
        - 内存泄露：内存可能在某些情况增加几十字节空间但未能释放，每次被GC，很老的对象被GC的较慢。
            - Session中放数据，Session回话消失才会被注销，但是会话要很长的时间才会被注销。
            - 不断full gc，每次时间变长频率变小。当次数达到一定量时，平均full gc时间达到一定比例时会报：`OutOfMemoryError：GC over head limit exceeded`，不断的FULL GC就是不抛出OOM的现象，通常是藏匿的Bug或配置导致。
            - Tomcat的session导致，session信息保存在全局的currentHashMap中，大量的HTTPClient访问创建的临时session。但并没有保存系统中为之分配的SessionKey中和相关的Cookies信息，导致每次请求都创建session（几十个字节，request.getSession被调用时），一般看不出来，是一个堆积如山的过程。
    
- GC效率低下引起的OOM
    - 如果堆空间小，那GC所占时间就多，回收所释放的内存不会少。根据GC占用的系统时间，以及释放内存的大小，虚机会评估GC的效率，一旦虚机认为GC的效率过低，就有可能直接抛出OOM异常。但这个判定不会太随意。`Sun 官方对此的定义是：“并行/并发回收器在GC回收时间过长时会抛出OutOfMemroyError。过长的定义是，超过98%的时间用来做GC并且回收 了不到2%的堆内存。用来避免内存过小造成应用不能正常工作。“`

    - 一般虚机会检查几项：
        - 花在GC上的时间是否超过了98%。
        - 老年代释放的内存是否小于2%。
        - eden区释放的内存是否小于2%。
        - 是否连续最近5次GC都出现了上述几种情况（注意是同时出现）。只有满足所有条件，虚机才会抛出`OutOfMemoryError：GC over head limit exceeded`
    - 解决方法：
        - 检查项目中是否有大量的死循环或有使用大内存的代码，优化代码。
        - 添加参数-XX:-UseGCOverheadLimit 禁用这个检查(默认开启)，其实这个参数解决不了内存问题，只是把错误的信息延后，最终出现 `java.lang.OutOfMemoryError: Java heap space`。(参数UseGCOverheadLimit的意义：当频繁Full GC导致程序僵死现象，一致耗着，如果加上-XX:+UseGCOverheadLimit参数就可以让程序提前退出，避免僵死程序长期占用资源。)
        - dump内存，检查是否存在内存泄露，如果没有，加大heap size。

    - 如果生产环境中遇到了这个问题，在不知道原因时可以通过-verbose:gc -XX:+PrintGCDetails看下到底什么原因造成了异常。通常原因都是因为old区占用过多导致频繁Full GC，最终导致GC overhead limit exceed。如果gc log不够可以借助于JProfile等工具查看内存的占用，old区是否有内存泄露。分析内存泄露还有一个方法-XX:+HeapDumpOnOutOfMemoryError，这样OOM时会自动做Heap Dump，可以拿MAT来排查了。还要留意young区，如果有过多短暂对象分配，可能也会抛这个异常。

[相关链接1](https://www.cnblogs.com/kongzhongqijing/articles/7283599.html)
[相关链接2](https://juejin.cn/post/6965810180605345806)
