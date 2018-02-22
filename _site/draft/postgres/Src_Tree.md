##Organization of Source Tree
* doc/: documentation, FAQs
* src/
  * bin/: client programs(psql, pg_dump, ...)
  * include/: headers
   *catalog/: system catalog definitions
  * interfaces/: libpq, ecpg
  * pl/: procedural languages(PL/PgSQL, PL/Perl, ...)
  * test/regress/: SQL regression tests.
* Makefiles
 * Makefile per directory(recursive make)
 * src/makefiles has platform-specific Makefiles
 * src/Makefile.global.in is the top-level Makefile

## Backend Source Tree(src/backend)
* access/: index implementations, heap access manager, transaction management, write-ahead log
* commands/: implementation of **DDL commands**
* executor/:executor logic, implementation of **executor nodes**
* libpq/: implementation of backend side of FE/BE protocol
* optimizer/: query planner
* parser/: lexer, parser, analysis phase
* postmaster/: postmaster, stats daemon, AV daemon, ...
* rewrite/: application of **query rewrite rules**
* storage/: shmem, locks ,bufmgr, storage management, ...
* tcop/: "traffic cop", FE/BE query loop, dispatching from protocol commands -> implementation
* utils/:
 * adt/: builtin data types, functions, operators
 * cache/: caches for system catalog lookups, query plans
 * hash/: in-memory hash tables
 * mmgr/: memory management
 * sort/: external sorting, TupleStore.
