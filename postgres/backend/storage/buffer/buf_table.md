## src/backend/storage/buffer/

### buf_table.c

**将 BufferTags 映射到缓冲区索引的例程。**

* Include dependency graph for buf_table.c

  ![](http://doxygen.postgresql.org/buf__table_8c__incl.png)

  ```C
   typedef struct HASHCTL
  {
      long        num_partitions; /* # partitions (must be power of 2) */
      long        ssize;          /* segment size */
      long        dsize;          /* (initial) directory size */
      long        max_dsize;      /* limit to dsize if dir size is limited */
      long        ffactor;        /* fill factor */
      Size        keysize;        /* hash key length in bytes */
      Size        entrysize;      /* total user element size in bytes */
      HashValueFunc hash;         /* hash function */
      HashCompareFunc match;      /* key comparison function */
      HashCopyFunc keycopy;       /* key copying function */
      HashAllocFunc alloc;        /* memory allocator */
      MemoryContext hcxt;         /* memory context to use for allocations */
      HASHHDR    *hctl;           /* location of header in shared mem */
  } HASHCTL;
  ```

  ```C
  //Buffer tag identifies which disk block the buffer contains.
  typedef struct buftag {
    RelfileNode rnode;  /* Physical relation identifier */
    ForkNumber forkNum;
    BlockNumber blockNum; /* blknum relative to begin of reln */
  } BufferTag;
  ```

  ```C
  //all the below funcs that related to table operation using a function named "hash_search_...", in that, there is an enum options of hash search operations:
  typedef enum {
    HASH_FIND,   //BufTableLookup()
    HASH_ENTER,   //BufTableInsert()
    HASH_REMOVE,   //BufTableDelete()
    HASH_ENTER_NULL
  } HASHACTION;
  ```

  ​

  | Data Structures |                                          |
  | --------------- | ---------------------------------------- |
  | struct          | **BufferLookupEnt**       //entry for buffer lookup hash table |

  | Functions |                                          |
  | --------- | ---------------------------------------- |
  | **Size**  | **BufTableShmemSize** (int size)   //estimate space needed for mapping hash table, size is the desired hash table size, possibly more than NBuffers. |
  | void      | **InitBufTable** (int size)     //Initialize shame hash table for mapping buffers.  the _HASHCTL_ is parameter data structure for hash create. only those fields indicated by **hash_flags** need to be set.    **Referenced by StrategyInitialize()**. |
  | uint32    | **BufTableHashCode**  (BufferTag \*tagPtr)   //compute the hash code associated with a Buffer tag.  **HTAB** is the top control structure for a hash table — in a shared table, each backend has its own copy |
  | int       | **BufTableLookup** (BufferTag \*tagPtr, uint32 hashcode)   // Lookup the given Buffet tag; return buffer ID, or -1 if not found. |
  | int       | **BufTableInsert** (BufferTag \*tagPtr, uint32 hashcode, int buf_id)   //insert a  hash table entry for given tag and buffer id, unless an entry already exist for that tag. |
  | void      | **BufTableDelete** (BufferTag \*tagPtr, uint32 hashcode)   //delete the hashtable entry for given tag(which must exist). |

  | Variables          |                                          |
  | ------------------ | ---------------------------------------- |
  | static **HTAB** \* | SharedBufHash   //the handler of top control structure for a hash table.  in a shared table, each backend has its own copy. |


```C
struct HTAB
{
    HASHHDR    *hctl;           /* => shared control information */
    HASHSEGMENT *dir;           /* directory of segment starts */
    HashValueFunc hash;         /* hash function */
    HashCompareFunc match;      /* key comparison function */
    HashCopyFunc keycopy;       /* key copying function */
    HashAllocFunc alloc;        /* memory allocator */
    MemoryContext hcxt;         /* memory context if default allocator used */
    char       *tabname;        /* table name (for error messages) */
    bool        isshared;       /* true if table is in shared memory */
    bool        isfixed;        /* if true, don't enlarge */

    /* freezing a shared table isn't allowed, so we can keep state here */
    bool        frozen;         /* true = no more inserts allowed */

    /* We keep local copies of these fixed values to reduce contention */
    Size        keysize;        /* hash key length in bytes */
    long        ssize;          /* segment size --- must be power of 2 */
    int         sshift;         /* segment shift = log2(ssize) */
};
```

