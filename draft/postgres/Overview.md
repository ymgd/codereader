# PostgreSQL Source Code

for more info, click [here](http://doxygen.postgresql.org/buf__init_8c.html#ae973fd1ced927338dad757d854d2f348).

**Tools**:
* [Bison](https://www.gnu.org/software/bison/)
* [Flex](http://www.adobe.com/products/flex.html)
* [CVS](http://www.nongnu.org/cvs/)
* [autotools](https://www.gnu.org/software/automake/manual/html_node/Autotools-Introduction.html)
* [GDB](https://www.gnu.org/software/gdb/)
* [Sublime Text](https://www.sublimetext.com/)

###Storage Management
Table -> Files
* Tables and indexes are stored in normal operating-system files
* Each table/index divided into "segments" of at most 1GB
* Tablespaces just control the filesystem location of segments.

Files -> Blocks
* Each file is divided into **blocks** of BLCKSZ betes each
  * 8192 by default; compile-time constant.
* Blocks consist of **items**, such as **heap tuples(in tables)**, or **index entries(in indexes)**, along with **metadata**.
* Tuple versions uniquely identified by triple(r, p, i):
  * relation OID
  * block number
  * offset within block
  known as 'ctid'.

###The Buffer Manager
* **Almost all I/O is not done directly: to access a _page_, a process asks the _buffer manager_ for it**.
* The buffer manager implements a hash table in shared memory, maping page identifiers -> buffers.
  * if the requested page is in shared_buffers, return it 
  * Otherwise, ask the kernel for it and stash it in shared_buffers.
    * If no free buffers, replace an existing one (which one?)
    * The kernel typically does its own I/O caching as well
* Keep a **pin** on the page, to ensure it isn't replaced while in use.

###Concurrency Control
####Table-Level Locks
* Also known as 'lmgr locks', 'heavyweight locks'
* Protect entire tables against concurrent DDL operations
* Many different **lock modes**; matrix for determining if two locks conflict.
* Automatic deadlock detection and resolution

####Row-level Locks
* Writers don't block readers: MVCC
* Writers must **block writers**: implemented via **row-level locks**
* Implemented by marking the row itself(on disk)
* ALso used for SELECT FOR UPDATE, FOR SHARE


####Concurrency Control: Low-Level Locks
#####LWLocks("Latches")
* Protect shared data structures against concurrent access
* Tow locks modes: shared and exclusive(read/writer)
* No deadlock detection: should only be held for short durations.

#####Spinlocks
* LWLocks are implemented on top of spinlocks, which are in turn a thin layer on top of an atomic test-and-set(TAS) primitive provided by the platform.
* If an LWLock is contended, waiting is done via blocking on a SysV semaphore; spinlocks just busywait, then micro-sleep.
