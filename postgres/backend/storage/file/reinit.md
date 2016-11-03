## src/backend/storage/file/

### reinit.c

1. include dependency graph for reinit.c

   ![](http://doxygen.postgresql.org/reinit_8c__incl.png)

用于 unlogged的relations的再初始化（reinitialization）。

---

```C
#define OIDCHARS        10      /* max chars printed by %u */
typedef struct {
    char        oid[OIDCHARS + 1];
} unlogged_relation_entry;
```





| Data Structures |                             |
| --------------- | --------------------------- |
| struct          | **unlogged_relation_entry** |

| Functions                                |                                          |
| ---------------------------------------- | ---------------------------------------- |
| static void                              | [ResetUnloggedRelationsInTablespaceDir](http://doxygen.postgresql.org/reinit_8c.html#a2beb89405cca25fed43ee6a308671ad6) (const char *tsdirname, int op) |
|                                          | //用于为**ResetUnloggedRelations**准备一个_tablespace_. |
| static void                              | [ResetUnloggedRelationsInDbspaceDir](http://doxygen.postgresql.org/reinit_8c.html#af0b83d337c7551d2c07f1e99b4615189) (const char *dbspacedirname, int op) |
|                                          |                                          |
| static [bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0) | [parse_filename_for_nontemp_relation](http://doxygen.postgresql.org/reinit_8c.html#a6d4775cb2bdcac533a58a5f64a5d8034) (const char *[name](http://doxygen.postgresql.org/encode_8c.html#a8f8f80d37794cde9472343e4487ba3eb), int *oidchars, [ForkNumber](http://doxygen.postgresql.org/relpath_8h.html#a4e49e3b48d6a980e40dbde579f89237d) *fork) |
|                                          |                                          |
| void                                     | [ResetUnloggedRelations](http://doxygen.postgresql.org/reinit_8c.html#a5497e1b3d8e5145ab7a2a2cf9e3d4c76) (int op) |