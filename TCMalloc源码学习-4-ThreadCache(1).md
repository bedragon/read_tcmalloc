这一篇讲一讲上层的ThreadCache，第一篇已经说过，每个线程都有自己的缓存，线程间的小内存申请和释放不需要全局锁抢锁，提高了分配效率，并且ThreadCache还有比较好的内存归还策略，尽可能的做到线程间的平衡，下面就来详细说一说

线程局部缓存

怎么做到的呢，那就是pthread_key_create，来看定义：
```
NAME
       pthread_key_create - thread-specific data key creation

SYNOPSIS
       #include <pthread.h>

       int pthread_key_create(pthread_key_t *key, void (*destructor)(void*));


DESCRIPTION
       The pthread_key_create() function shall create a thread-specific data 
       key visible to all threads in the process. Key values provided by 
       pthread_key_create() are opaque objects used  to  locate  thread-spe-
       cific  data.  Although the same key value may be used by different 
       threads, the values bound to the key by pthread_setspecific() are 
       maintained on a per-thread basis and persist for the life of the 
       calling thread.

       Upon key creation, the value NULL shall be associated with the new key
       in all active threads. Upon thread creation, the value NULL shall be 
       associated with all defined keys in the new thread.

       An optional destructor function may be associated with each key value.
       At thread exit, if a key value has a non-NULL destructor pointer, and 
       the thread has a non-NULL value associated with that  key,  the
       value of the key is set to NULL, and then the function pointed to is 
       called with the previously associated value as its sole argument. The
       order of destructor calls is unspecified if more than one destruc-
       tor exists for a thread when it exits.

       If, after all the destructors have been called for all non-NULL values
       with associated destructors, there are still some non-NULL values with
       associated destructors, then  the  process  is  repeated.   If,
       after  at  least  {PTHREAD_DESTRUCTOR_ITERATIONS}  iterations of destr
       uctor calls for outstanding non-NULL values, there are still some 
       non-NULL values with associated destructors, implementations may stop
       calling destructors, or they may continue calling destructors until 
       no non-NULL values with associated destructors exist, even though this
       might result in an infinite loop.
```
来看看ThreadCreate如何用的，在初始化函数InitTSD里，通过pthread_key_create(&heap_key_, DestroyThreadCache)来定义一个pthread_key_t变量heap_key_，这样，后来每个线程在自己初始化的时候，只要通过pthread_setspecific(heap_key_, heap)就可以定义自己的局部缓存heap，然后通过pthread_getspecific(heap_key_)便可以拿到自己的局部缓存了，并且在线程退出的时候，会回调DestroyThreadCache来释放自己的heap，好了原理就是这个了，还有一个是如果内核支持TLS，就用TLS，这里还没有具体看

结构
```
class ThreadCache {
 public:
  // All ThreadCache objects are kept in a linked list (for stats collection)
  ThreadCache* next_;
  ThreadCache* prev_;
  
  FreeList      list_[kNumClasses];     // Array indexed by size-class
  
  // Allocate an object of the given size and class. The size given
  // must be the same as the size of the class in the size map.
  void* Allocate(size_t size, size_t cl);
  void Deallocate(void* ptr, size_t size_class);
  
  static pthread_key_t heap_key_;
  static ThreadCache* thread_heaps_;
  static int thread_heap_count_;
  
  static void         InitTSD();
  static ThreadCache* GetThreadHeap();
  static ThreadCache* GetCache();
  static ThreadCache* GetCacheIfPresent();
  static ThreadCache* CreateCacheIfNecessary();
  static ThreadCache* NewHeap(pthread_t tid);

  // Use only as pthread thread-specific destructor function.
  static void DestroyThreadCache(void* ptr);

  static void DeleteCache(ThreadCache* heap);
}

extern PageHeapAllocator<ThreadCache> threadcache_allocator;
```
这个就是ThreadCache的主要成员，可以看到他有很多static的变量和函数，包括上一节说到的InitTSD，每个ThreadCache对象管理每个线程的缓存，并且所有的ThreadCache串成链表中，由next和prev来记录先后顺序，可能要问了：为什么用链表，线程A为什么要知道线程B对应的ThreadCache？因为为了内存分配的平衡，需要线程之间做动态调整，比如线程A会把线程B的缓存上限调小来迫使线程B释放一部分缓存。最后，list_便是每个线程的局部缓存，同样是有86个，ThreadCache对象和Span对象的创建一样，都是通过PageHeapAllocator来创建

用户调用malloc，判读如果是小内存，它会首先执行GetCache来获取所属线程自己的ThreadCache对象，来看下实现：
```
inline ThreadCache* ThreadCache::GetCache() {
  ThreadCache* ptr = NULL;
  //如果是首次调用，则执行InitModule来初始化一堆static变量，包括CentralFreeList和PageHeap
  if (!tsd_inited_) {
    InitModule();
  } else {
  //否则，获取所属线程的局部缓存heap，也就是通过pthread_getspecific来获取
    ptr = GetThreadHeap();
  }
  //第一次肯定是还没有创建局部heap，所以上一步返回的ptr是NULL，这里创建一个heap，
  //并通过pthread_setspecific绑定局部heap
  if (ptr == NULL) ptr = CreateCacheIfNecessary();
  return ptr;
}
```
获取到ThreadCache对象之后，便可以通过Allocate和Deallocate来申请和释放小内存了，来分别看下实现：
```
//调用者do_malloc会根据用户传的usersize来判断，如果小于256k，则通过查表找到
//usersize对应的classsize索引cl和对应的实际分配size，传给Allocate
inline void* ThreadCache::Allocate(size_t size, size_t cl) {
  ASSERT(size <= kMaxSize);
  ASSERT(size == Static::sizemap()->ByteSizeForClass(cl));

  //直接在对应的缓存list里面找
  FreeList* list = &list_[cl];
  if (list->empty()) {
    //如果没有，从CentralFreeList申请，并插回list_
    return FetchFromCentralCache(cl, size);
  }
  size_ -= size;
  return list->Pop();
}

// Remove some objects of class "cl" from central cache and add to thread heap.
// On success, return the first object for immediate use; otherwise return NULL.
void* ThreadCache::FetchFromCentralCache(size_t cl, size_t byte_size) {
  FreeList* list = &list_[cl];
  ASSERT(list->empty());
  //cl对应的每次向CentralFreeList申请object个数
  const int batch_size = Static::sizemap()->num_objects_to_move(cl);
  //每次移动的object数不超过max_length
  const int num_to_move = min<int>(list->max_length(), batch_size);
  void *start, *end;
  //从CentralFreeList中取回一批object
  int fetch_count = Static::central_cache()[cl].RemoveRange(
      &start, &end, num_to_move);

  ASSERT((start == NULL) == (fetch_count == 0));
  if (--fetch_count >= 0) {
    size_ += byte_size * fetch_count;
    //插入缓存list_
    list->PushRange(fetch_count, SLL_Next(start), end);
  }

  // Increase max length slowly up to batch_size.  After that,
  // increase by batch_size in one shot so that the length is a
  // multiple of batch_size.
  //这里是对每次从CentralFreeList中拿多少个object进行动态调整，第一篇说到
  //有一个数组记录着每个cl每次从CentralFreeList中拿多少，但并不是这样，
  //max_length初始值是1，第一次拿1个，max_length加1，第二次拿两个，
  //max_length加2，直到某次拿的数目，也就是max_length >= batch_size，这
  //以后就每次拿batch_size个，max_length每次加batch_size，类似于慢启动的
  //方式，这里是一个优化
  if (list->max_length() < batch_size) {
    list->set_max_length(list->max_length() + 1);
  } else {
    // Don't let the list get too long.  In 32 bit builds, the length
    // is represented by a 16 bit int, so we need to watch out for
    // integer overflow.
    int new_length = min<int>(list->max_length() + batch_size,
                              kMaxDynamicFreeListLength);
    // The list's max_length must always be a multiple of batch_size,
    // and kMaxDynamicFreeListLength is not necessarily a multiple
    // of batch_size.
    new_length -= new_length % batch_size;
    ASSERT(new_length % batch_size == 0);
    list->set_max_length(new_length);
  }
  return start;
}
```
再来看一下Deallocate
```
inline void ThreadCache::Deallocate(void* ptr, size_t cl) {
  FreeList* list = &list_[cl];
  size_ += Static::sizemap()->ByteSizeForClass(cl);
  //max_size和当前size的差值
  ssize_t size_headroom = max_size_ - size_ - 1;

  // This catches back-to-back frees of allocs in the same size
  // class. A more comprehensive (and expensive) test would be to walk
  // the entire freelist. But this might be enough to find some bugs.
  ASSERT(ptr != list->Next());
  //插入回list
  list->Push(ptr);
  //max_length和当前list长度的差值
  ssize_t list_headroom =
      static_cast<ssize_t>(list->max_length()) - list->length();

  // There are two relatively uncommon things that require further work.
  // In the common case we're done, and in that case we need a single branch
  // because of the bitwise-or trick that follows.
  if ((list_headroom | size_headroom) < 0) {
    //如果当前list的长度大于max_length，刚开始的时候是每次1，2，3，4...个
    //从CentralFreeList中拿，max_length也每次增加同样的长度，如果此时list
    //的长度大于max_length，也就是说已经过了慢启动的状态，list每次向
    //CentralFreeList都申请batch_size个，这时有内存还回来，list过长，需要缩容
    if (list_headroom < 0) {
      //这里做了一些事情，下面具体看一下
      ListTooLong(list, cl);
    }
    	//如果当前的缓存的size大于max_size，这回收内存
    if (size_ >= max_size_) Scavenge();
  }
}

void ThreadCache::ListTooLong(FreeList* list, size_t cl) {
  const int batch_size = Static::sizemap()->num_objects_to_move(cl);
  //还batch_size个object给CentralFreeList
  ReleaseToCentralCache(list, cl, batch_size);

  // If the list is too long, we need to transfer some number of
  // objects to the central cache.  Ideally, we would transfer
  // num_objects_to_move, so the code below tries to make max_length
  // converge on num_objects_to_move.
  //感觉这里不会进来？？？需要再确认一下
  if (list->max_length() < batch_size) {
    // Slow start the max_length so we don't overreserve.
    list->set_max_length(list->max_length() + 1);
  } else if (list->max_length() > batch_size) {
    //如果连还kMaxOverages次，每次max_length都大于batch_size，
    //则将max_size减小，减小batch_size个
    // If we consistently go over max_length, shrink max_length.  If we don't
    // shrink it, some amount of memory will always stay in this freelist.
    list->set_length_overages(list->length_overages() + 1);
    if (list->length_overages() > kMaxOverages) {
      ASSERT(list->max_length() > batch_size);
      list->set_max_length(list->max_length() - batch_size);
      list->set_length_overages(0);
    }
  }
}

// Release idle memory to the central cache
void ThreadCache::Scavenge() {
  // If the low-water mark for the free list is L, it means we would
  // not have had to allocate anything from the central cache even if
  // we had reduced the free list size by L.  We aim to get closer to
  // that situation by dropping L/2 nodes from the free list.  This
  // may not release much memory, but if so we will call scavenge again
  // pretty soon and the low-water marks will be high on that call.
  //int64 start = CycleClock::Now();
  for (int cl = 0; cl < kNumClasses; cl++) {
    FreeList* list = &list_[cl];
    const int lowmark = list->lowwatermark();
    if (lowmark > 0) {
    	//lowatermark初始都为0，也就是说这个函数的第一次调用不会进到这里
      const int drop = (lowmark > 1) ? lowmark/2 : 1;
      //每个list释放lowmark一般长度的缓存给CentralFreeList
      ReleaseToCentralCache(list, cl, drop);

      // Shrink the max length if it isn't used.  Only shrink down to
      // batch_size -- if the thread was active enough to get the max_length
      // above batch_size, it will likely be that active again.  If
      // max_length shinks below batch_size, the thread will have to
      // go through the slow-start behavior again.  The slow-start is useful
      // mainly for threads that stay relatively idle for their entire
      // lifetime.
      const int batch_size = Static::sizemap()->num_objects_to_move(cl);
      if (list->max_length() > batch_size) {
        //如果当前max_length大于batch_size，设置max_length为差值和batch_size的最小值
        list->set_max_length(
            max<int>(list->max_length() - batch_size, batch_size));
      }
    }
    //设置lowwatermark为当前list长度
    list->clear_lowwatermark();
  }
  IncreaseCacheLimit();
}

void ThreadCache::IncreaseCacheLimit() {
  SpinLockHolder h(Static::pageheap_lock());
  IncreaseCacheLimitLocked();
}

void ThreadCache::IncreaseCacheLimitLocked() {
  //unclaimed初始值是32M，也就是说所有每个线程一共可以缓存
  //32M(好像不是硬上限)，如果缓存空间还没用完，每次将
  //max_size_增加kStealAmount即可
  if (unclaimed_cache_space_ > 0) {
    // Possibly make unclaimed_cache_space_ negative.
    unclaimed_cache_space_ -= kStealAmount;
    max_size_ += kStealAmount;
    return;
  }
  // Don't hold pageheap_lock too long.  Try to steal from 10 other
  // threads before giving up.  The i < 10 condition also prevents an
  // infinite loop in case none of the existing thread heaps are
  // suitable places to steal from.
  // 如果缓存空间已经用完，则需要从其他线程去偷一些，
  //尝试偷10次，遍历其他线程的ThreadCache
  for (int i = 0; i < 10;
       ++i, next_memory_steal_ = next_memory_steal_->next_) {
    // Reached the end of the linked list.  Start at the beginning.
    if (next_memory_steal_ == NULL) {
      ASSERT(thread_heaps_ != NULL);
      next_memory_steal_ = thread_heaps_;
    }
    //其他线程的max_size_小于kMin，不偷
    if (next_memory_steal_ == this ||
        next_memory_steal_->max_size_ <= kMinThreadCacheSize) {
      continue;
    }
    //偷：减少其他线程的max_size_ ，增加给自己，其他线程的max_size减少
    //后，会在Deallocate最后触发Scavenge()，还一些给CentralFreeList
    next_memory_steal_->max_size_ -= kStealAmount;
    max_size_ += kStealAmount;

    next_memory_steal_ = next_memory_steal_->next_;
    return;
  }
}
```
可以看到，真正Allocate和Deallocate不难，难的是Deallocate进行内存归还的策略，主要靠max_length和max_size来触发，并且max_size可以由其他线程来改，确保每个线程不会缓存太多，假设用户陆陆续续申请了加起来有100M的小内存，那么在陆陆续续释放超过32M的时候，由于当前已经缓存了32M，所以会触发Scavenge，还回去一些，这里有一个问题是，假如不是自己触发的，而是因为其他线程偷了一些，修改了本线程的max_size，这里是不是只要这个线程再不申请和归还内存，就不会进到判断是否触发scavenge的逻辑，也就是说线程缓存的内存一直不释放，当然，这个值不会太多，否则当初还的时候自己会触发scavenge的，这是自己的理解，不知道对不对

总结

ThreadCache应该是TCMalloc里比较复杂的部分了，复杂的不是实现，而是内存申请和归还的策略，感觉自己理解的还不够充分，先写这么多，以后有新的理解了再加，下篇讲讲用户接口，下篇见^^
