# 期中报告

## 选题内容

首先在这里总结一下最终选题，主要是 **rocksdb中的多核相关性能研究** ，期中主要会通过探索rocksdb源代码中的各种细节来研究其关于多核的可扩展性，一致性等相关问题。是的，这里需要注意的是，多核上的性能问题不仅仅在于可扩展性，还包括各种优化措施（原子操作，CAS，内存模型等问题）。我会重点关注rocksdb里面的设计以及和内核多核相关并会从该数据库的实现细节一直深入到操作系统的实现细节来深入研究其性能。

**如果有条件的话，也会测试一下目前系统的可扩展性等问题，定位瓶颈所在，并尝试去解决。**

## 进展罗列

1. 下载rocksdb[源代码](https://github.com/facebook/rocksdb "rocksdb")并且编译获得相关链接库以及测试用例
2. 利用CLion阅览源代码，并且尝试熟悉代码细节，并利用github上的[Wiki](https://github.com/facebook/rocksdb/wiki/RocksDB-Basics "wiki")对代码总体进行了解
3. 先利用网络资料查询rocksdb中代码有关[可扩展性](https://github.com/facebook/rocksdb/blob/189f0c27aaecdf17ae7fc1f826a423a28b77984f/docs/_posts/2014-05-14-lock.markdown)的内容

## 代码分析展示

1. Lock Scalability

   Before:

   ```c++
   SuperVersion* ColumnFamilyData::GetThreadLocalSuperVersion(
       InstrumentedMutex* db_mutex)
       SuperVersion* sv = nullptr;
   	db_mutex->Lock();
   	sv = super_verison->Ref();
   	db_mutex->Unlock();
   	return sv;
   }
   ```

   After:

   ```c++
   SuperVersion* ColumnFamilyData::GetThreadLocalSuperVersion(
       InstrumentedMutex* db_mutex) {
     void* ptr = local_sv_->Swap(SuperVersion::kSVInUse);
     assert(ptr != SuperVersion::kSVInUse);
     SuperVersion* sv = static_cast<SuperVersion*>(ptr);
     if (sv == SuperVersion::kSVObsolete ||
         sv->version_number != super_version_number_.load()) {
       RecordTick(ioptions_.statistics, NUMBER_SUPERVERSION_ACQUIRES);
       SuperVersion* sv_to_delete = nullptr;
       if (sv && sv->Unref()) {
         RecordTick(ioptions_.statistics, NUMBER_SUPERVERSION_CLEANUPS);
         db_mutex->Lock();
         sv->Cleanup();
         sv_to_delete = sv;
       } else {
         db_mutex->Lock();
       }
       sv = super_version_->Ref();
       db_mutex->Unlock();
       delete sv_to_delete;
     }
     assert(sv != nullptr);
     return sv;
   }
   ```

   ​		上面代码就是一个经典的例子，首先说明这个函数GetThreadLocalSuperVersion是主要出现在GetImpl这个函数里面的，而GetImpl则是整个rocksdb最常用的一个函数，基本上就是给一个Key，获取对应的Value的所有Get类型函数最终调用的那个读取函数，所以这里面的Scalability基本上会对整个系统有比较大的影响（尤其是当Workload集中在度的情况的时候）。因此，上面代码的Scalability非常重要。

   ​		在分析其代码之前，我简单对这一段代码做一下简要的介绍。首先是SuperVersion的用处和概念，它是RocksDB的内部概念。一个SuperVersion包括SST文件列表（一个“版本”）和在一个时间点的活动内存表列表。无论是压缩或刷新，还是一个mem表开关都会导致创建一个新的SuperVersion。旧的SuperVersion可以继续被正在进行的读取请求使用。旧的超级版本最终将被垃圾收集后不再需要。因此如果我需要读取的话，我需要更新一下到最新的SuperVersion，然后在通过SuperVersion这个结构体进行后续的访问，而使用SuperVersion也是数据库里为了并发而使用类似的多版本读写协议而制造而成的。

   ​		显然，根据上面的情况，我们可以立即Before这个对应的代码的含义了，但是很显然如果每次读都需要上个锁那基本上就是灾难性的事情，对于多核完全没有可扩展性，据测试在32个核的时候，在系统态话费的时间基本上已经是在用户态时间的2倍左右了。

   ​		紧接着，就是很自然考虑到类似到SeqLock这个技术了（刚好在ScaleFS中学到这个技术），本身SeqLock在读取之前获取一下版本号，然后在读取之后在获取版本号，获得两者之间的是否相同，相同并且为偶数则代表读成功，否则失败重试，而写操作则是前后各更新一下版本号。这里采用有些相近的做法，就是首先存一个ThreadLocal的备份，然后每次比较这个备份的版本号和全局的那个版本号是否相同，如果相同那么直接返回就结束了，然后如果不相同，则更新备份就好了，当然还涉及到对旧备份的处理。

   ​		上面的操作就能够很好的避免在HotPath下各种读操作对于锁的竞争，然而这里感觉其实有一个可以尝试的地方，那就是这里为什么要费尽心思维护一个备份的，就直接对于全局的super_version_ 使用super_version_number_ 一个SeqLock包围住就好，这样其实还能够节省各种备份的管理和开销。

   ​		然而实际上这里采用SeqLock会比较麻烦可能也是有原因的，那就是一旦对于Ref之后，我们一定要进行各种释放，否则会导致在读操作中循环被Ref了之后的旧的SuperVersion就被保留下来释放不了了，而如果处理不好的话，也可能会导致多次申请和释放。

2. 另外，虽然这个探索的并不是很完全，但是也有一些让我比较感兴趣的地方，就是里面对于内存模型的各种情况的使用。其实，在代码中，Rocksdb利用了很多C++的内存模型的部分，比如就拿上面的代码而言，

   ```c++
   sv = super_version_->Ref(); // in file ./db/column_family.cc
   
   struct SuperVersion {
     /* ... */
     private:  
       std::atomic<uint32_t> refs;
     /* ... */  
   };
   
   // in file ./db/column_family.cc
   SuperVersion* SuperVersion::Ref() {
     refs.fetch_add(1, std::memory_order_relaxed);
     return this;
   }
   
   // in file /usr/include/c++/7/atomic
   _GLIBCXX_ALWAYS_INLINE __int_type
         fetch_add(__int_type __i,
   		memory_order __m = memory_order_seq_cst) noexcept
         { return __atomic_fetch_add(&_M_i, __i, __m); }
   ```

   ​		这里其实目前让我有一些费解的是这里对于refs中的内存模型特地指定了std::memory_order_relaxed，首先在我的影响中这个其实是对于一致性有一些放松，首先是在这里为什么可行，其次我也想研究一下数据库的对于所为的内存模型以及一致性究竟有多少的要求。虽然应该都是在锁这个层面上的，但是似乎不同的地方对于一致性有不同的要求，这一点也让我非常的好奇。

   ​		多核的情况对于一致性的处理也是非常让人困惑的，我之前有所了解过Coherence和Consistency的差别以及为什么需要关注，然而目前在遇到具体的情况的时候，依然再次感到了迷茫，我觉得也应该需要更加具体的结合实际情况进行掌握了解。而且以后编码也可以更加高效，毕竟越轻松的约束条件相对性能也会更好一些，甚至某种程度上能够减少锁总线和Cache Invalidaty的情况，从而提升整体指令的Throughput.

3. 目前对于rocksdb本身（与OS无关的）可扩展性主要是他的内存存储结构，主要在这篇文章Scaling [Concurrent Log-Structured Data Stores](http://webee.technion.ac.il/~idish/ftp/clsm.pdf)提出了一个具有高可扩展性的还能够对磁盘I/O有比较好的读写Throughput。当然Rocksdb也有和[ScaleFS](https://pdos.lcs.mit.edu/papers/scalefs.pdf)类似的想法，就是延迟更新，并且在内存中也有一块用于Concurrent的内存记录（Concurrent Skip List），然后等到Sync采取刷到他的结构内容是SST。其中也类似于Oplog，则使用了上面的Log-Structured的结构进行记录和Merge。当然上面的这个结构和ScaleFS不一样的地方，在于Concurrent Log-Structured Data Stores还需要维护各种时间点的备份。

   然而这里，个人又有一点看法，因为感觉类似于[Commutor](http://sigops.org/s/conferences/sosp/2013/papers/p1-clements.pdf)，不知道这里是否可以也对一些命令进行一些Concurrent的分析，并给出各种应该并行的情况，然后找出里面的潜在的Bottleneck，并给予优化。

   另外，2017年似乎也有人进一步提出了更新的专门针对多核的结构[Concurrent Log-Structured Memoryfor Many-Core Key-Value Stores](http://www.vldb.org/pvldb/vol11/p458-merritt.pdf)也可以考虑和Rocksdb进行结合尝试一下性能提升。

## 未来的计划和想法

1. 中期进展相对主要还是停留在研究代码，并在一定程度上结合之前研究过的论文提出了一些结合的点
2. 计划寻找一些常用的baseline，然后根据上面提出来的一些idea来进行一些代码上的修改并进行测试
3. 继续研究代码，可以利用Commutor的思想，先找出里面理论上的并行情况，然后在制定测试用例，进行瓶颈的寸照

## 参考文献

因为直接使用Markdown自带的链接形式，[]() 这样的形式就直接吧需要的论文附在对应内容的后面，这样可以使用Github浏览的时候直接点击即可，更加的方便直接。