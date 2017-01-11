线程局部缓存
tcmalloc采用线程局部存储技术为每一个线程创建一个ThreadCache，所有这些ThreadCache通过链表串起来。
线程局部缓存有两种实现：
1. 静态局部缓存，通过__thread关键字定义一个静态变量。
2. 动态局部缓存，通过pthread_key_create,pthread_setspecific,pthread_getspecific来实现。
静态局部缓存的优点是设置和读取的速度非常快，比动态方式快很多，但是也有它的缺点。

主要有如下两个缺点：

1. 静态缓存在线程结束时没有办法清除。
2. 不是所有的操作系统都支持。

ThreadCache局部缓存的实现
tcmalloc采用的是动态局部缓存，但同时检测系统是否支持静态方式，如果支持那么同时保存一份拷贝，方便快速读取。
```
// If TLS is available, we also store a copy of the per-thread object  
  // in a __thread variable since __thread variables are faster to read  
  // than pthread_getspecific().  We still need pthread_setspecific()  
  // because __thread variables provide no way to run cleanup code when  
  // a thread is destroyed.  
  // We also give a hint to the compiler to use the "initial exec" TLS  
  // model.  This is faster than the default TLS model, at the cost that  
  // you cannot dlopen this library.  (To see the difference, look at  
  // the CPU use of __tls_get_addr with and without this attribute.)  
  // Since we don't really use dlopen in google code -- and using dlopen  
  // on a malloc replacement is asking for trouble in any case -- that's  
  // a good tradeoff for us.  
#ifdef HAVE_TLS  
  static __thread ThreadCache* threadlocal_heap_  
# ifdef HAVE___ATTRIBUTE__  
   __attribute__ ((tls_model ("initial-exec")))  
# endif  
   ;  
#endif  
  
  
  // Thread-specific key.  Initialization here is somewhat tricky  
  // because some Linux startup code invokes malloc() before it  
  // is in a good enough state to handle pthread_keycreate().  
  // Therefore, we use TSD keys only after tsd_inited is set to true.  
  // Until then, we use a slow path to get the heap object.  
  static bool tsd_inited_;  
  static pthread_key_t heap_key_;  
```
尽管在编译器和连接器层面可以支持TLS，但是操作系统未必支持，因此需要实时的检查系统是否支持。主要是通过手动方式标识一些不支持的操作系统，代码如下：
thread_cache.h
```
// Even if we have support for thread-local storage in the compiler  
// and linker, the OS may not support it.  We need to check that at  
// runtime.  Right now, we have to keep a manual set of "bad" OSes.  
#if defined(HAVE_TLS)  
extern bool kernel_supports_tls;   // defined in thread_cache.cc  
void CheckIfKernelSupportsTLS();  
inline bool KernelSupportsTLS() {  
  return kernel_supports_tls;  
}  
#endif    // HAVE_TLS  
thread_cache.cc  
#if defined(HAVE_TLS)  
bool kernel_supports_tls = false;      // be conservative  
# if defined(_WIN32)    // windows has supported TLS since winnt, I think.  
    void CheckIfKernelSupportsTLS() {  
      kernel_supports_tls = true;  
    }  
# elif !HAVE_DECL_UNAME    // if too old for uname, probably too old for TLS  
    void CheckIfKernelSupportsTLS() {  
      kernel_supports_tls = false;  
    }  
# else  
#   include <sys/utsname.h>    // DECL_UNAME checked for <sys/utsname.h> too  
    void CheckIfKernelSupportsTLS() {  
      struct utsname buf;  
      if (uname(&buf) != 0) {   // should be impossible  
        Log(kLog, __FILE__, __LINE__,  
            "uname failed assuming no TLS support (errno)", errno);  
        kernel_supports_tls = false;  
      } else if (strcasecmp(buf.sysname, "linux") == 0) {  
        // The linux case: the first kernel to support TLS was 2.6.0  
        if (buf.release[0] < '2' && buf.release[1] == '.')    // 0.x or 1.x  
          kernel_supports_tls = false;  
        else if (buf.release[0] == '2' && buf.release[1] == '.' &&  
                 buf.release[2] >= '0' && buf.release[2] < '6' &&  
                 buf.release[3] == '.')                       // 2.0 - 2.5  
          kernel_supports_tls = false;  
        else  
          kernel_supports_tls = true;  
      } else if (strcasecmp(buf.sysname, "CYGWIN_NT-6.1-WOW64") == 0) {  
        // In my testing, this version of cygwin, at least, would hang  
        // when using TLS.  
        kernel_supports_tls = false;  
      } else {        // some other kernel, we'll be optimisitic  
        kernel_supports_tls = true;  
      }  
      // TODO(csilvers): VLOG(1) the tls status once we support RAW_VLOG  
    }  
#  endif  // HAVE_DECL_UNAME  
#endif    // HAVE_TLS  
```
Thread Specific Key初始化
接下来看看每一个局部缓存是如何创建的。首先看看heap_key_的创建，它在InitTSD函数中
```
void ThreadCache::InitTSD() {  
  ASSERT(!tsd_inited_);  
  perftools_pthread_key_create(&heap_key_, DestroyThreadCache);  
  tsd_inited_ = true;  
  
  
#ifdef PTHREADS_CRASHES_IF_RUN_TOO_EARLY  
  // We may have used a fake pthread_t for the main thread.  Fix it.  
  pthread_t zero;  
  memset(&zero, 0, sizeof(zero));  
  SpinLockHolder h(Static::pageheap_lock());  
  for (ThreadCache* h = thread_heaps_; h != NULL; h = h->next_) {  
    if (h->tid_ == zero) {  
      h->tid_ = pthread_self();  
    }  
  }  
#endif  
}  
```
该函数在TCMallocGuard的构造函数中被调用。TCMallocGuard类的声明和定义分别在tcmalloc_guard.h和tcmalloc.cc文件中。
```
class TCMallocGuard {  
 public:  
  TCMallocGuard();  
  ~TCMallocGuard();  
};  
// The constructor allocates an object to ensure that initialization  
// runs before main(), and therefore we do not have a chance to become  
// multi-threaded before initialization.  We also create the TSD key  
// here.  Presumably by the time this constructor runs, glibc is in  
// good enough shape to handle pthread_key_create().  
//  
// The constructor also takes the opportunity to tell STL to use  
// tcmalloc.  We want to do this early, before construct time, so  
// all user STL allocations go through tcmalloc (which works really  
// well for STL).  
//  
// The destructor prints stats when the program exits.  
static int tcmallocguard_refcount = 0;  // no lock needed: runs before main()  
TCMallocGuard::TCMallocGuard() {  
  if (tcmallocguard_refcount++ == 0) {  
#ifdef HAVE_TLS    // this is true if the cc/ld/libc combo support TLS  
    // Check whether the kernel also supports TLS (needs to happen at runtime)  
    tcmalloc::CheckIfKernelSupportsTLS();  
#endif  
    ReplaceSystemAlloc();    // defined in libc_override_*.h  
    tc_free(tc_malloc(1));  
    ThreadCache::InitTSD();  
    tc_free(tc_malloc(1));  
    // Either we, or debugallocation.cc, or valgrind will control memory  
    // management.  We register our extension if we're the winner.  
#ifdef TCMALLOC_USING_DEBUGALLOCATION  
    // Let debugallocation register its extension.  
#else  
    if (RunningOnValgrind()) {  
      // Let Valgrind uses its own malloc (so don't register our extension).  
    } else {  
      MallocExtension::Register(new TCMallocImplementation);  
    }  
#endif  
  }  
}  
  
  
TCMallocGuard::~TCMallocGuard() {  
  if (--tcmallocguard_refcount == 0) {  
    const char* env = getenv("MALLOCSTATS");  
    if (env != NULL) {  
      int level = atoi(env);  
      if (level < 1) level = 1;  
      PrintStats(level);  
    }  
  }  
}  
#ifndef WIN32_OVERRIDE_ALLOCATORS  
static TCMallocGuard module_enter_exit_hook;  
#endif  
```
线程局部缓存Cache的创建和关联
接下来看如何创建各个线程的ThreadCache的创建。我们看GetCache代码，该代码在do_malloc中被调用。
```
inline ThreadCache* ThreadCache::GetCache() {  
  ThreadCache* ptr = NULL;  
  if (!tsd_inited_) {  
    InitModule();  
  } else {  
    ptr = GetThreadHeap();  
  }  
  if (ptr == NULL) ptr = CreateCacheIfNecessary();  
  return ptr;  
}  
void ThreadCache::InitModule() {  
  SpinLockHolder h(Static::pageheap_lock());  
  if (!phinited) {  
    Static::InitStaticVars();  
    threadcache_allocator.Init();  
    phinited = 1;  
  }  
}  
```
该函数首先判断tsd_inited_是否为true，该变量在InitTSD中被设置为true。那么首次调用GetCache时tsd_inited_肯定为false，这时就InitModule就会被调用。InitModule函数主要是来进行系统的内存分配器初始化。如果tsd_inited_已经为true了，那么线程的thread specific就可以使用了，GetThreadHeap就是通过heap_key_查找当前线程的ThreadCache. 如果ptr为NULL，那么CreateCacheIfNecessary就会被调用，该函数来创建ThreadCache。
```
ThreadCache* ThreadCache::CreateCacheIfNecessary() {  
  // Initialize per-thread data if necessary  
  ThreadCache* heap = NULL;  
  {  
    SpinLockHolder h(Static::pageheap_lock());  
    // On some old glibc's, and on freebsd's libc (as of freebsd 8.1),  
    // calling pthread routines (even pthread_self) too early could  
    // cause a segfault.  Since we can call pthreads quite early, we  
    // have to protect against that in such situations by making a  
    // 'fake' pthread.  This is not ideal since it doesn't work well  
    // when linking tcmalloc statically with apps that create threads  
    // before main, so we only do it if we have to.  
#ifdef PTHREADS_CRASHES_IF_RUN_TOO_EARLY  
    pthread_t me;  
    if (!tsd_inited_) {  
      memset(&me, 0, sizeof(me));  
    } else {  
      me = pthread_self();  
    }  
#else  
    const pthread_t me = pthread_self();  
#endif  
  
  
    // This may be a recursive malloc call from pthread_setspecific()  
    // In that case, the heap for this thread has already been created  
    // and added to the linked list.  So we search for that first.  
    for (ThreadCache* h = thread_heaps_; h != NULL; h = h->next_) {  
      if (h->tid_ == me) {  
        heap = h;  
        break;  
      }  
    }  
  
  
    if (heap == NULL) heap = NewHeap(me);  
  }  
  
  
  // We call pthread_setspecific() outside the lock because it may  
  // call malloc() recursively.  We check for the recursive call using  
  // the "in_setspecific_" flag so that we can avoid calling  
  // pthread_setspecific() if we are already inside pthread_setspecific().  
  if (!heap->in_setspecific_ && tsd_inited_) {  
    heap->in_setspecific_ = true;  
    perftools_pthread_setspecific(heap_key_, heap);  
#ifdef HAVE_TLS  
    // Also keep a copy in __thread for faster retrieval  
    threadlocal_heap_ = heap;  
#endif  
    heap->in_setspecific_ = false;  
  }  
  return heap;  
}  
ThreadCache* ThreadCache::NewHeap(pthread_t tid) {  
  // Create the heap and add it to the linked list  
  ThreadCache *heap = threadcache_allocator.New();  
  heap->Init(tid);  
  heap->next_ = thread_heaps_;  
  heap->prev_ = NULL;  
  if (thread_heaps_ != NULL) {  
    thread_heaps_->prev_ = heap;  
  } else {  
    // This is the only thread heap at the momment.  
    ASSERT(next_memory_steal_ == NULL);  
    next_memory_steal_ = heap;  
  }  
  thread_heaps_ = heap;  
  thread_heap_count_++;  
  return heap;  
}  
```
CreateIfNecessary创建一个ThreadCache对象，并且将该对象与当前线程的pthread_key_关联，同时添加到ThreadCache链表的头部。这里有个特别的情况需要说明，首次调用malloc所创建的ThreadCache对象没有和pthread_key_关联，只是添加到了ThreadCache链表中去了，程序可能还会在tsd_inited_为true之前多次调用malloc，也就会多次进入CreateCacheIfNecessary函数，这时函数中会去遍历ThreadCache链表，发现当前线程已经创建好的ThreadCache对象。
总结
1. 线程局部数据的实现可分为静态和动态两种。
2. tcmalloc以动态线程局部数据实现为主，静态为辅。
3. 通过全局静态对象的构造函数来创建Thread Specific Key。
4. 线程首次调用GetCache函数会触发线程专属的ThreadCache对象创建并与pthread_key_关联，添加到ThreadCache链表。
