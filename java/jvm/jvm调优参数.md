## jvm内存调优常用参数

### 堆设置 

- **-Xms**: 初始堆大小，eg: Xms20M
- **-Xmx**: 最大堆大小 ，eg:Xmx20M；**将Xmx、Xms大小设置一样，防止内存动态扩展，导致性能不稳定**
- **-Xmn**: 年轻代大小，eg：-Xmn10m。整个堆大小=年轻代大小 + 年老代大小 + 持久代大小。持久代一般固定大小为64m，所以增大年轻代后，将会减小年老代大小，此值对系统性能影响较大,Sun官方推荐配置为整个堆的3/8
- **-XX: MaxDirectMemorySize**： 直接内存大小，**不指定时 和-Xmx大小一样**
- **-XX:NewSize=n**: 设置年轻代大小，eg: -XX:NewSize=10M
- **-XX:NewRatio=n**: *设置年轻代(包括Eden和两个Survivor区)与年老代的比值(除去持久代)*，eg: -XX:NewRatio=2。设置为2，则年轻代与年老代所占比值为1:2，年轻代占整个堆栈的1/3
- **-XX:SurvivorRatio=n**:  设置年轻代中Eden区与Survivor区的大小比值设置为n,  eg:-XX:SurvivorRatio=8 。则两个Survivor区与一个Eden区的比值为2:8，一个Survivor区占整个年轻代的1/10
- **-XX:+/-useTLAB**：线程分配缓冲，*不使用线程缓冲，则分配对象内存时需要使用锁实现同步*
- ~~-XX:MaxPermSize=n~~: 设置持久代大小，java7之前方法区内存大小 

### 收集器设置

- **-XX:+UseSerialGC**: 设置串行收集器，虚拟机在client模式下默认收集器：**serial + serial old **
- **-XX:+UseParallelGC**:设置并行收集器, 虚拟机在server模式下默认收集器：**parallel scavenge + serial old**
- **-XX:+UseParalledlOldGC**:设置并行年老代收集器, **parallel scavenge + parallel old**
- **-XX:+UseConcMarkSweepGC**:设置并发收集器, **parnew + cms + serial old**

### 垃圾回收统计信息 

- -XX:+PrintGC 
- -XX:+PrintGCDetails 
- -XX:+PrintGCTimeStamps 
- -Xloggc:filename

### 并行收集器设置 

- **-XX:ParallelGCThreads=n**: 设置并行收集器收集时使用的CPU数.并行收集线程数. 
- **-XX:MaxGCPauseMillis=n**: 设置并行收集最大暂停时间 
- **-XX:GCTimeRatio=n**: 设置垃圾回收时间占程序运行时间的百分比.公式为1/(1+n)
- **-XX:+CMSIncrementalMode**: 设置为增量模式.适用于单CPU情况. 
- **-XX:ParallelGCThreads=n**: 设置并发收集器年轻代收集方式为并行收集时,使用的CPU数.并行收集线程数.
- **-XX:MaxTenuringThreshold=0**: *设置垃圾最大年龄*。如果设置为0的话，则年轻代对象不经过Survivor区，直接进入年老代。对于年老代比较多的应用，可以提高效率；如果将此值设置为一个较大值，则年轻代对象会在Survivor区进行多次复制，这样可以增加对象再年轻代的存活 时间，增加在年轻代即被回收的概论。
- **-XX:PretenureSizeThreshold**：大于这个值的参数直接在老年代分配，eg：XX:PretenureSizeThreshold=3145728设置3m。默认值为0，即所有的对象现在eden区分配内存

