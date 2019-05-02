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

2. 

## 总结

