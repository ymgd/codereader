## src/backend/storage/buffer/

### localbuf.c

local buffer manager. Fast buffer manager for **temporary tables**, which **never** need to be WAL-logged or checkpointed, etc.

* include dependency graph for localbuf.c

  ![](http://doxygen.postgresql.org/localbuf_8c__incl.png)

* 有一点需要说明的是，使用**负数**指示local buffer，正数指示shared buffer。 因此，shared buffer 从**0** 开始，**local buffer从-1开始**。因此，buf_id要从**-2**开始，因为BufferDescriptorGetBuffers 会将buf_id 加1.

  ---


* **Data Structures**

  * struct  **_LocalBufferLookupEnt_**

    **entry** for buffer lookup hashtable. 类似于buf_table.c中的**BufferLookupEnt**，有（BufferTag）key和（int）id两个变量。

* **Macros**

  * define **_LocalBufHdrGetBlock_**(bufHdr) _LocalBufferBlockPointers_[-((bufHdr)->buf_id + 2)]
    * Referenced by **LocalBufferAlloc()**
    * 仅仅在local buffer上工作，不在共享的缓冲区中使用。
    * ​

* **Functions**

  * static void  **_InitLocalBuffers_** (void)

    用于初始化local buffer cache。由于大多数queries（multi-users queries）不使用local buffer， 因此使用**懒加载，直到需要的时候才为它分配空间**。 而**只定义buffer header**。

    执行过程：

    * 并行的workers无法获取临时table中的data，因为local buffer的header对其不可见。**首先检查是否是并行作业，如果是，抛出异常**“cannot access temporary tables during a parallel operation.”

    * **分配并0化buffer header和辅助数组**：为localbufferdescriptor，localbufferblockpointer和localrefcount分配空间。将nextfreelocalbuf置为0（第一个buf）。并对初始值不能为空的之进行初始化（(BufferDesc \*)buf->buf_id）

    * **创建新的lookup hashtable**：主要是设置HASHCTL的有关变量，如keysize，entrysize等。并用其初始化LocalBufHash,即我们需要的local hash table。

      ```C
      static HTAB * 	LocalBufHash = NULL;
      ```




* static _Block_ **_GetLocalBufferStorage_** (void)
    * **职责：** 为一个local buffer 分配memory。这个函数采用**聚集**（aggregate） 的方式来**批处理大量相关的small requests**。
    * 执行过程：
      * 检查total_bufs_allocated是否小于NLocBuffer。当next_buf_in_block小于num_bufs_in_block时，分配当前memory block的下一个buffer给requrest。
      * 如果大于num_bufs_in_block，需要make new requests to **memmgr** ：新建BufferContext，第一次分配16个buffer，后续的requests每次double。但是不能超过剩余local buf的数量，不能超出MaxAllocSize的要求。




* void **_LocalPrefetchBuffer_** (_SMgrRelation_ smgr, _ForkNumber_ forkNum, _BlockNumber_ blockNum)

    * 初始化一个relation的对某一个block的异步read。
    * 大致就是：检测被requested的block是否在buffer中，如果不在，调用**smgrprefectch（smgr, forkNum, blockNum）**





* BufferDesc_ * **_LocalBufferAlloc_** (_SMgrRelation_ smgr, _ForkNumber_ forkNum, _BlockNumber_ blockNum, bool *foundPtr) 
    * 用于查找或者create给定ralation的给定page的local buffer，不同于bufmgr，不需要加锁。也不需要设置IO_IN_PROGRESS，并且，**只支持默认的access strategy**。
    * 执行过程：
      1. 对于某个session中的first request，初始化local buffers
      2. 检查要找的buffer是否已经存在：hash_search()
      3. 如果存在，检查该buf的localRefCount和usage_count是否符合要求，如果符合，将localRefCount+1，并将OwnerBuffer同步更新。最后将foundPtr置为true，并且返回该buf的BufferDesc* bufHdr。
      4. 如果不存在，需要获取一个新的buffer。使用**clock sweep**算法。这与freelist中的做法一样，不再赘述。…… 获取到buf之后，如果其实dirty的，需要在re-use 它之前将它写回。**写回后**， 采用**懒加载**的方式为某一个buffer的first use分配空间。
      5. 在此之后，更新hashtable： 删除旧entry（如果有的话），再make a new one。
      6. 再次检查hash table，从中**hash_search** 所需的buffer。
      7. 将获取到的buf**复制**到bufHdr和buf_state，并将bufHdr返回





*   void **_MarkLocalBufferDirty_** (_Buffer_ buffer)

    *   用于讲一个localbuf 标记为**dirty**
    *   **Referenced by MairBufferDirty()  and MarkBufferDirtyHint()**





*   void **_DropRelFileNodeLocalBuffers_** (_RelFileNode_ rnode, _ForkNumber_ forkNum, _BlockNumber_ firstDelBlock)

    *   referenced by **DropRelFileNodeBuffers()**
    *   用于将制定关系的缓冲池中blcknum大于firstDelBlock的pages删除掉。dirty pages被**丢弃**， 不写回。因此，这个操作**不支持回滚**。因此，只在不得已的情况下才使用。
    *   此操作主要是检查每个fileNode的refcount，如果仍有引用，抛出错误，如果没有，将该entry从hashtable中移除。然后将buffer标记为invalid（0）。





*   void **_DropRelFileNodeAllLocalBuffers_** (_RelFileNode_ rnode)

    *   该函数和上面的类似，可以视作上面的firstDelBlock==0的版本，因此不再赘述。





*   static void _CheckForLocalBufferLeaks_ (void）

    * 用于保证当前backend不hold任何local buffer pins。类似以CheckBufferLeaks（），只不过是local buffer的版本。
    * 过程：遍历NLocBuffer个buf, 对应的LocalRefCount[i] 不为0的即为leaks的项。并统计出errors的总数。





*   void _AtEOXact_LocalBuffers_ (bool isCommit) 

    * 用于在事务结束时做必要的清理工作。实际上就是**调用一下CheckForLocalBufferLeaks（）**





*   void _AtProcExit_LocalBuffers_ (void)  //用于保证在backend退出时，正确释放pin。

    * 实际上就是**调用一下CheckForLocalBufferLeaks（）**





*   **Variables**

    * int NLocBuffer = 0  //直到buffers are initialized

    * _BufferDesc_ * LocalBufferDescriptors = NULL

    * _Block_ * LocalBufferBlockPointers = NULL

      ```C
      typedef void* Block;
      ```

    * int32 * LocalRefCount = NULL  //局部引用计数

    * static int nextFreeLocalBuf = 0   //下一个free的localbuffer的index，是即将被操作的对象

      * static _HTAB_ * LocalBufHash = NULL

      **HTAB** is the **top control structure** for a hashtable.

      in a **shared table**, each backend has its **own copy**.

      * HTAB is defined in _dynahash.c

        it supports both local-to-a-backend hash tables and hash tables in shared memory.



