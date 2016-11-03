## src/backend/storage/file/

## fd.c

* include dependency graph for fd.c

  ![](http://doxygen.postgresql.org/fd_8c__incl.png)

**Virtual file descriptor code.**

> **tablespace** in postgresql 允许administrator定义表示数据库对象的文件的文件在文件系统中存储的位置。一旦created，一个tablespace可以被创建database object时的name所引用（referenced）。

#### NOTES：

* 这个文件管理一个virtual file descriptor（VFDs）的cache。由于server可能为多种原因打开许多file descriptors，这导致很容易就会超过系统的单个进程可以打开的文件上限。（this is around 256 on many modern operating systems, but can be as low as 32 on others).
* VFDs使用LRU（Least Recently Used）算法管理，伴随着真是的操作系统文件描述符的按需开闭。显然，如果一个routine使用这些API(interfaces in fd.c)打开文件，那么所有的后续操作也必须通过这些接口，（**the _File_ type is not a read file descriptor.** 而不是使用系统的C library。

#### INTERFACE ROUTINES：

* **PathNameOpenFile** and **OpenTemporatyFile** are used to open virtual files. 
  * 一个被OpenTemporaryFile打开的 **File** 会在File关闭，或者显式或隐式的事物结束时自动删除。
  * PathNameOpenFile用于那些长时间被打开的文件。例如relation files。由调用者负责关闭它们，不同于OpenTemporaryFile， PahtNameOpenFile没有自动关闭的机制。
* **AllocateFile**, **AllocateDir**, **OpenPipeStream** and **OpenTransientFile**分别是_fopen_， _opendir_, _popen_,  _open_的wrappers(封装）。
  * 它们像原生的函数一样工作，除了handle是在当前的sub transaction 中registered，也会被自动关闭。它们被用于**short operations** 例如读取配置文件。任何时候，使用它们可以打开的文件数量是有限的。
* **BasicOpenFile** 是open的thin wrapper，可以在必要的时候通过virtual file descriptor释放file descriptor，没有自动清理fd的机制。这是caller的职责：在调用结束时关闭file descriptor。


---

* _Data Structures_

  * struct  	**vfd**

    * struct  **AllocateDesc**

    AllocateFile/Dir 和 OpenTransitentFile相关的Desc（file， dir， fd）

  ​

* _Macros_

  * \#define 	**NUM_RESERVED_FDS**   10

    需要为system（）预留一部分file descriptor，供不使用fd.c的其他进程使用。

    * \#define **FD_MINFREE**   10

     如果剩余少于FD_MINFREE的可用FDs，阻塞。

    * \#define DO_DB(A)   ((void) 0)

    * \#define **VFD_CLOSED**   (-1)

    * \#define **FileIsValid**(file)   ((file) > 0 && (file) < (int) SizeVfdCache && VfdCache[file].fileName != NULL)

    * \#define **FileIsNotOpen**(file)   (VfdCache[file].fd == VFD_CLOSED)

    * \#define FileUnknownPos   ((off_t) -1)

    * \#define FD_TEMPORARY   (1 << 0)       ``` /* T = delete when closed */```
    * \#define FD_XACT_TEMPORARY   (1 << 1)      ```/* T = delete at eoXact */```

* _Typedefs_

  typedef struct vfd 	**Vfd**  //fd对应的vdd，有fd的编号，状态，所有者，当前位置，文件大小文件名文件flag文件mode等属性，并唯一个lru的双向链表。

* _Enumerations_

  enum  	AllocateDescKind {

  ​						 AllocateDescFile, 

  ​						AllocateDescPipe,

  ​						 AllocateDescDir,

  ​						 AllocateDescRawFD  }

  ​

* Variables

  * int 	max_files_per_process = 1000
  * int 	max_safe_fds = 32

  ​

  * VFD 数组的pointer和size。可以按需增长。File值被所引导这个数组中，VfdCache[0]不是一个可用的VFD，而是链表的header。

    static **Vfd *** 	VfdCache

    static **Size** 	SizeVfdCache = 0
    ​

  * static int 	nfile = 0   //已知被VFD entries使用的 file descriptor的数量

  * static bool 	have_xact_temporary_files = false   //flage，用于标识是是否值得scanning vdfcache来寻找将被关闭的file。

  * static uint64 	temporary_files_size = 0

    **tracks the total files of all temporary files.**

  * static int 	numAllocatedDescs = 0

    static int 	maxAllocatedDescs = 0

    static AllocateDesc * 	allocatedDescs = NULL

  * static long 	tempFileCounter = 0

    Number of temporary files opened during the current session.

  * static Oid * 	tempTableSpaces = NULL

    static int 	numTempTableSpaces = -1

    static int 	nextTempTableSpace = 0

    **temp tablespace的OIDs的数组**， 当numTempTableSpaces = -1 时，表示还没有在当前transaction中set。

    ​

* Functions

  ```C
  /*--------------------
   *
   * Private Routines
   *
   * Delete          - delete a file from the Lru ring
   * LruDelete       - remove a file from the Lru ring and close its FD
   * Insert          - put a file at the front of the Lru ring
   * LruInsert       - put a file at the front of the Lru ring and open it
   * ReleaseLruFile  - Release an fd by closing the last entry in the Lru ring
   * ReleaseLruFiles - Release fd(s) until we're under the max_safe_fds limit
   * AllocateVfd     - grab a free (or new) file record (from VfdArray)
   * FreeVfd         - free a file record
   *
   * The Least Recently Used ring is a doubly linked list that begins and
   * ends on element zero.  Element zero is special -- it doesn't represent
   * a file and its "fd" field always == VFD_CLOSED.  Element zero is just an
   * anchor that shows us the beginning/end of the ring.
   * Only VFD elements that are currently really open (have an FD assigned) are
   * in the Lru ring.  Elements that are "virtually" open can be recognized
   * by having a non-null fileName field.
   *
   * example:
   *
   *     /--less----\                /---------\
   *     v           \              v           \
   *   #0 --more---> LeastRecentlyUsed --more-\ \
   *    ^\                                    | |
   *     \\less--> MostRecentlyUsedFile   <---/ |
   *      \more---/                    \--less--/
   *
   *--------------------
   */
  ```

  * static void 	Delete (File file)   //**delete a file from the Lru ring**

    ```C
    typedef int File;
    ```

    VfdCache[vfdP->[lruLessRecently](http://doxygen.postgresql.org/structvfd.html#a454e6545774d4f62eb32a9fff21157b3)].[lruMoreRecently](http://doxygen.postgresql.org/structvfd.html#a540311ab0f6283c5d66bc01d3e69fd39) = vfdP->[lruMoreRecently](http://doxygen.postgresql.org/structvfd.html#a540311ab0f6283c5d66bc01d3e69fd39);

    VfdCache[vfdP->[lruMoreRecently](http://doxygen.postgresql.org/structvfd.html#a540311ab0f6283c5d66bc01d3e69fd39)].[lruLessRecently](http://doxygen.postgresql.org/structvfd.html#a454e6545774d4f62eb32a9fff21157b3) = vfdP->[lruLessRecently](http://doxygen.postgresql.org/structvfd.html#a454e6545774d4f62eb32a9fff21157b3); 

    **static void 	LruDelete (File file)**  同理。remove a file from the Lru ring and **close its FD**

  ​

  * static void 	Insert (File file）   //**put a file at the front of the Lru ring**

    static int 	LruInsert (File file)    //put a file at the front of the Lru ring **and open it**

    ```C
    vfdP = &VfdCache[file];

    vfdP->lruMoreRecently = 0;
    vfdP->lruLessRecently = VfdCache[0].lruLessRecently;
    VfdCache[0].lruLessRecently = file;
    VfdCache[vfdP->lruLessRecently].lruMoreRecently = file;
    ```

    ​

  * static bool 	ReleaseLruFile (void)  //Release an fd by **closing the last entry in the Lru ring**

    static void 	ReleaseLruFiles (void)  //Release fd(s) until we're **under the max_safe_fds limit**

  ​

  * static File 	AllocateVfd (void)  //**grab a free (or new) file record (from VfdArray)**

  ​

  * static void 	FreeVfd (File file)  //**free a file record.**

  ​
  static int 	FileAccess (File file)
  static File 	OpenTemporaryFileInTablespace (Oid tblspcOid, bool rejectError)
  static bool 	reserveAllocatedDesc (void)
  static int 	FreeDesc (AllocateDesc *desc)
  static struct dirent * 	ReadDirExtended (DIR *dir, const char *dirname, int elevel)

  ​
  static void 	AtProcExit_Files (int code, Datum arg)
  static void 	CleanupTempFiles (bool isProcExit)
  static void 	RemovePgTempFilesInDir (const char *tmpdirname)
  static void 	RemovePgTempRelationFiles (const char *tsdirname)
  static void 	RemovePgTempRelationFilesInDbspace (const char *dbspacedirname)
  static bool 	looks_like_temp_rel_name (const char *name)

  ​
  static void 	walkdir (const char *path, void(*action)(const char *fname, bool isdir, int elevel), bool process_symlinks, int elevel)
  static void 	datadir_fsync_fname (const char *fname, bool isdir, int elevel)

  ​

  * static int 	**fsync_fname_ext** (const char *fname, bool isdir, bool ignore_perm, int elevel)

    try to fsync a file or directory.

  static int 	fsync_parent_path (const char *fname, int elevel)

  ​

  //**pg_sync** do sync with or without writethrough.

  * int 	**pg_fsync** (int fd)

    根据id和context调用pg_fsync_writethrough 或者pg_fsync_no_writethrough

    int 	pg_fsync_no_writethrough (int fd)
    int 	pg_fsync_writethrough (int fd)

  ​
  //**pg_fdatasync**, same as fdatasync，只不过在enableFsync off时，不做任何事。

  * int 	**pg_fdatasync** (int fd)

    判断enbaleFsync，根据平台不同，选择调用fdatasync或者fsync

  ​	

  //**pg_flush_data**:  建议OS将被标记为dirty的数据flush到disk。

  * void 	**pg_flush_data** (int fd, off_t offset, off_t nbytes)

    file flushing主要用于避免对后续的fsync或者fdatasync产生影响。因此，**不应该在fsync在disable时触发。**

  ​

  * void 	**fsync_fname** (const char *fname, bool isdir)

    fsync a file or a directory, handling errors properly.

    ​

  * int 	**_durable_rename_** (const char *oldfile, const char *newfile, int elevel)

    * rename的封装，提供了fsync需要的耐用性。

    * 它确保了在返回后，遇到crash时，rename file的效果可以被保持。如果这个routine运行过程中崩溃，要么留下预先存在的状态，要么留下移动后的文件，绝对不会是这两个的中间过渡状态。

    * **它通过在重命名之前，在老文件名和可能存在的目标文件名上使用fsync，然后是targetfile 和 directory**。

    * **注意**： 重命名不能再跨多个任意目录中使用，因为他们可能处于不同的文件系统中。因此，该routine**不支持跨目录重命名**。

    * 提供**错误日志**，根据调用者的severity。

    * **执行过程**：

      首先fsync old path和targetpath，保证它们被正确的持久化到硬盘。syncing target file不是必须的，但它可以使处理crashes更容易。因为这可以保证在一次crash之后，sourcefile和targetfile都存在。

      static int **fsync_fname_ext**(const char *fname, bool isdir, bool ignore_perm, int elevel)

      上面这个函数是durable_rename的核心函数调用。

      * 首先fsync old file
      * 然后打开一个新文件，进行一些列判断是否打开成功。
      * 没有错误的话，重命名oldfile with new file。
      * 最后将已经重命名的newfile 调用fsync_fname_ext持久化到disk。调用fysnc_parent_path和它包含的目录。

  * int 	**durable_link_or_rename** (const char *oldfile, const char *newfile, int elevel)

    类似于durable_rename（），只不过该函数会尝试在不重写targetfile的情况下rename。

    崩溃会留下**two links** to the **target file**.

  ​

  * void 	**InitFileAccess** (void)

    用于在backend startup时，初始化该模块。（fd)

    主要工作是初始化VfdCache数组。然后register一个pro-exit的hook，来保证临时文件在退出时被释放掉：**on_proc_exit(AtProcExit_Files, 0)**。

  ​

  * static void 	**count_usable_fds** (int max_to_probe, int *usable_fds, int *already_open)

    count how many FDs the system will let us open, and estimate how many are already open.

    Return results.  **usable_fds** is just the **number of successful dups**. We assume that the system **limit is highestfd+1** (remember **0 is a legal FD number**) and so **already_open** is highestfd+1 - usable_fds.

    ​

  * void 	**set_max_safe_fds** (void)

    设置fd.c被允许使用的file descriptor的数目的最大值

    * 使用count_usable_fds获取usable_fds和already_open的数目，然后减去系统预留的，在检查max_safe_fds不小于**FD_MINFREE**.

  ​

  * int 	BasicOpenFile (FileName fileName, int fileFlags, int fileMode)

    pg 唯一一个直接调用系统open()的入口，所有其他routine使用vfd定义的interface。

    **return fd = open(fileName, fileFlags, fileMode);**

  ​

  File 	PathNameOpenFile (FileName fileName, int fileFlags, int fileMode)
  File 	OpenTemporaryFile (bool interXact)
  void 	FileClose (File file)
  int 	FilePrefetch (File file, off_t offset, int amount)
  void 	FileWriteback (File file, off_t offset, off_t nbytes)
  int 	FileRead (File file, char *buffer, int amount)
  int 	FileWrite (File file, char *buffer, int amount)
  int 	FileSync (File file)
  off_t 	FileSeek (File file, off_t offset, int whence)
  int 	FileTruncate (File file, off_t offset)
  char * 	FilePathName (File file)
  int 	FileGetRawDesc (File file)
  int 	FileGetRawFlags (File file)
  int 	FileGetRawMode (File file)
  FILE * 	AllocateFile (const char *name, const char *mode)
  int 	OpenTransientFile (FileName fileName, int fileFlags, int fileMode)
  FILE * 	OpenPipeStream (const char *command, const char *mode)
  int 	FreeFile (FILE *file)
  int 	CloseTransientFile (int fd)
  DIR * 	AllocateDir (const char *dirname)
  struct dirent * 	ReadDir (DIR *dir, const char *dirname)
  int 	FreeDir (DIR *dir)
  int 	ClosePipeStream (FILE *file)
  void 	closeAllVfds (void)
  void 	SetTempTablespaces (Oid *tableSpaces, int numSpaces)
  bool 	TempTablespacesAreSet (void)
  Oid 	GetNextTempTableSpace (void)
  void 	AtEOSubXact_Files (bool isCommit, SubTransactionId mySubid, SubTransactionId parentSubid)
  void 	AtEOXact_Files (void)
  void 	RemovePgTempFiles (void)
  void 	SyncDataDirectory (void)