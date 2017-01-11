这一篇该讲CentralFreeList，CentralFreeList在TCMalloc中扮演一个承上启下的作用，它负责给上层的ThreadCache提供object，并且负责向PageHeap申请页，还是比较重要的，下面慢慢讲

结构

CentralFreeList全局有86个，对应第一篇说的86种小内存的分配策略，先看下主要结构：
```
class CentralFreeList {
  Span     empty_;          // Dummy header for list of empty spans
  Span     nonempty_;       // Dummy header for list of non-empty spans
  TCEntry tc_slots_[kMaxNumTransferEntries];
}
```
主要就这么3个数组，empty_ 和 nonempty_ 是两个Span的链表，empty保存着他从PageCache中申请过来的一批页对应的Span，每页都已经按照预先规定的大小划分成等大的object，当上层的ThreadCache向它申请一批object时，它便从nonempty_ 链表的取出第一个Span，将这个Span对应的页里剩余还没有分配出去的object分配出去，假如这个Span里所有的object都已经分配出去，那么就把这个Span从empty_ 摘掉，放到empty_ ，tc_slots_ 是缓存从ThreadCache还回来的一批object，TCEntry保存还回来的object链表的start和end，没错，上层是一批一批的申请，还的时候也是一批一批的返回，其实就是链表，主要的三个数组讲完了，现在来看下代码，先来看一下申请一批object的函数：

```
int CentralFreeList::RemoveRange(void **start, void **end, int N) {
  //这个是上层ThreadCache向本层申请内存时调用的接口，N是要申请多少个object，申请完
  //之后start和end就指向这些object的第一个和最后一个，返回的一批object其实是个链表
  ASSERT(N > 0);
  lock_.Lock();
  //先从缓存里面找，这里面存着最近被换回来的一批批object
  if (N == Static::sizemap()->num_objects_to_move(size_class_) &&
      used_slots_ > 0) {
    int slot = --used_slots_;
    ASSERT(slot >= 0);
    TCEntry *entry = &tc_slots_[slot];
    *start = entry->head;
    *end = entry->tail;
    lock_.Unlock();
    return N;
  }

  int result = 0;
  void* head = NULL;
  void* tail = NULL;
  // TODO: Prefetch multiple TCEntries?
  //这个函数就是之所以叫safe就是他在从empty_链表里取Span之前，先判断有没有，如果
  //当前已经没有可用的Span，则调用populate向PageHeap申请,返回第一个可用的object的地址
  tail = FetchFromSpansSafe();
  if (tail != NULL) {
    SLL_SetNext(tail, NULL);
    head = tail;
    result = 1;
    while (result < N) {
      // 因为需要N个object，所以不断的调用FetchFromSpans，每次获取一个object，将
      // 他们连成链表
      void *t = FetchFromSpans();
      if (!t) break;
      SLL_Push(&head, t);
      result++;
    }
  }
  lock_.Unlock();
  //最终start指向链表头，end指向链表尾
  *start = head;
  *end = tail;
  return result;
}
```
逻辑还是非常清晰的，这里重点说一个东西，返回的一批object虽然说是链表，可没有多花一个字节去存节点间的关系，类似于数组链表的实现，虽然每个object是连续的，但是每个object的前4个字节是下一个object的地址，即使object的前4个字节被用了，但当这个object分出去以后，这个字节是可以被用户覆盖掉的，当它还回来以后，再把这个object的前四个字节赋值为当前链表head对应的object的地址，就是插到链表头，然后再将head指向这个object

补充一句，上面代码里说的populate就是向PageHeap申请N个页，将这N个也对应的span插入到nonempty链表，将页划分成等大小的object，除此之外，还有两步重要的操作，第一是RegisterSizeClass，他会把在PageHeap的page_map中这个span的N个页对应的leaf节点的指针指向这个span，方便用地址映射到对应的span，第二个是CacheSizeClass，这个主要记录N个页的每个页属于86个class_size_中的哪一个，这会在用户free的时候用到，比如用户free(ptr)，通过在缓存里查找ptr对应的页属于哪个size_class，就知道该把它放回86个策略的哪个缓存链表里，另外大内存的cachesizeclass一律将每一页记录成0，free的时候只要查到ptr对应的是0就知道这个地址对应的是个大内存，不应该走小内存的归还步骤

退还一批object的实现也很简单，来看代码：
```
//start，end是object链表的头和尾，N是个数
void CentralFreeList::InsertRange(void *start, void *end, int N) {
  SpinLockHolder h(&lock_);
  // 如果N等于自己对应的size，将这批object放在缓存里，MakeCacheSpace其实就是
  // 如果缓存满了，就从里面踢一条，也就是一批object，把他们还给所属的span
  if (N == Static::sizemap()->num_objects_to_move(size_class_) &&
    MakeCacheSpace()) {
    int slot = used_slots_++;
    ASSERT(slot >=0);
    ASSERT(slot < max_cache_size_);
    TCEntry *entry = &tc_slots_[slot];
    entry->head = start;
    entry->tail = end;
    return;
  }
  //否则，直接还给所属的span
  ReleaseListToSpans(start);
}

void CentralFreeList::ReleaseListToSpans(void* start) {
  //遍历链表，一个一个还，注意数组链表的实现，虽然每个object相邻，但数组序和
  //链表序不应定相同，初次返回一批object给上层的ThreadCache肯定是一样的，但
  //这批object在ThreadCache中分出去的顺序和每个object还回来的顺序不一定一样，
  //所以从ThreadCache还回来的Object链表的链表序不一定和数组序一样，所以链表
  //关系的next值记录在每个object的前四个字节，只要知道start第一个节点，就可
  //以遍历完链表
  while (start) {
    void *next = SLL_Next(start);
    ReleaseToSpans(start);
    start = next;
  }
}

void CentralFreeList::ReleaseToSpans(void* object) {
  //找到object对应的span，PageHeap里不是有一个page_map吗？从那个里面找
  Span* span = MapObjectToSpan(object);
  ASSERT(span != NULL);
  ASSERT(span->refcount > 0);

  // If span is empty, move it to non-empty list
  //如果所属的span之前已经没有可用的object了，那么肯定会被加到empty链表，
  //现在还了一个，所以将他移回nonempty
  if (span->objects == NULL) {
    tcmalloc::DLL_Remove(span);
    tcmalloc::DLL_Prepend(&nonempty_, span);
    Event(span, 'N', 0);
  }

  // The following check is expensive, so it is disabled by default
  if (false) {
    // Check that object does not occur in list
    int got = 0;
    for (void* p = span->objects; p != NULL; p = *((void**) p)) {
      ASSERT(p != object);
      got++;
    }
    ASSERT(got + span->refcount ==
           (span->length<<kPageShift) /
           Static::sizemap()->ByteSizeForClass(span->sizeclass));
  }

  counter_++; //可用的object的数加1
  span->refcount--;  //引用数减1，当为0的时候就代表这个span完全没被用
  if (span->refcount == 0) {
    //当span的所有object都被还回来，就将这个span从empty链表去掉，还回PageHeap
    Event(span, '#', 0);
    counter_ -= ((span->length<<kPageShift) /
                 Static::sizemap()->ByteSizeForClass(span->sizeclass));
    tcmalloc::DLL_Remove(span);
    --num_spans_;

    // Release central list lock while operating on pageheap
    lock_.Unlock();
    {
      SpinLockHolder h(Static::pageheap_lock());
      Static::pageheap()->Delete(span);
    }
    lock_.Lock();
  } else {
    //将还回来的object插入到可用object链表表头
    *(reinterpret_cast<void**>(object)) = span->objects;
    span->objects = object;
  }
}
```
归还策略

我们看到CentralFreeList并没有刻意的去缓存从PageHeap中申请到的页Span，在nonempty的span链表中，只要有一个span内的所有object都还回来了，他就会把这个span还回PageHeap，没有多做缓存，其实呢，也真没必要，因为PageHeap会对这个span做缓存的

总结

CentralFreeList的实现其实也不难，不过一些细节还是值得细扣的，比如object数组链表，RegisterSizeClass，CacheSizeClass，还是挺好玩的

对了，有一个东西上一篇忘讲了，Span是用来管理一批页的，不过Span自身是怎么New出来的呢，其实用的是PageHeapAllocator，它的实现和CentralFreeList还挺像，一次申请一批页，划成等Span大小的object，构造成一个数组链表，每次用的时候分出去一个，还回来的时候插入链表头

好了，下一篇讲ThreadCache，下篇见^^
