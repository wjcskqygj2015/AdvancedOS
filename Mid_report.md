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

   上面代码就是一个经典的例子，

2. 

## 总结

