## src/backend/storage/file/

### copydir.c

1. include dependency graph ：

   ![](http://doxygen.postgresql.org/copydir_8c__incl.png)

用于复制一个目录。



| Macros  |                                          |
| ------- | ---------------------------------------- |
| #define | [COPY_BUF_SIZE](http://doxygen.postgresql.org/copydir_8c.html#ae7e817bb96172e443b859c0f81b8785a)   (8 * BLCKSZ) |
|         | Referenced by **copy_file()**            |

| Functions |                                          |
| --------- | ---------------------------------------- |
| void      | [copydir](http://doxygen.postgresql.org/copydir_8c.html#a191b8253259b94ec3bf51eef4cccbf4b) (char *fromdir, char *todir, [bool](http://doxygen.postgresql.org/c_8h.html#ad5c9d4ba3dc37783a528b0925dc981a0) recurse)  //copy a directory. |
|           | **Usage**: copydir (formdir, todir, true); |
| void      | [copy_file](http://doxygen.postgresql.org/copydir_8c.html#a1b75eb08826cde2382df5780f35c8d8d) (char *fromfile, char *tofile) |

```C
struct DIR {
  char* dirname;
  struct dirent ret;    //used to return to caller.
  HANDLE handle;
};
struct dirent {
  long d_ino;
  unsigned short d_reclen;
  unsigned short d_namlen;
  char d_name[MAX_PATH];
}
#define MAXPGPATH 1024   // the standard size of a pathname buffer in PostgreSQL.
```

