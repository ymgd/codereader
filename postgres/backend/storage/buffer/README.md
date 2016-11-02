# Dir Overview:

/src/backend/storage/buffer/

* localbuf.c: **593** lines
* freelist.c: **687** lines
* buf_init.c: **205** lines
* buf_table.c: **163** lines
* bufmgr.c: **4296** lines


The topological sort result is :

buf_init.c  —> buf_table.c —> freelist.c —> bufmgr.c —> localbuf.c

I think this order will help saving a lot of time and reinforce understanding of the src.^_^




Notes About Shared Buffer Access Rules
======================================

共享磁盘缓冲区有两种独立的存取控制机制：引用计数（即pin counts）和缓冲区内容锁。（事实上，还有另一种层级的存取控制：一个必须在某一个relation上持有适当类型的锁，在它可以合法的存取任意属于这个关系的page前。这种relation-level locks不在此讨论）。

* Pins（针）：每个进程在被允许对缓冲区做任何事之前，必须对该缓冲区持有一个pin（递增它的引用计数）。一个unpinned缓冲区需要被回收，并且被另一个页在任何时候重用。所以touch it是不安全的。 通常一个pin通过 ReadBuffer获取，通过 ReleaseBuffer释放。 对单个backend，并发地对一个page添加多个pin是可以的也是常见的行为。缓冲管理器可以高效地处理这些。长时间地持有一个品也被认为是允许的。例如，序列扫描对当前页一直持有pin，知道它处理完了该页的所有tuple（元组），这可能需要很长时间，如果这趟扫描（scan）是outer scan of a join。同样的，btree索引扫描持有当前索引页的品。这是合法的因为正常的操作从来不会等到一页的pin count将为0。（任何事件，可能需要做这种等待，相反由等待获取关系级别的锁，这就是为什么你最好首先持有一个pin再说）。然而， Pins不能被跨事务边界地持有 。



* 缓冲区内容锁：有两种此类型的lock，共享的和独占的，顾名思义：多个后台进程可以对同一个缓冲区持有多个shared locks，但是一个eclusive lock组织任何其他backend持有shared或者exclusive的lock。（因此，这可以称作“读/写锁”，这让我想起了操作系统中学到的reader/writer）。这些锁都被设计为短时间的：它们不能被长时间持有。  缓冲区锁通过**LockBuffer（）**来获取和释放。单个后台尝试对同一个缓冲区持有多个lock是行不通的。任何想要lock缓冲区的进程都必须先pin这个buffer。



Buffer access rules:

1. To scan a page for tuples, one must hold a pin and either shared or
   exclusive content lock.  To examine the commit status (XIDs and status bits)
   of a tuple in a shared buffer, one must likewise hold a pin and either shared
   or exclusive lock.

2. Once one has determined that a tuple is interesting (visible to the
   current transaction) one may drop the content lock, yet continue to access
   the tuple's data for as long as one holds the buffer pin.  This is what is
   typically done by heap scans, since the tuple returned by heap_fetch
   contains a pointer to tuple data in the shared buffer.  Therefore the
   tuple cannot go away while the pin is held (see rule #5).  Its state could
   change, but that is assumed not to matter after the initial determination
   of visibility is made.

3. To add a tuple or change the xmin/xmax fields of an existing tuple,
   one must hold a pin and an exclusive content lock on the containing buffer.
   This ensures that no one else might see a partially-updated state of the
   tuple while they are doing visibility checks.

4. It is considered OK to update tuple commit status bits (ie, OR the
   values HEAP_XMIN_COMMITTED, HEAP_XMIN_INVALID, HEAP_XMAX_COMMITTED, or
   HEAP_XMAX_INVALID into t_infomask) while holding only a shared lock and
   pin on a buffer.  This is OK because another backend looking at the tuple
   at about the same time would OR the same bits into the field, so there
   is little or no risk of conflicting update; what's more, if there did
   manage to be a conflict it would merely mean that one bit-update would
   be lost and need to be done again later.  These four bits are only hints
   (they cache the results of transaction status lookups in pg_clog), so no
   great harm is done if they get reset to zero by conflicting updates.
   Note, however, that a tuple is frozen by setting both HEAP_XMIN_INVALID
   and HEAP_XMIN_COMMITTED; this is a critical update and accordingly requires
   an exclusive buffer lock (and it must also be WAL-logged).

5. To physically remove a tuple or compact free space on a page, one
   must hold a pin and an exclusive lock, *and* observe while holding the
   exclusive lock that the buffer's shared reference count is one (ie,
   no other backend holds a pin).  If these conditions are met then no other
   backend can perform a page scan until the exclusive lock is dropped, and
   no other backend can be holding a reference to an existing tuple that it
   might expect to examine again.  Note that another backend might pin the
   buffer (increment the refcount) while one is performing the cleanup, but
   it won't be able to actually examine the page until it acquires shared
   or exclusive content lock.


Rule #5 only affects VACUUM operations.  Obtaining the
necessary lock is done by the bufmgr routine LockBufferForCleanup().
It first gets an exclusive lock and then checks to see if the shared pin
count is currently 1.  If not, it releases the exclusive lock (but not the
caller's pin) and waits until signaled by another backend, whereupon it
tries again.  The signal will occur when UnpinBuffer decrements the shared
pin count to 1.  As indicated above, this operation might have to wait a
good while before it acquires lock, but that shouldn't matter much for
concurrent VACUUM.  The current implementation only supports a single
waiter for pin-count-1 on any particular shared buffer.  This is enough
for VACUUM's use, since we don't allow multiple VACUUMs concurrently on a
single relation anyway.


Buffer Manager's Internal Locking
---------------------------------

Before PostgreSQL 8.1, all operations of the shared buffer manager itself
were protected by a single system-wide lock, the BufMgrLock, which
unsurprisingly proved to be a source of contention.  The new locking scheme
avoids grabbing system-wide exclusive locks in common code paths.  It works
like this:

* There is a system-wide LWLock, the BufMappingLock, that notionally
  protects the mapping from buffer tags (page identifiers) to buffers.
  (Physically, it can be thought of as protecting the hash table maintained
  by buf_table.c.)  To look up whether a buffer exists for a tag, it is
  sufficient to obtain share lock on the BufMappingLock.  Note that one
  must pin the found buffer, if any, before releasing the BufMappingLock.
  To alter the page assignment of any buffer, one must hold exclusive lock
  on the BufMappingLock.  This lock must be held across adjusting the buffer's
  header fields and changing the buf_table hash table.  The only common
  operation that needs exclusive lock is reading in a page that was not
  in shared buffers already, which will require at least a kernel call
  and usually a wait for I/O, so it will be slow anyway.

* As of PG 8.2, the BufMappingLock has been split into NUM_BUFFER_PARTITIONS
  separate locks, each guarding a portion of the buffer tag space.  This allows
  further reduction of contention in the normal code paths.  The partition
  that a particular buffer tag belongs to is determined from the low-order
  bits of the tag's hash value.  The rules stated above apply to each partition
  independently.  If it is necessary to lock more than one partition at a time,
  they must be locked in partition-number order to avoid risk of deadlock.

* A separate system-wide LWLock, the BufFreelistLock, provides mutual
  exclusion for operations that access the buffer free list or select
  buffers for replacement.  This is always taken in exclusive mode since
  there are no read-only operations on those data structures.  The buffer
  management policy is designed so that BufFreelistLock need not be taken
  except in paths that will require I/O, and thus will be slow anyway.
  (Details appear below.)  It is never necessary to hold the BufMappingLock
  and the BufFreelistLock at the same time.

* Each buffer header contains a spinlock that must be taken when examining
  or changing fields of that buffer header.  This allows operations such as
  ReleaseBuffer to make local state changes without taking any system-wide
  lock.  We use a spinlock, not an LWLock, since there are no cases where
  the lock needs to be held for more than a few instructions.

Note that a buffer header's spinlock does not control access to the data
held within the buffer.  Each buffer header also contains an LWLock, the
"buffer content lock", that *does* represent the right to access the data
in the buffer.  It is used per the rules above.

There is yet another set of per-buffer LWLocks, the io_in_progress locks,
that are used to wait for I/O on a buffer to complete.  The process doing
a read or write takes exclusive lock for the duration, and processes that
need to wait for completion try to take shared locks (which they release
immediately upon obtaining).  XXX on systems where an LWLock represents
nontrivial resources, it's fairly annoying to need so many locks.  Possibly
we could use per-backend LWLocks instead (a buffer header would then contain
a field to show which backend is doing its I/O).


Normal Buffer Replacement Strategy
----------------------------------

There is a "free list" of buffers that are prime candidates for replacement.
In particular, buffers that are completely free (contain no valid page) are
always in this list.  We could also throw buffers into this list if we
consider their pages unlikely to be needed soon; however, the current
algorithm never does that.  The list is singly-linked using fields in the
buffer headers; we maintain head and tail pointers in global variables.
(Note: although the list links are in the buffer headers, they are
considered to be protected by the BufFreelistLock, not the buffer-header
spinlocks.)  To choose a victim buffer to recycle when there are no free
buffers available, we use a simple clock-sweep algorithm, which avoids the
need to take system-wide locks during common operations.  It works like
this:

Each buffer header contains a usage counter, which is incremented (up to a
small limit value) whenever the buffer is pinned.  (This requires only the
buffer header spinlock, which would have to be taken anyway to increment the
buffer reference count, so it's nearly free.)

The "clock hand" is a buffer index, nextVictimBuffer, that moves circularly
through all the available buffers.  nextVictimBuffer is protected by the
BufFreelistLock.

The algorithm for a process that needs to obtain a victim buffer is:

1. Obtain BufFreelistLock.

2. If buffer free list is nonempty, remove its head buffer.  If the buffer
   is pinned or has a nonzero usage count, it cannot be used; ignore it and
   return to the start of step 2.  Otherwise, pin the buffer, release
   BufFreelistLock, and return the buffer.

3. Otherwise, select the buffer pointed to by nextVictimBuffer, and
   circularly advance nextVictimBuffer for next time.

4. If the selected buffer is pinned or has a nonzero usage count, it cannot
   be used.  Decrement its usage count (if nonzero) and return to step 3 to
   examine the next buffer.

5. Pin the selected buffer, release BufFreelistLock, and return the buffer.

(Note that if the selected buffer is dirty, we will have to write it out
before we can recycle it; if someone else pins the buffer meanwhile we will
have to give up and try another buffer.  This however is not a concern
of the basic select-a-victim-buffer algorithm.)


Buffer Ring Replacement Strategy
---------------------------------

When running a query that needs to access a large number of pages just once,
such as VACUUM or a large sequential scan, a different strategy is used.
A page that has been touched only by such a scan is unlikely to be needed
again soon, so instead of running the normal clock sweep algorithm and
blowing out the entire buffer cache, a small ring of buffers is allocated
using the normal clock sweep algorithm and those buffers are reused for the
whole scan.  This also implies that much of the write traffic caused by such
a statement will be done by the backend itself and not pushed off onto other
processes.

For sequential scans, a 256KB ring is used. That's small enough to fit in L2
cache, which makes transferring pages from OS cache to shared buffer cache
efficient.  Even less would often be enough, but the ring must be big enough
to accommodate all pages in the scan that are pinned concurrently.  256KB
should also be enough to leave a small cache trail for other backends to
join in a synchronized seq scan.  If a ring buffer is dirtied and its LSN
updated, we would normally have to write and flush WAL before we could
re-use the buffer; in this case we instead discard the buffer from the ring
and (later) choose a replacement using the normal clock-sweep algorithm.
Hence this strategy works best for scans that are read-only (or at worst
update hint bits).  In a scan that modifies every page in the scan, like a
bulk UPDATE or DELETE, the buffers in the ring will always be dirtied and
the ring strategy effectively degrades to the normal strategy.

VACUUM uses a 256KB ring like sequential scans, but dirty pages are not
removed from the ring.  Instead, WAL is flushed if needed to allow reuse of
the buffers.  Before introducing the buffer ring strategy in 8.3, VACUUM's
buffers were sent to the freelist, which was effectively a buffer ring of 1
buffer, resulting in excessive WAL flushing.  Allowing VACUUM to update
256KB between WAL flushes should be more efficient.

Bulk writes work similarly to VACUUM.  Currently this applies only to
COPY IN and CREATE TABLE AS SELECT.  (Might it be interesting to make
seqscan UPDATE and DELETE use the bulkwrite strategy?)  For bulk writes
we use a ring size of 16MB (but not more than 1/8th of shared_buffers).
Smaller sizes have been shown to result in the COPY blocking too often
for WAL flushes.  While it's okay for a background vacuum to be slowed by
doing its own WAL flushing, we'd prefer that COPY not be subject to that,
so we let it use up a bit more of the buffer arena.


Background Writer's Processing
------------------------------

The background writer is designed to write out pages that are likely to be
recycled soon, thereby offloading the writing work from active backends.
To do this, it scans forward circularly from the current position of
nextVictimBuffer (which it does not change!), looking for buffers that are
dirty and not pinned nor marked with a positive usage count.  It pins,
writes, and releases any such buffer.

If we can assume that reading nextVictimBuffer is an atomic action, then
the writer doesn't even need to take the BufFreelistLock in order to look
for buffers to write; it needs only to spinlock each buffer header for long
enough to check the dirtybit.  Even without that assumption, the writer
only needs to take the lock long enough to read the variable value, not
while scanning the buffers.  (This is a very substantial improvement in
the contention cost of the writer compared to PG 8.0.)

During a checkpoint, the writer's strategy must be to write every dirty
buffer (pinned or not!).  We may as well make it start this scan from
nextVictimBuffer, however, so that the first-to-be-written pages are the
ones that backends might otherwise have to write for themselves soon.

The background writer takes shared content lock on a buffer while writing it
out (and anyone else who flushes buffer contents to disk must do so too).
This ensures that the page image transferred to disk is reasonably consistent.
We might miss a hint-bit update or two but that isn't a problem, for the same
reasons mentioned under buffer access rules.

As of 8.4, background writer starts during recovery mode when there is
some form of potentially extended recovery to perform. It performs an
identical service to normal processing, except that checkpoints it
writes are technically restartpoints.



---

 ### PostgreSQL源码分析之shared buffer的分配与替换

    shared buffer 本质是一个cache，缓存了常用的磁盘文件的某些个内容，了解shared buffer到磁盘文件的映射关系。既然是缓存，shared buffer的capacity终究是低于磁盘文件的capacity的，不可能将所有磁盘文件，一律缓存到shared buffer。比如我们把shared buffer设置成64M，而磁盘上的文件，随着relation中记录的增加，会变得越来越大，这就必然牵扯到cache的page replacement。就是找不到空闲buffer的时候，应该把哪个buffer作为victim踢出去。本篇博客重点学习shared buffer的分配（alloc）和替换（replacement）。
    代码落在src/backend/storage/buffer目录下，最重要的文件是bufmgr.c这个文件里面有两个函数是全文件的核心BufferAlloc和BgBufferSync。第一个函数顾名思义，就是用来分配buffer的，第二个函数BgBufferSync，说它重要，是因为他是一个主要进程BackgroundWriterMain的主要干活函数。这个函数非常重要，代码量很少，无奈代码不好懂，这个函数折磨的我死去活来，让我费了不少功夫，按下不表（第二个函数不是本文的内容，我还再此提及，可见怨念极深啊）。
   OK，我们关心的重点函数是BufferAlloc，以他为脉络，讲解Alloc以及replacment。
   BufferAlloc要做的事情是给某个relation对应的磁盘文件的某个8KB block分配一个shared buffer，用来存放数据作为缓存。稍等一下，OS的cache不也能提供这种缓存的机制吗？为啥PostgreSQL非要自己再多此一举，自己做个缓存机制。原因就是OS的cache是为OS下的所有进程服务的，不会专门为PostgreSQL定制cache，而且OS的cache的替换策略是LRU，而PostgreSQL自己的shared buffer采用了完全按不同的替换策略，叫做clock-sweep出来，可以这么理解：第一PostgreSQL很自私，首先占领一块内存，自己玩，至于其他的进程，不好意思，不要动我的shared buffer，您一边玩儿而去，第二PostgreSQL为自己的shared buffer的定制了一套替换的策略或者说是机制，PostgreSQL可能认为比OS的LRU换页机制更适合自己。OK，解释了为啥多此一举。
   需要注意两点： 
   1 可能同一个页面即在OS的cache中，又在PostgreSQL的shared buffer中，有点浪费哈，总之了，为了性能。判断那些页面即在OS cahce 又在shared buffer本身这个话题就可以延伸出一篇文章，可是我得克制，否则唧唧歪歪本文就没完没了了，这谁受得了啊。关心这个话题的，自行google pgfincore 。
   2 换页机制，目前9.1.9采用的是我提到的clock-sweep，这个换页机制其实是操作系统很有名的一个问题，操作系统experts也搞出了很多算法，如LRU，LFU,LIRS（Low Inter-reference Recency Set），ARC（Adaptive Replacement Cache ）CLOCK-Pro。其中LRU是Linux用的，LIRS是很有名气的，MySQL采用这种换页算法，ARC是IBM搞出来的，好像很厉害，直接说比LRU更好的换页算法，这个算法细节我也不太懂，总是是个好东西，好像ZFS也用了这个ARC相关的算法算法，可惜有专利，我们PostgreSQL曾经用过ARC做为换页的算法，后来因为专利剔除了ARC。我们不多说，我功力不到，而且这也不是一篇博客所能讲述清楚的，这换页估计可以搞出一本书的内容。
    我瞎扯了半天，可以开始分析code了，要不然，源码分析就成了挂羊头买白菜了。**     Hash 查找**
    shared buffer是有hash table来方便定位某个文件的某个block是否在shared buffers中。
static volatile BufferDesc *
BufferAlloc(SMgrRelation smgr, char relpersistence, ForkNumber forkNum,
            BlockNumber blockNum,
            BufferAccessStrategy strategy,
            bool *foundPtr)
{    
    ....       /* create a tag so we can lookup the buffer */
    INIT_BUFFERTAG(newTag, smgr->smgr_rnode.node, forkNum, blockNum);
    /* determine its hash code and partition lock ID */
    newHash = BufTableHashCode(&newTag);
    newPartitionLock = BufMappingPartitionLock(newHash);
    /* see if the block is in the buffer pool already */
    LWLockAcquire(newPartitionLock, LW_SHARED);
    buf_id = BufTableLookup(&newTag, newHash);
    if (buf_id >= 0)
    {
             ...
             *foundPtr = TRUE;
             ....
             return buf;
    }    ...}    foundPtr是传进来的指针，在函数中会对它赋值，如果TRUE表示在hash table中找到了对应的文件的对应的block，如果FALSE表示当前shared buffer中压根就没有这个block，是我从shared buffer中分配了一个。当然了，如果新分配一个，会插入到hash table，下次来找这个block，就很方便的找到了。 
\#define INIT_BUFFERTAG(a,xx_rnode,xx_forkNum,xx_blockNum) 
( 
    (a).rnode = (xx_rnode), 
    (a).forkNum = (xx_forkNum), 
    (a).blockNum = (xx_blockNum) 
)    上篇博文已经讲过BufferTag到relation的磁盘文件的映射了，不赘述，此处以这个BufferTag作为key去hash table查找，shared buffer中有没有对应的block。这里面有一个算法思想在里面，hash table需要插入，需要删除，需要查找，并发的情况下需要加锁，这个是显而易见的的，但是如果1把锁会引起性能的降低，Shared buffer做了改进，16把锁，相当与将竞争减少到了1把锁的1/16,一个小技巧就缓解了竞争能力。至于LWLockAcquire属于PostgreSQL的Lock机制，我还不太动，不瞎扯，总之是加了一把共享锁。
\#define BufTableHashPartition(hashcode) 
    ((hashcode) % NUM_BUFFER_PARTITIONS)
\#define BufMappingPartitionLock(hashcode) 
    ((LWLockId) (FirstBufMappingLock + BufTableHashPartition(hashcode)))
 newPartitionLock = BufMappingPartitionLock(newHash);
 LWLockAcquire(newPartitionLock, LW_SHARED);
    如果在hashtable中找到了BufferTag对应的block，就意味着在Shared buffer中存在该block，皆大欢喜啊，返回buffer同时将foundPtr设置成TRUE告诉调用者 ，在share buffer中找到了该block。
   如果没找到，就需要查找空闲的buffer，如果没有空闲，那么就要将现有的buffer置换出去，用这个buffer响应系当前这个请求。这个查找free 以及没有free的buffer就选择一个牺牲品这个事情是由StrategyGetBuffer函数完成的，所谓的clock sweep算法也是这个短小函数实现的。因为这里面有很多的细节，我接触的时间短，不能做到事事了然，所以我主要讲算法思想，细节我就力不逮己了。
    **查找free的buffer   **首先是查找free：
while (StrategyControl->firstFreeBuffer >= 0)
    {
        buf = &BufferDescriptors[StrategyControl->firstFreeBuffer];
        Assert(buf->freeNext != FREENEXT_NOT_IN_LIST);
        /* Unconditionally remove buffer from freelist */
        StrategyControl->firstFreeBuffer = buf->freeNext;
        buf->freeNext = FREENEXT_NOT_IN_LIST;
        /*
         * If the buffer is pinned or has a nonzero usage_count, we cannot use
         * it; discard it and retry. (This can only happen if VACUUM put a
         * valid buffer in the freelist and then someone else used it before
         * we got to it. It's probably impossible altogether as of 8.3, but
         * we'd better check anyway.)
         */
        LockBufHdr(buf);
        if (buf->refcount == 0 && buf->usage_count == 0)
        {
            if (strategy != NULL)
                AddBufferToRing(strategy, buf);
            return buf;
        }
        UnlockBufHdr(buf);
    }    原理比较简单，所有的free buffer都在 StrategyControl->firstFreeBuffer为头节点的链表上，初始化的时候，将所有的buffer都放入这个这个单链表。每次取头部的第一个拿来用。本来然后将第一的freeNext作为新的firstFreeBuffer。没啥说的，单链表，大家懂的。但是这里面判断了refcount和usage_count，按说没被使用这俩值不会有大于0的情况，这个判断纯属多余，但是上面有注释，解释了某情况下会出现，blabla我也不懂。
   能取到free当然好，但是如果你的shared buffer比较少，没有free的，那就惨了，就要选择一个牺牲品，将现有的内容驱逐出去，然后将buffer给当前的这个请求用。但是选谁当牺牲品呢？这是PostgreSQL clock sweep算法干的事儿。
    clock sweep 置换算法
   所谓clock sweep听起来很NB，其实原理还是蛮简单的，复杂的原理必然带来复杂的实现，对于BufferAlloc这种调用很频繁的函数，复杂的实现必然带来性能的降低，所以take it easy 。
   buffer有个参数叫usage_count,顾名思义就是使用次数，如果你需要这个BufferTag对应的文件，我也需要，那么这个次数就是2 ，也就是BufferAlloc之后，会对这个值+1,PostgreSQL不是无限制的增加：  
        buf->refcount++;
        if (strategy == NULL)
        {
            if (buf->usage_count < BM_MAX_USAGE_COUNT)
                buf->usage_count++;
        }    BM_MAX_USAGE_COUNT = 5,这个usage_count最多是5.这个值来决定该置换哪个buffer。所有的buffer都不是空闲的，那么就剔除那个usage_count最小的。如何执行？环形扫描，如果扫到了你，你的usage_count--,直到遇到第一usage_count变成0的buffer。
    /* Nothing on the freelist, so run the "clock sweep" algorithm */
    trycounter = NBuffers;
    for (;;)
    {
        buf = &BufferDescriptors[StrategyControl->nextVictimBuffer];
        if (++StrategyControl->nextVictimBuffer >= NBuffers)
        {
            StrategyControl->nextVictimBuffer = 0;
            StrategyControl->completePasses++;
        }
        /*
         * If the buffer is pinned or has a nonzero usage_count, we cannot use
         * it; decrement the usage_count (unless pinned) and keep scanning.
         */
        LockBufHdr(buf);
        if (buf->refcount == 0)
        {
            if (buf->usage_count > 0)
            {
                buf->usage_count--; //扫到buffer，buffer的usage_count--，buffer usage_count小的会首先顶不住减小到0
                trycounter = NBuffers;
            }
            else
            {
                /* Found a usable buffer */
                if (strategy != NULL)
                    AddBufferToRing(strategy, buf);
                return buf;     //有buffer顶不住了，变成了0,那么就选择这个buffer作为牺牲品，替换掉。
            }
        }
        else if (--trycounter == 0)
        {
            /*
             * We've scanned all the buffers without making any state changes,
             * so all the buffers are pinned (or were when we looked at them).
             * We could hope that someone will free one eventually, but it's
             * probably better to fail than to risk getting stuck in an
             * infinite loop.
             */
            UnlockBufHdr(buf);
            elog(ERROR, "no unpinned buffers available");
        }
        UnlockBufHdr(buf);
    }    PostgreSQL有pg_buffercache的扩展，可以查看每个buffer的一些信息：如下图所示：
  ![img](http://blog.chinaunix.net/attachment/201306/10/24774106_137085467021PC.png)
   选到了buffer有可能是dirty的，换言之，buffer上的内容比磁盘上的对应block要新，需要sync到磁盘上（调用FlushBuffer），如果原hashtable中不存在，是新加入的buffer，需要插入的hash table中（调用BufTableInsert），这是事情我就不一一赘述了。 
   对于这个算法，新加入的buffer 容易被置换出去，已经有开发者提出并质疑这个问题了，今年三月份有个讨论十分牛，各路大神出没，有人整理成了30多页的文档，对这个topic的理解帮助极大。讨论名字叫 Page replacement algorithm in buffer cache。


