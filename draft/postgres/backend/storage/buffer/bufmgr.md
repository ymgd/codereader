## src/backend/storage/buffer/

### bufmgr.c

**include dependency graph for bufmgr.c**

![](http://doxygen.postgresql.org/bufmgr_8c__incl.png)

**This is the _buffer manager interface_**.

### structure of a postgres disk page:

```markdown
/* 22
   23  * A postgres **disk page** is an abstraction layered on top of a postgres
   24  * disk block (which is simply a unit of i/o, see block.h).
   25  *
   26  * specifically, while a disk block can be unformatted, a **postgres
   27  * disk** page is always a slotted page of the form:
   28  *
   29  * +----------------+---------------------------------+
   30  * | PageHeaderData | linp1 linp2 linp3 ...           |
   31  * +-----------+----+---------------------------------+
   32  * | ... linpN |                                      |
   33  * +-----------+--------------------------------------+
   34  * |           ^ pd_lower                             |
   35  * |                                                  |
   36  * |             v pd_upper                           |
   37  * +-------------+------------------------------------+
   38  * |             | tupleN ...                         |
   39  * +-------------+------------------+-----------------+
   40  * |       ... tuple3 tuple2 tuple1 | "special space" |
   41  * +--------------------------------+-----------------+
   42  *                                  ^ pd_special
   43  *
   44  * a page is full when nothing can be added between **pd_lower** and
   45  * **pd_upper**.
   46  *
   47  * all blocks written out by an access method **must be disk pages**.
   48  *
   49  * EXCEPTIONS:
   50  *
   51  * obviously, a page is not formatted before it is initialized by
   52  * a call to **PageInit**.
   53  *
   54  * NOTES:
   55  *
   56  * **linp1..N form an ItemId array**.  ItemPointers point into this array
   57  * rather than pointing directly to a tuple.  Note that **OffsetNumbers**
   58  * conventionally start at 1, not 0.
   59  *
   60  * tuple1..N are added "backwards" on the page.  because a tuple's
   61  * ItemPointer points to its ItemId entry rather than its actual
   62  * byte-offset position, tuples can be physically shuffled on a page
   63  * whenever the need arises.
   64  *
   65  * AM-generic per-page information is kept in PageHeaderData.
   66  *
   67  * AM-specific per-page data (if any) is kept in the area marked "special
   68  * space"; each AM has an "opaque" structure defined somewhere that is
   69  * stored as the page trailer.  an access method should always
   70  * initialize its pages with PageInit and then set its own opaque
   71  * fields.
*/
```

# [49.3. Database Page Layout](undefined)

This section provides an overview of the page format used within PostgreSQL tables and indexes.[[1\]](https://www.postgresql.org/docs/8.0/static/storage-page-layout.html#FTN.AEN58382) Sequences and TOAST tables are formatted just like a regular table.

In the following explanation, a *byte* is assumed to contain 8 bits. In addition, the term *item* refers to an individual data value that is stored on a page. In a table, an item is a row; in an index, an item is an index entry.

Every table and index is stored as an array of *pages* of a fixed size (usually 8Kb, although a different page size can be selected when compiling the server). In a table, all the pages are logically equivalent, so a particular item (row) can be stored in any page. In indexes, the first page is generally reserved as a *metapage* holding control information, and there may be different types of pages within the index, depending on the index access method.

[Table 49-2](https://www.postgresql.org/docs/8.0/static/storage-page-layout.html#PAGE-TABLE) shows the overall layout of a page. There are five parts to each page.

Table 49-2. Overall **Page Layout**

| Item           | Description                              |
| -------------- | ---------------------------------------- |
| PageHeaderData | 20 bytes long. Contains general information about the page, including free space pointers. |
| ItemIdData     | Array of (offset,length) pairs pointing to the actual items. 4 bytes per item. |
| Free space     | The unallocated space. New item pointers are allocated from the start of this area, new items from the end. |
| Items          | The actual items themselves.             |
| Special space  | Index access method specific data. Different methods store different data. Empty in ordinary tables. |

The first 20 bytes of each page consists of a page header (PageHeaderData). Its format is detailed in [Table 49-3](https://www.postgresql.org/docs/8.0/static/storage-page-layout.html#PAGEHEADERDATA-TABLE). The first two fields track the most recent WAL entry related to this page. They are followed by three 2-byte integer fields (pd_lower, pd_upper, and pd_special). These contain byte offsets from the page start to the start of unallocated space, to the end of unallocated space, and to the start of the special space. The last 2 bytes of the page header, pd_pagesize_version, store both the page size and a version indicator. Beginning with PostgreSQL 8.0 the version number is 2; PostgreSQL 7.3 and 7.4 used version number 1; prior releases used version number 0. (The basic page layout and header format has not changed in these versions, but the layout of heap row headers has.) The page size is basically only present as a cross-check; there is no support for having more than one page size in an installation.

Table 49-3. **PageHeaderData** Layout

| Field               | Type          | Length  | Description                              |
| ------------------- | ------------- | ------- | ---------------------------------------- |
| pd_lsn              | XLogRecPtr    | 8 bytes | LSN: next byte after last byte of xlog record for last change to this page |
| pd_tli              | TimeLineID    | 4 bytes | TLI of last change                       |
| pd_lower            | LocationIndex | 2 bytes | Offset to start of free space            |
| pd_upper            | LocationIndex | 2 bytes | Offset to end of free space              |
| pd_special          | LocationIndex | 2 bytes | Offset to start of special space         |
| pd_pagesize_version | uint16        | 2 bytes | Page size and layout version number information |

All the details may be found in src/include/storage/bufpage.h.

Following the page header are item identifiers (ItemIdData), each requiring four bytes. An item identifier contains a byte-offset to the start of an item, its length in bytes, and a few attribute bits which affect its interpretation. New item identifiers are allocated as needed from the beginning of the unallocated space. The number of item identifiers present can be determined by looking at pd_lower, which is increased to allocate a new identifier. Because an item identifier is never moved until it is freed, its index may be used on a long-term basis to reference an item, even when the item itself is moved around on the page to compact free space. In fact, every pointer to an item (ItemPointer, also known as CTID) created by PostgreSQL consists of a page number and the index of an item identifier.

The items themselves are stored in space allocated backwards from the end of unallocated space. The exact structure varies depending on what the table is to contain. Tables and sequences both use a structure namedHeapTupleHeaderData, described below.

The final section is the "special section" which may contain anything the access method wishes to store. For example, b-tree indexes store links to the page's left and right siblings, as well as some other data relevant to the index structure. Ordinary tables do not use a special section at all (indicated by setting pd_special to equal the page size).

All table rows are structured in the same way. There is a fixed-size header (occupying 27 bytes on most machines), followed by an optional null bitmap, an optional object ID field, and the user data. The header is detailed in [Table 49-4](https://www.postgresql.org/docs/8.0/static/storage-page-layout.html#HEAPTUPLEHEADERDATA-TABLE). The actual user data (columns of the row) begins at the offset indicated by t_hoff, which must always be a multiple of the MAXALIGN distance for the platform. The null bitmap is only present if the*HEAP_HASNULL* bit is set in t_infomask. If it is present it begins just after the fixed header and occupies enough bytes to have one bit per data column (that is, t_natts bits altogether). In this list of bits, a 1 bit indicates not-null, a 0 bit is a null. When the bitmap is not present, all columns are assumed not-null. The object ID is only present if the *HEAP_HASOID* bit is set in t_infomask. If present, it appears just before the t_hoff boundary. Any padding needed to make t_hoff a MAXALIGN multiple will appear between the null bitmap and the object ID. (This in turn ensures that the object ID is suitably aligned.)

Table 49-4. HeapTupleHeaderData Layout

| Field      | Type            | Length  | Description                              |
| ---------- | --------------- | ------- | ---------------------------------------- |
| t_xmin     | TransactionId   | 4 bytes | insert XID stamp                         |
| t_cmin     | CommandId       | 4 bytes | insert CID stamp                         |
| t_xmax     | TransactionId   | 4 bytes | delete XID stamp                         |
| t_cmax     | CommandId       | 4 bytes | delete CID stamp (overlays with t_xvac)  |
| t_xvac     | TransactionId   | 4 bytes | XID for VACUUM operation moving a row version |
| t_ctid     | ItemPointerData | 6 bytes | current TID of this or newer row version |
| t_natts    | int16           | 2 bytes | number of attributes                     |
| t_infomask | uint16          | 2 bytes | various flag bits                        |
| t_hoff     | uint8           | 1 byte  | offset to user data                      |

All the details may be found in src/include/access/htup.h.

Interpreting the actual data can only be done with information obtained from other tables, mostly pg_attribute. The key values needed to identify field locations are attlen and attalign. There is no way to directly get a particular attribute, except when there are only fixed width fields and no NULLs. All this trickery is wrapped up in the functions *heap_getattr*, *fastgetattr* and *heap_getsysattr*.

To read the data you need to examine each attribute in turn. First check whether the field is NULL according to the null bitmap. If it is, go to the next. Then make sure you have the right alignment. If the field is a fixed width field, then all the bytes are simply placed. If it's a variable length field (attlen = -1) then it's a bit more complicated. All variable-length datatypes share the common header structure varattrib, which includes the total length of the stored value and some flag bits. Depending on the flags, the data may be either inline or in a TOAST table; it might be compressed, too (see [Section 49.2](https://www.postgresql.org/docs/8.0/static/storage-toast.html)).

### Notes

| [[1\]](https://www.postgresql.org/docs/8.0/static/storage-page-layout.html#AEN58382) | Actually, index access methods need not use this page format. All the existing index methods do use this basic format, but the data kept on index metapages usually doesn't follow the item layout rules. |
| ---------------------------------------- | ---------------------------------------- |
|                                          |                                          |



**Principal entry points:**

* ReadBuffer():

  find or create a buffer holding the requested page and pin it so that no one can destory it while this process is using it.

* ReleaseBuffer():

  unpin a buffer.

* MarkBufferDirty():

  mark a pinned buffer's countents as "dirty". the **disk write** is **delayed** until buffer **replacement** or at **checkpoint**.

---

| Data Structures |                                          |
| --------------- | ---------------------------------------- |
| struct          | [PrivateRefCountEntry](http://doxygen.postgresql.org/structPrivateRefCountEntry.html) |
|                 | Buffer, refcount. 表示refcoun_array中的一个entry实体 |
| struct          | [CkptTsStatus](http://doxygen.postgresql.org/structCkptTsStatus.html)   //checkpoint tablespace status |
|                 | 表示buffer在某个tablespace的checkpoint的状态，在BufferSync内部使用。                   it has the **Oid** of the tablespace. 为了使得不同tablespace内的进程可以比较，tablespace的checkpoint进程以0到num（需要被checkpointed的pages总数）表啊是。每个已经被check的pages将会将这个tablespace的progresss增加progress_slice。该结构还会记录num_to_scan, num_scanned, current offset in ckptbuffid for this tablespace. |

| Macros  |                                          |
| ------- | ---------------------------------------- |
| #define | [BufHdrGetBlock](http://doxygen.postgresql.org/bufmgr_8c.html#a6d0dee25d1b976e85a4bd4f35ca117df)(bufHdr)   (([Block](http://doxygen.postgresql.org/bufmgr_8h.html#a915a8d917b74a02ca876a2d4bdb09113)) ([BufferBlocks](http://doxygen.postgresql.org/bufmgr_8h.html#aeb3f13c9a5f988b4e77ff61dabef2a36) + (([Size](http://doxygen.postgresql.org/c_8h.html#af9ecec2d692138fab9167164a457cbd4)) (bufHdr)->buf_id) * BLCKSZ)) |
|         | BufferBlocks: char *, Block: void *, Size: size_t,用于根据bufHdr的buf_id来获取指定的bufferblock。 |
| #define | [BufferGetLSN](http://doxygen.postgresql.org/bufmgr_8c.html#a44d61775b7a7eaccc9152b6825341f09)(bufHdr)   ([PageGetLSN](http://doxygen.postgresql.org/bufpage_8h.html#afce0427fb3dedb8efa6921ac61f778c2)([BufHdrGetBlock](http://doxygen.postgresql.org/bufmgr_8c.html#a6d0dee25d1b976e85a4bd4f35ca117df)(bufHdr))) |
|         | \#define PageGetLSN(page) PageXLogRecPtrGet(((PageHeader)(page)) -> pd_lsn).                          The pg_lsn data type can be used to store **LSN(Log Sequence Number)** data which is a pointer to a location in the XLOG. |
| #define | [LocalBufHdrGetBlock](http://doxygen.postgresql.org/bufmgr_8c.html#a40dd99e3353d9f3f35eb0dcb467df9ed)(bufHdr)   [LocalBufferBlockPointers](http://doxygen.postgresql.org/bufmgr_8h.html#a40961e6ad22a8ab6be09b4e963a1a9ad)[-((bufHdr)->buf_id + 2)] |
|         | local的buffer block编号从-1开始，其他细节和shared相似，因此BufHdrGetBlock也和shared相似。 |
| #define | [BUF_WRITTEN](http://doxygen.postgresql.org/bufmgr_8c.html#afe69627033d23343e54128a61c862324)   0x01 |
| #define | [BUF_REUSABLE](http://doxygen.postgresql.org/bufmgr_8c.html#a7166d91fd5228c5d5e819f7319d719e1)   0x02 |
|         | 这两个宏作为SyncOneBuffer的返回值的bits。            |
| #define | [DROP_RELS_BSEARCH_THRESHOLD](http://doxygen.postgresql.org/bufmgr_8c.html#ab8c524e04d5f83106ebb6842fbc9a106)   20 |
|         | drop_rels_bsearch_threshold              |
| #define | [REFCOUNT_ARRAY_ENTRIES](http://doxygen.postgresql.org/bufmgr_8c.html#a8c9b241ffedfeab215e054d26e74457f)   8 |
|         | 64位，与常见系统的一个cache line的size有关            |
| #define | [BufferIsPinned](http://doxygen.postgresql.org/bufmgr_8c.html#ab2fd7b5f4f98e9c646d29b955728528b)(bufnum) |
|         | 用于检查bufnum对应的buf有没有pin（bufferisvalid 或者 localrefcount 或者privaterefcount） |

| Typedefs                                 |                                          |
| ---------------------------------------- | ---------------------------------------- |
| [typedef](http://doxygen.postgresql.org/mingwcompat_8c.html#ad87859e2d4486b3c76fa7783ab3d2ccc) struct [PrivateRefCountEntry](http://doxygen.postgresql.org/structPrivateRefCountEntry.html) | [PrivateRefCountEntry](http://doxygen.postgresql.org/bufmgr_8c.html#a349c87d11986ea2adadf654811ce2115) |
| [typedef](http://doxygen.postgresql.org/mingwcompat_8c.html#ad87859e2d4486b3c76fa7783ab3d2ccc) struct [CkptTsStatus](http://doxygen.postgresql.org/structCkptTsStatus.html) | [CkptTsStatus](http://doxygen.postgresql.org/bufmgr_8c.html#accad32efc165f297130568fd74b9fda8) |


| Functions                                |                                          |
| ---------------------------------------- | ---------------------------------------- |
| static void                              | [ReservePrivateRefCountEntry](http://doxygen.postgresql.org/bufmgr_8c.html#ab8d3486e7706444e3648150471713418) (void) |
|                                          |                                          |
| static [PrivateRefCountEntry](http://doxygen.postgresql.org/structPrivateRefCountEntry.html) * | [NewPrivateRefCountEntry](http://doxygen.postgresql.org/bufmgr_8c.html#a134e9affa0dbd9045a503777d1f8a371) ([Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) buffer) |
|                                          |                                          |
| static [PrivateRefCountEntry](http://doxygen.postgresql.org/structPrivateRefCountEntry.html) * | [GetPrivateRefCountEntry](http://doxygen.postgresql.org/bufmgr_8c.html#ad6c47bd4c55cf1ced820faec2a7cc069) ([Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) buffer, [bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0) do_move) |
|                                          |                                          |
| static [int32](http://doxygen.postgresql.org/c_8h.html#a43d43196463bde49cb067f5c20ab8481) | [GetPrivateRefCount](http://doxygen.postgresql.org/bufmgr_8c.html#abcbf8ed97899cf62784efe9d00b5bbcf) ([Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) buffer) |
|                                          |                                          |
| static void                              | [ForgetPrivateRefCountEntry](http://doxygen.postgresql.org/bufmgr_8c.html#ac92a69fc33dab659a84178b9ac984e04) ([PrivateRefCountEntry](http://doxygen.postgresql.org/structPrivateRefCountEntry.html) *ref) |
|                                          |                                          |
| static [Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) | [ReadBuffer_common](http://doxygen.postgresql.org/bufmgr_8c.html#ad7c65b4449cce1235afbbb99f1e23592) ([SMgrRelation](http://doxygen.postgresql.org/smgr_8h.html#ab1753f7a9304d3a0b8cfb4d349441591) reln, char relpersistence, [ForkNumber](http://doxygen.postgresql.org/relpath_8h.html#a4e49e3b48d6a980e40dbde579f89237d) forkNum, [BlockNumber](http://doxygen.postgresql.org/block_8h.html#a0be1c1ab88d7f8120e2cd2e8ac2697a1) blockNum, [ReadBufferMode](http://doxygen.postgresql.org/bufmgr_8h.html#af2fba14fc04ffc8c72f1c250fa499cc0) mode,[BufferAccessStrategy](http://doxygen.postgresql.org/buf_8h.html#aa259999f7bb734ea3bb310d66d1f6f1c) strategy, [bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0) *hit) |
|                                          |                                          |
| static [bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0) | [PinBuffer](http://doxygen.postgresql.org/bufmgr_8c.html#a318f9075a74b4a583d2b657e27997f8a) ([BufferDesc](http://doxygen.postgresql.org/structBufferDesc.html) *[buf](http://doxygen.postgresql.org/pg__test__fsync_8c.html#ac14417684334d01b4e1e807a19d92816), [BufferAccessStrategy](http://doxygen.postgresql.org/buf_8h.html#aa259999f7bb734ea3bb310d66d1f6f1c) strategy) |
|                                          |                                          |
| static void                              | [PinBuffer_Locked](http://doxygen.postgresql.org/bufmgr_8c.html#a367007f8359f85c05c377835eb3a0827) ([BufferDesc](http://doxygen.postgresql.org/structBufferDesc.html) *[buf](http://doxygen.postgresql.org/pg__test__fsync_8c.html#ac14417684334d01b4e1e807a19d92816)) |
|                                          |                                          |
| static void                              | [UnpinBuffer](http://doxygen.postgresql.org/bufmgr_8c.html#afc319381e884a3755a3f9c8d4c06e544) ([BufferDesc](http://doxygen.postgresql.org/structBufferDesc.html) *[buf](http://doxygen.postgresql.org/pg__test__fsync_8c.html#ac14417684334d01b4e1e807a19d92816), [bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0) fixOwner) |
|                                          |                                          |
| static void                              | [BufferSync](http://doxygen.postgresql.org/bufmgr_8c.html#a74bb530d6e82d93436c68b788234b481) (int flags) |
|                                          |                                          |
| static [uint32](http://doxygen.postgresql.org/c_8h.html#a1134b580f8da4de94ca6b1de4d37975e) | [WaitBufHdrUnlocked](http://doxygen.postgresql.org/bufmgr_8c.html#a3d338c527c69b810d3962c92405bb7d6) ([BufferDesc](http://doxygen.postgresql.org/structBufferDesc.html) *[buf](http://doxygen.postgresql.org/pg__test__fsync_8c.html#ac14417684334d01b4e1e807a19d92816)) |
|                                          |                                          |
| static int                               | [SyncOneBuffer](http://doxygen.postgresql.org/bufmgr_8c.html#ac941343e591b1fa2ae0935345ae343a2) (int buf_id, [bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0) skip_recently_used, [WritebackContext](http://doxygen.postgresql.org/structWritebackContext.html) *flush_context) |
|                                          |                                          |
| static void                              | [WaitIO](http://doxygen.postgresql.org/bufmgr_8c.html#afbc394f250d6db764cd849d12841f302) ([BufferDesc](http://doxygen.postgresql.org/structBufferDesc.html) *[buf](http://doxygen.postgresql.org/pg__test__fsync_8c.html#ac14417684334d01b4e1e807a19d92816)) |
|                                          |                                          |
| static [bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0) | [StartBufferIO](http://doxygen.postgresql.org/bufmgr_8c.html#a98a0b9e6db044d15faeff2b23beca23f) ([BufferDesc](http://doxygen.postgresql.org/structBufferDesc.html) *[buf](http://doxygen.postgresql.org/pg__test__fsync_8c.html#ac14417684334d01b4e1e807a19d92816), [bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0) forInput) |
|                                          |                                          |
| static void                              | [TerminateBufferIO](http://doxygen.postgresql.org/bufmgr_8c.html#ae62b4c6bb256791b3da45ff81a224640) ([BufferDesc](http://doxygen.postgresql.org/structBufferDesc.html) *[buf](http://doxygen.postgresql.org/pg__test__fsync_8c.html#ac14417684334d01b4e1e807a19d92816), [bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0) clear_dirty, [uint32](http://doxygen.postgresql.org/c_8h.html#a1134b580f8da4de94ca6b1de4d37975e) set_flag_bits) |
|                                          |                                          |
| static void                              | [shared_buffer_write_error_callback](http://doxygen.postgresql.org/bufmgr_8c.html#aa7c848e5a2b56c976b044ed5bc026f17) (void *[arg](http://doxygen.postgresql.org/pg__backup__utils_8c.html#a9ce2ec4812a92cb6ab39f6e81e9173a9)) |
|                                          |                                          |
| static void                              | [local_buffer_write_error_callback](http://doxygen.postgresql.org/bufmgr_8c.html#a74b40e83acf76d736e58f4c70e1bdb8d) (void *[arg](http://doxygen.postgresql.org/pg__backup__utils_8c.html#a9ce2ec4812a92cb6ab39f6e81e9173a9)) |
|                                          |                                          |
| static [BufferDesc](http://doxygen.postgresql.org/structBufferDesc.html) * | [BufferAlloc](http://doxygen.postgresql.org/bufmgr_8c.html#a78bfd80a6cacde30acf733a66ad494c0) ([SMgrRelation](http://doxygen.postgresql.org/smgr_8h.html#ab1753f7a9304d3a0b8cfb4d349441591) smgr, char relpersistence, [ForkNumber](http://doxygen.postgresql.org/relpath_8h.html#a4e49e3b48d6a980e40dbde579f89237d) forkNum, [BlockNumber](http://doxygen.postgresql.org/block_8h.html#a0be1c1ab88d7f8120e2cd2e8ac2697a1) blockNum, [BufferAccessStrategy](http://doxygen.postgresql.org/buf_8h.html#aa259999f7bb734ea3bb310d66d1f6f1c) strategy, [bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0)*foundPtr) |
|                                          |                                          |
| static void                              | [FlushBuffer](http://doxygen.postgresql.org/bufmgr_8c.html#ada5a531b19119f200e07f4dfa8c997c6) ([BufferDesc](http://doxygen.postgresql.org/structBufferDesc.html) *[buf](http://doxygen.postgresql.org/pg__test__fsync_8c.html#ac14417684334d01b4e1e807a19d92816), [SMgrRelation](http://doxygen.postgresql.org/smgr_8h.html#ab1753f7a9304d3a0b8cfb4d349441591) reln) |
|                                          |                                          |
| static void                              | [AtProcExit_Buffers](http://doxygen.postgresql.org/bufmgr_8c.html#a699f1f9fb44bd8e2203bd7b8857da316) (int code, [Datum](http://doxygen.postgresql.org/postgres_8h.html#a43879eb52e27b2413d7a05056a173ca6) [arg](http://doxygen.postgresql.org/pg__backup__utils_8c.html#a9ce2ec4812a92cb6ab39f6e81e9173a9)) |
|                                          |                                          |
| static void                              | [CheckForBufferLeaks](http://doxygen.postgresql.org/bufmgr_8c.html#a545f1e7e0730fb1dbb6cebf851e4df09) (void) |
|                                          |                                          |
| static int                               | [rnode_comparator](http://doxygen.postgresql.org/bufmgr_8c.html#a3d3538abb372816cf9bd5b3c8a6f3b0c) (const void *p1, const void *p2) |
|                                          |                                          |
| static int                               | [buffertag_comparator](http://doxygen.postgresql.org/bufmgr_8c.html#af25c9dfdc685ecb244e4368f2e4fab7d) (const void *p1, const void *p2) |
|                                          |                                          |
| static int                               | [ckpt_buforder_comparator](http://doxygen.postgresql.org/bufmgr_8c.html#a59bd03487e4600ab584970cd55924092) (const void *pa, const void *pb) |
|                                          |                                          |
| static int                               | [ts_ckpt_progress_comparator](http://doxygen.postgresql.org/bufmgr_8c.html#a6bd07ef2a3cb7c8e3c9b22653b1dcd51) ([Datum](http://doxygen.postgresql.org/postgres_8h.html#a43879eb52e27b2413d7a05056a173ca6) a, [Datum](http://doxygen.postgresql.org/postgres_8h.html#a43879eb52e27b2413d7a05056a173ca6) b, void *[arg](http://doxygen.postgresql.org/pg__backup__utils_8c.html#a9ce2ec4812a92cb6ab39f6e81e9173a9)) |
|                                          |                                          |
| [bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0) | [ComputeIoConcurrency](http://doxygen.postgresql.org/bufmgr_8c.html#a94f326ffc3cb39d26382ee102aced4f6) (int io_concurrency, double *target) |
|                                          |                                          |
| void                                     | [PrefetchBuffer](http://doxygen.postgresql.org/bufmgr_8c.html#afe3e061501dabd8ec39aa0307143ff46) ([Relation](http://doxygen.postgresql.org/relcache_8h.html#a16307ba9a89dc9c630f47ad0a913bc3c) reln, [ForkNumber](http://doxygen.postgresql.org/relpath_8h.html#a4e49e3b48d6a980e40dbde579f89237d) forkNum, [BlockNumber](http://doxygen.postgresql.org/block_8h.html#a0be1c1ab88d7f8120e2cd2e8ac2697a1) blockNum) |
|                                          |                                          |
| [Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) | [ReadBuffer](http://doxygen.postgresql.org/bufmgr_8c.html#af17af157a9a0c1a7ab6f141d6ffb685e) ([Relation](http://doxygen.postgresql.org/relcache_8h.html#a16307ba9a89dc9c630f47ad0a913bc3c) reln, [BlockNumber](http://doxygen.postgresql.org/block_8h.html#a0be1c1ab88d7f8120e2cd2e8ac2697a1) blockNum) |
|                                          |                                          |
| [Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) | [ReadBufferExtended](http://doxygen.postgresql.org/bufmgr_8c.html#adeac0f793ec0fd897c447fc3f64bf77a) ([Relation](http://doxygen.postgresql.org/relcache_8h.html#a16307ba9a89dc9c630f47ad0a913bc3c) reln, [ForkNumber](http://doxygen.postgresql.org/relpath_8h.html#a4e49e3b48d6a980e40dbde579f89237d) forkNum, [BlockNumber](http://doxygen.postgresql.org/block_8h.html#a0be1c1ab88d7f8120e2cd2e8ac2697a1) blockNum, [ReadBufferMode](http://doxygen.postgresql.org/bufmgr_8h.html#af2fba14fc04ffc8c72f1c250fa499cc0) mode, [BufferAccessStrategy](http://doxygen.postgresql.org/buf_8h.html#aa259999f7bb734ea3bb310d66d1f6f1c) strategy) |
|                                          |                                          |
| [Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) | [ReadBufferWithoutRelcache](http://doxygen.postgresql.org/bufmgr_8c.html#a7582df28e97567b75eea4e3ed7a4092c) ([RelFileNode](http://doxygen.postgresql.org/structRelFileNode.html) rnode, [ForkNumber](http://doxygen.postgresql.org/relpath_8h.html#a4e49e3b48d6a980e40dbde579f89237d) forkNum, [BlockNumber](http://doxygen.postgresql.org/block_8h.html#a0be1c1ab88d7f8120e2cd2e8ac2697a1) blockNum, [ReadBufferMode](http://doxygen.postgresql.org/bufmgr_8h.html#af2fba14fc04ffc8c72f1c250fa499cc0) mode, [BufferAccessStrategy](http://doxygen.postgresql.org/buf_8h.html#aa259999f7bb734ea3bb310d66d1f6f1c)strategy) |
|                                          |                                          |
| static void                              | [InvalidateBuffer](http://doxygen.postgresql.org/bufmgr_8c.html#ad2edb8166a9b40ba35214c474feafffb) ([BufferDesc](http://doxygen.postgresql.org/structBufferDesc.html) *[buf](http://doxygen.postgresql.org/pg__test__fsync_8c.html#ac14417684334d01b4e1e807a19d92816)) |
|                                          |                                          |
| void                                     | [MarkBufferDirty](http://doxygen.postgresql.org/bufmgr_8c.html#ab08d100d95c8e4d38e36241a56b53e69) ([Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) buffer) |
|                                          |                                          |
| [Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) | [ReleaseAndReadBuffer](http://doxygen.postgresql.org/bufmgr_8c.html#a1968079066a5e423edd3db9be35c7c86) ([Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) buffer, [Relation](http://doxygen.postgresql.org/relcache_8h.html#a16307ba9a89dc9c630f47ad0a913bc3c) relation, [BlockNumber](http://doxygen.postgresql.org/block_8h.html#a0be1c1ab88d7f8120e2cd2e8ac2697a1) blockNum) |
|                                          |                                          |
| [bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0) | [BgBufferSync](http://doxygen.postgresql.org/bufmgr_8c.html#a8119081a2ecd442a29abfd4bc34b710f) ([WritebackContext](http://doxygen.postgresql.org/structWritebackContext.html) *wb_context) |
|                                          |                                          |
| void                                     | [AtEOXact_Buffers](http://doxygen.postgresql.org/bufmgr_8c.html#a4cdec4507a8c0f87c54984fff8c01f4a) ([bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0) isCommit) |
|                                          |                                          |
| void                                     | [InitBufferPoolAccess](http://doxygen.postgresql.org/bufmgr_8c.html#ab0bb373e77241bbed5ddc60f44858bdd) (void) |
|                                          |                                          |
| void                                     | [InitBufferPoolBackend](http://doxygen.postgresql.org/bufmgr_8c.html#ae118a05ebf79cda1b4da1d82e8ace242) (void) |
|                                          |                                          |
| void                                     | [PrintBufferLeakWarning](http://doxygen.postgresql.org/bufmgr_8c.html#abcabc7aa4066d2d01ce1e26024f6a8ef) ([Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) buffer) |
|                                          |                                          |
| void                                     | [CheckPointBuffers](http://doxygen.postgresql.org/bufmgr_8c.html#a7ce49d5675330c80e71e676374e38fa0) (int flags) |
|                                          |                                          |
| void                                     | [BufmgrCommit](http://doxygen.postgresql.org/bufmgr_8c.html#a39492d568c7220b3abd05df3b6b1223f) (void) |
|                                          |                                          |
| [BlockNumber](http://doxygen.postgresql.org/block_8h.html#a0be1c1ab88d7f8120e2cd2e8ac2697a1) | [BufferGetBlockNumber](http://doxygen.postgresql.org/bufmgr_8c.html#a12ed528d88b55c558ed8d9c6eae22591) ([Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) buffer) |
|                                          |                                          |
| void                                     | [BufferGetTag](http://doxygen.postgresql.org/bufmgr_8c.html#ad0a770f05ea270a804d3affe32351d9e) ([Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) buffer, [RelFileNode](http://doxygen.postgresql.org/structRelFileNode.html) *rnode, [ForkNumber](http://doxygen.postgresql.org/relpath_8h.html#a4e49e3b48d6a980e40dbde579f89237d) *forknum, [BlockNumber](http://doxygen.postgresql.org/block_8h.html#a0be1c1ab88d7f8120e2cd2e8ac2697a1) *blknum) |
|                                          |                                          |
| [BlockNumber](http://doxygen.postgresql.org/block_8h.html#a0be1c1ab88d7f8120e2cd2e8ac2697a1) | [RelationGetNumberOfBlocksInFork](http://doxygen.postgresql.org/bufmgr_8c.html#a9b1d6345761ca7d5a9ef30f5a3d5cd82) ([Relation](http://doxygen.postgresql.org/relcache_8h.html#a16307ba9a89dc9c630f47ad0a913bc3c) relation, [ForkNumber](http://doxygen.postgresql.org/relpath_8h.html#a4e49e3b48d6a980e40dbde579f89237d) forkNum) |
|                                          |                                          |
| [bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0) | [BufferIsPermanent](http://doxygen.postgresql.org/bufmgr_8c.html#a8a80c85d7e8a28752a2fb905472aaad1) ([Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) buffer) |
|                                          |                                          |
| [XLogRecPtr](http://doxygen.postgresql.org/xlogdefs_8h.html#a9f798cf05369dc78cd7688060d2c5993) | [BufferGetLSNAtomic](http://doxygen.postgresql.org/bufmgr_8c.html#aa029f59f5cac09100d148579e5febdec) ([Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) buffer) |
|                                          |                                          |
| void                                     | [DropRelFileNodeBuffers](http://doxygen.postgresql.org/bufmgr_8c.html#a51df3dd0bbad48e423f04977e63b3a41) ([RelFileNodeBackend](http://doxygen.postgresql.org/structRelFileNodeBackend.html) rnode, [ForkNumber](http://doxygen.postgresql.org/relpath_8h.html#a4e49e3b48d6a980e40dbde579f89237d) forkNum, [BlockNumber](http://doxygen.postgresql.org/block_8h.html#a0be1c1ab88d7f8120e2cd2e8ac2697a1) firstDelBlock) |
|                                          |                                          |
| void                                     | [DropRelFileNodesAllBuffers](http://doxygen.postgresql.org/bufmgr_8c.html#a955ca00097bd4071c46ab6f1e6237ec2) ([RelFileNodeBackend](http://doxygen.postgresql.org/structRelFileNodeBackend.html) *rnodes, int nnodes) |
|                                          |                                          |
| void                                     | [DropDatabaseBuffers](http://doxygen.postgresql.org/bufmgr_8c.html#a0cd4782c314b706ece06fc5d263b6125) ([Oid](http://doxygen.postgresql.org/postgres__ext_8h.html#a545a1e974d4c848ee2e2dcfdd503335f) dbid) |
|                                          |                                          |
| void                                     | [FlushRelationBuffers](http://doxygen.postgresql.org/bufmgr_8c.html#ac46973f353ef3fab852b5bd90ed87dde) ([Relation](http://doxygen.postgresql.org/relcache_8h.html#a16307ba9a89dc9c630f47ad0a913bc3c) rel) |
|                                          |                                          |
| void                                     | [FlushDatabaseBuffers](http://doxygen.postgresql.org/bufmgr_8c.html#ad1d73cc66c32ec425d48ac54fae90961) ([Oid](http://doxygen.postgresql.org/postgres__ext_8h.html#a545a1e974d4c848ee2e2dcfdd503335f) dbid) |
|                                          |                                          |
| void                                     | [FlushOneBuffer](http://doxygen.postgresql.org/bufmgr_8c.html#afef43118653f4b957f7d11b6b839148d) ([Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) buffer) |
|                                          |                                          |
| void                                     | [ReleaseBuffer](http://doxygen.postgresql.org/bufmgr_8c.html#a9bc03ec5d254ee2f9fe05aa3536bb037) ([Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) buffer) |
|                                          |                                          |
| void                                     | [UnlockReleaseBuffer](http://doxygen.postgresql.org/bufmgr_8c.html#aa3b15009fb5e6fa3440d2e8fc6eb6e3f) ([Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) buffer) |
|                                          |                                          |
| void                                     | [IncrBufferRefCount](http://doxygen.postgresql.org/bufmgr_8c.html#a098140e762fc3118a109db6755a4a665) ([Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) buffer) |
|                                          |                                          |
| void                                     | [MarkBufferDirtyHint](http://doxygen.postgresql.org/bufmgr_8c.html#ac40bc4868e97a49a25dd8be7c98b6773) ([Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) buffer, [bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0) buffer_std) |
|                                          |                                          |
| void                                     | [UnlockBuffers](http://doxygen.postgresql.org/bufmgr_8c.html#a8fd149e71f4c9d074908ca6d793e7fb7) (void) |
|                                          |                                          |
| void                                     | [LockBuffer](http://doxygen.postgresql.org/bufmgr_8c.html#ac2e9eb8cc03820be51a779e988c73635) ([Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) buffer, int mode) |
|                                          |                                          |
| [bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0) | [ConditionalLockBuffer](http://doxygen.postgresql.org/bufmgr_8c.html#a97ab5d75fbb259986a58ae7acde00922) ([Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) buffer) |
|                                          |                                          |
| void                                     | [LockBufferForCleanup](http://doxygen.postgresql.org/bufmgr_8c.html#ac199b0a6778f158b7a5ed5716d36ac3b) ([Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) buffer) |
|                                          |                                          |
| [bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0) | [HoldingBufferPinThatDelaysRecovery](http://doxygen.postgresql.org/bufmgr_8c.html#aa1beb2f86764639969c2dbb167ece0bf) (void) |
|                                          |                                          |
| [bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0) | [ConditionalLockBufferForCleanup](http://doxygen.postgresql.org/bufmgr_8c.html#af785065350368fa9bcf9468a9a0f2c69) ([Buffer](http://doxygen.postgresql.org/buf_8h.html#a13e825fcb656b677bc8fa431fd8582fe) buffer) |
|                                          |                                          |
| void                                     | [AbortBufferIO](http://doxygen.postgresql.org/bufmgr_8c.html#ab3ec091f30f272584886ea5ad06bbaa8) (void) |
|                                          |                                          |
| [uint32](http://doxygen.postgresql.org/c_8h.html#a1134b580f8da4de94ca6b1de4d37975e) | [LockBufHdr](http://doxygen.postgresql.org/bufmgr_8c.html#ae22815a16bdc95944b8e5b1202998b45) ([BufferDesc](http://doxygen.postgresql.org/structBufferDesc.html) *desc) |
|                                          |                                          |
| void                                     | [WritebackContextInit](http://doxygen.postgresql.org/bufmgr_8c.html#aab73dfd9c1eae42581e533ef7a589d79) ([WritebackContext](http://doxygen.postgresql.org/structWritebackContext.html) *context, int *max_pending) |
|                                          |                                          |
| void                                     | [ScheduleBufferTagForWriteback](http://doxygen.postgresql.org/bufmgr_8c.html#a7307f78b4d2d43d418c505deab258539) ([WritebackContext](http://doxygen.postgresql.org/structWritebackContext.html) *context, [BufferTag](http://doxygen.postgresql.org/buf__internals_8h.html#aee50fc0771d3e2fc859dfeb52381a367) *tag) |
|                                          |                                          |
| void                                     | [IssuePendingWritebacks](http://doxygen.postgresql.org/bufmgr_8c.html#aec080df36ed5f516b606798746d4321c) ([WritebackContext](http://doxygen.postgresql.org/structWritebackContext.html) *context) |
|                                          |                                          |
| void                                     | [TestForOldSnapshot_impl](http://doxygen.postgresql.org/bufmgr_8c.html#a6284f743f2a83800be75a5dc0e254b1d) ([Snapshot](http://doxygen.postgresql.org/snapshot_8h.html#a09bb8105293579973411e4b7adfa7969) snapshot, [Relation](http://doxygen.postgresql.org/relcache_8h.html#a16307ba9a89dc9c630f47ad0a913bc3c) relation) |
|                                          |                                          |

| Variables                                |                                          |
| ---------------------------------------- | ---------------------------------------- |
| [bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0) | [zero_damaged_pages](http://doxygen.postgresql.org/bufmgr_8c.html#a32a9eb37e328bf5b6f4c824e081ad5a9) = [false](http://doxygen.postgresql.org/ecpglib_8h.html#a65e9886d74aaee76545e83dd09011727) |
| **这5个变量： GUC variables**                 | 标识是否有damaged的page                        |
| int                                      | [bgwriter_lru_maxpages](http://doxygen.postgresql.org/bufmgr_8c.html#af94a4acfdafa8c80b9afe46cf14299bb) = 100 |
| double                                   | [bgwriter_lru_multiplier](http://doxygen.postgresql.org/bufmgr_8c.html#ab3411d9b50c5edc6c58538a4d83643e0) = 2.0 |
| [bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0) | [track_io_timing](http://doxygen.postgresql.org/bufmgr_8c.html#a006eef129cd18a8ee4a765499c294ad2) = [false](http://doxygen.postgresql.org/ecpglib_8h.html#a65e9886d74aaee76545e83dd09011727) |
| int                                      | [effective_io_concurrency](http://doxygen.postgresql.org/bufmgr_8c.html#ad6687c61be4b0358323dcbb635323ba7) = 0 |
| GUC：guard unified configuration/Grand unified configuration，用于在multilevel控制postgres。比如：global level 和 local level | **下面三个变量是 关于trigger write back for buffer written** 的GUC variables。 |
| int                                      | [checkpoint_flush_after](http://doxygen.postgresql.org/bufmgr_8c.html#aa35adb0ac46a667c6e64d2cb0f0a35ab) = 0 |
| int                                      | [bgwriter_flush_after](http://doxygen.postgresql.org/bufmgr_8c.html#a9b685f1c73679f02286a70c25f04e9f0) = 0 |
| int                                      | [backend_flush_after](http://doxygen.postgresql.org/bufmgr_8c.html#ab4ccf2cf0bc9f3dfb22fe7fa7fc2649d) = 0 |
|                                          |                                          |
| int                                      | [target_prefetch_pages](http://doxygen.postgresql.org/bufmgr_8c.html#a5746d6eca9dc8305c84136403167f8e6) = 0 |
|                                          | zero 表示 从不提前fetch。 这个值只用于不属于那些有自己的parameter set的tablespace |
| static [BufferDesc](http://doxygen.postgresql.org/structBufferDesc.html) * | [InProgressBuf](http://doxygen.postgresql.org/bufmgr_8c.html#a7943fe96204889e4e259b9c339ff4be6) = [NULL](http://doxygen.postgresql.org/c_8h.html#a070d2ce7b6bb7e5c05602aa8c308d0c4)   //用于标识正在处理的buf |
| **上下这两个变量**： 与**StartBufferIO**的局部状态和相关函数有关。 |                                          |
| static [bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0) | [IsForInput](http://doxygen.postgresql.org/bufmgr_8c.html#ab8ead2b9392438f4baee876a2a06e47c)  //判断是否用于输入 |
|                                          |                                          |
| static [BufferDesc](http://doxygen.postgresql.org/structBufferDesc.html) * | [PinCountWaitBuf](http://doxygen.postgresql.org/bufmgr_8c.html#ab9cc8e426666412e3e4f221777cc4edf) = [NULL](http://doxygen.postgresql.org/c_8h.html#a070d2ce7b6bb7e5c05602aa8c308d0c4) |
|                                          | local state for LockBufferForCleanup     |
|                                          |                                          |
| static struct [PrivateRefCountEntry](http://doxygen.postgresql.org/structPrivateRefCountEntry.html) | [PrivateRefCountArray](http://doxygen.postgresql.org/bufmgr_8c.html#ae36390f487059e0609d39fe05f84de6d) [[REFCOUNT_ARRAY_ENTRIES](http://doxygen.postgresql.org/bufmgr_8c.html#a8c9b241ffedfeab215e054d26e74457f)] |
| static [HTAB](http://doxygen.postgresql.org/structHTAB.html) * | [PrivateRefCountHash](http://doxygen.postgresql.org/bufmgr_8c.html#af4ae93ace0d49a450180b13696d30021) = [NULL](http://doxygen.postgresql.org/c_8h.html#a070d2ce7b6bb7e5c05602aa8c308d0c4) |
| static [int32](http://doxygen.postgresql.org/c_8h.html#a43d43196463bde49cb067f5c20ab8481) | [PrivateRefCountOverflowed](http://doxygen.postgresql.org/bufmgr_8c.html#af54fc90aa3b77b81b58a86481426e4e9) = 0 |
| static [uint32](http://doxygen.postgresql.org/c_8h.html#a1134b580f8da4de94ca6b1de4d37975e) | [PrivateRefCountClock](http://doxygen.postgresql.org/bufmgr_8c.html#a20fabd3d6f193085d7258ad03294cf97) = 0 |
| static [PrivateRefCountEntry](http://doxygen.postgresql.org/structPrivateRefCountEntry.html) * | [ReservedRefCountEntry](http://doxygen.postgresql.org/bufmgr_8c.html#ac320d7f367c4d71f95431f5d5aa93451) = [NULL](http://doxygen.postgresql.org/c_8h.html#a070d2ce7b6bb7e5c05602aa8c308d0c4) |



>Backend-Private refcount management :后台私有引用计数管理策略
>
>除了shared refcount， 每个buffer也都已一个私有的引用计数，用于跟踪当前buffer在当前process中被pin的次数。每个process只修改shared refcount by one while 修改 private refcount many times。也用于检查在transaction结束或者exit的时候， 保证no buffer还在pinned。
>
>
>
>不使用整个NBuffers entries，而是使用一个小的sequential searched array  **PrivateRefCountArray** 和 一个 overflow的hashtbale：**PrivateRefCounthash** 来跟踪local pins。
>
>
>
>大多数情况下，the number of pinned buffer will **not** exceed REFCOUNT_ARRAY_ENTRIES(8).
>
>
>
>如何使用：
>
>to enter a buffer into the refcount tracking mechanism ,
>
>* first reserve a free entry using **ReservePrivateRefCountEntry()**
>* later, if necessary, fill it with **NewPrivateRefCountEntry()**
>* the above two steps are split, it lets us avoid doing memory allocations in **NewPricvateCountEntry()**. 这是很重要的，因为有时它会在持有（hold）一个spinlock的时候被调用。