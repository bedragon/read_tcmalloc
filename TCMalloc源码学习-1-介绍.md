整体介绍

TCMalloc的设计让每个线程有自己的thread_cache，当线程自己的thread_cache不足时，它会向全局的central_freelist申请，当central_freelist也不够的时候，会向全局page_heap申请，page_page则是管理全部已申请到的内存，当他不够用是才会向系统申请，可以看到内存申请的顺序是thread_cache->central_freelist->page_heap->os，当然逐级申请的时候也不是用多少就申请多少，具体分配多少TCMalloc根据申请的大小还有会自己的策略，稍后介绍，当然内存的归还也是这么一个顺序，每层都不是用完了马上就向下还，而是自己缓存起来，这样下次用的时候不用再向下去借，提高速度，当然为了避免某些恶心的线程申请完大内存并归还后cache到自己的地盘，占着茅肯不拉屎，TCMalloc在内存的归还也有自己的策略，总之设计的还是挺合理的，详细的内容后面再慢慢说。另外，每个线程管理自己的内存，线程内部的分配加自己的锁，线程间无竞争，提高了分配速度，只有在thread_cache不够的时向全局central_freelist以及page_heap申请时才需要加全局锁，这相比于glibc的malloc优化还是大大滴

小内存分配策略

先抛出一个问题，既然TCMalloc是按照thread_cache->central_freelist->page_heap这样的逐级申请来分配，那么每级每次该申请多少呢，比如用户向thread_cache申请168字节，thread_cache应该分配多少？是168字节吗？然后假如thread_cache自己不够，他需要向central_freelist申请，这该分配多少？168字节？同样，如果central_freelist也不够，它向page_heap申请，这又该分配多少呢？168字节?

答案当然不可能是每级都按168字节来分配，TCMalloc的策略就是将小内存的分配策略提前定义好，只要用户申请的内存少于256K（小内存申请），直接通过查表得到用时申请size对应的每一级该分配的大小，那么这个表是如何定义的呢？

极端点的办法就是把1~256k的每一个字节都精确预先定好策略，这样的话每一级都需要一个元素个数为256K的数组来保存，这个是最简单的办法了，不过没必要，其实用户向thread_cache申请1个字节的大小和申请7个字节的大小，由于对齐的缘故，它返回给用户的都是8字节，所以数组大小可以精简，怎么精简呢，TCMalloc是这样做的，先看看它如何定义对齐，下面是一些常量定义：
```
static const size_t kPageShift  = 13;
static const size_t kPageSize   = 1 << kPageShift; //页大小-8k
static const size_t kMaxSize    = 256 * 1024; //最大的小内存-256k
static const size_t kAlignment  = 8; //8字节对齐
static const size_t kNumClasses = 86;
static const int kMaxSmallSize = 1024;
static const size_t kClassArraySize = ((kMaxSize + 127 + (120 << 7)) >> 7) + 1;
```
对齐的算法很有意思：
```
int AlignmentForSize(size_t size) {
  int alignment = kAlignment;
  if (size > kMaxSize) {
    // Cap alignment at kPageSize for large sizes.
    alignment = kPageSize;
  } else if (size >= 128) {
    // Space wasted due to alignment is at most 1/8, i.e., 12.5%.
    alignment = (1 << LgFloor(size)) / 8;
  } else if (size >= 16) {
    // We need an alignment of at least 16 bytes to satisfy
    // requirements for some SSE types.
    alignment = 16;
  }
  // Maximum alignment allowed is page size alignment.
  if (alignment > kPageSize) {
    alignment = kPageSize;
  }
  return alignment;
}
```
如果size>256k,那就是按页大小（8k）来对齐，如果128<=size<=256k字节，则是按照(1«LgFloor(size))/8来对齐，LgFloor其实就是取size二进制最高位1的位置，如LgFloor(256)=8,所以如果size是256字节，则通过上面的公式，是按32字节来对齐的，如果16<=size<128则是按16字节对齐的，如果size<16字节，则按8字节对齐

通过将256k的每种字节大小向这些对齐档位划分，最终确定下来86个策略，任何<=256k的size申请，每一级在需要的时候都可以查到对应该size在本级的分配策略，具体是怎么个策略呢？每一级86个链表，分别对应86中策略下相同大小的object，比如thread_cache,它有86个链表，每一个链表各不相同，存着大小不同的内存块（对应不同的分配策略），不过链表内部的内存块大小则相同，每次用的话分配一个出去就好，假如某一策略下的链表已全部分配，则需要向central_freelist对应策略的链表申请（86个central_freelist), 假如central_list某一策略下的链表也用完了，则他直接向page_heap申请，page_heap会根据central_freelist自身对应的策略，找出给该central_freelist分配的策略

好了，那么再详细点，每级之间是个什么策略呢？thread_cache向用户分配的策略就：你申请N个字节，我通过查表算出我该实际给你M个字节（考虑对齐的原因，M>=N），然后再从M对应的链表中取出来一个M大小的内存给用户，假如这个链表为空，它要向对应M大小的central_freelist申请，对应M大小的central_freelist收到请求后，并不只是给一个M块，而是继续通过查表算出一次该给多少个M块，然后一同给thread_cache，假如对应M大小的central_list块用完了，他需要向page_heap申请，向page_heap申请的单位就是页了，page_heap通过查表，算出应该每次给对应M大小的central_freelist多少个页，然后拿出这么多页，返回给central_freelist,central_freelist会将这些也划分成等M大小的块

OK，策略讲完啦，那么再具体点，一直说的表长什么样呢，其实就是三个数组
```
static const size_t kNumClasses = 86;
size_t class_to_size_[kNumClasses];
int num_objects_to_move_[kNumClasses];
size_t class_to_pages_[kNumClasses];
```
class_to_size_就是thread_cache用的，用来查实际该给用户返回多大内存，num_objects_to_move_是central_freelist用的，用来查每次给对应thread_cache分配多少个object，class_to_pages_是page_heap用的，用来查每次给不同central_freelist分多少个页

是不是更清晰了，那么具体数组的值又是多少呢？ 这些值是在SizeMap的init中算出来的，我把init的逻辑抽出来打印了一下，得到这样的结果，数据较多，截取部分：

index | class_to_size_ | num_objects_to_move_ | class_to_pages_
---|---|---|---
0 | 0 | 0 | 0
1 | 8 | 32 | 1
12 | 176 | 32 | 1
32 | 1152 | 32 | 2
85 | 262144 | 2 | 32
> 注意现在新版num_objects_to_move_最大是512 为什么是这个现在没弄懂，也许这个值以后还会变
这里有一点要特殊说明一下，index=32的策略，objectsize=1152，可是大家注意到，pages_=2,也就是说central_freelist向page_heap每次申请2页，可是这里其实1页也是够的啊，为什么要2页呢，关于页数的确定有两个因素，一是保证移动的页数按照objectsize切分不小于num_objects_to_move/4，1152对应的num_objects_to_move是32，而一页最多只能分配7个object，不够32/4=8个，所以需要两页，二是因为TCMalloc为了保证尽可能少的空间浪费，假设页数为N，需要保证((N * 8k) % objectsize / （N * 8k）) > 12.5%

分配策略已经压缩在86种，可是用户传过来的值怎么映射到这86种呢，每一个用户申请的size(<=256k)通过ClassIndex(size)都会算出一个值，这些值的范围是0~2168, ClassIndex算法如下，大致就是<=1024字节的按照8字节对齐，>1024字节的按照128字节对齐，所以256k一共有2169种，TCMalloc通过class_array_来保存着2169中索引向86种策略的映射
```
static inline int ClassIndex(int s) {
  ASSERT(0 <= s);
  ASSERT(s <= kMaxSize);
  const bool big = (s > kMaxSmallSize);
  const int add_amount = big ? (127 + (120<<7)) : 7;
  const int shift_amount = big ? 7 : 3;
  return (s + add_amount) >> shift_amount;
}
```
好啦，到此，关键的4个数组都已经初始化完毕！

得到这4个数组，小内存的分配策略就全部出来啦，一起走一遍分配流程：

首先通过用户申请的size(<=256k)使用ClassIndex算出的index1，再通过在class_array_中index1的位置找到这个size对应的分配策略索引index2，index2就是剩下的三个数组的索引，举个例子： 假如申请168字节的内存，首先通过ClassIndex算出index1=21，然后通过class_array_[21]得到index2=12，通过index2在三个数组中依次得到176，32，1，代表的意思就是thread_cache从每个节点大小为176（下标索引为12）的链表中取出一个返回给用户，如果thread_cache不够，需要从central_freelist[12]中取，则每次取32个大小为176字节的块，如果central_freelist[12]也不够,则向page_heap申请1页并把1页按照176字节为单位切分成块，返回32个给thread_cache，好了，总结：

用户向thread_cache申请168字节
假设此时thread_cache为空，通过查表，得知168字节的申请应该向对应176字节的central_freelist[12]申请，并且每次申请32个object（每个176字节）
假设此时central_freelist[12]也为空，通过查表，得知自己每次应该向page_heap申请1页，也就是8K，然后把8K分成以176字节为单位的块，并且返回32个块给thread_cache,剩下的留着备用
thread_cache从central_freelist拿到了32个object，把其中一个返回给用户，剩下的31个留着备用
大内存的分配

大内存的分配不走thread_cache和central_freelist，直接从page_heap申请，这个比较简单，后面的章节再说

总结

尼玛，这一章我本来只是打算简单介绍一下TCMalloc的，小内存、大内存的分配点到为止，可是这里的细节不说透，后面代码分析便没有意义，啰啰嗦嗦的算是说完啦，后面我会从下向上重点介绍page_heap，central_freelist，thread_cache的实现，然后再从用户接口串一下整体，下篇见^^
