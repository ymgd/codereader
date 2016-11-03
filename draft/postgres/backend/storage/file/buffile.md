## src/backend/storage/file/

### buffile.c

1. include dependency graph for buffile.c

   ![](http://doxygen.postgresql.org/buffile_8c__incl.png)

用于管理large buffered files， 主要是临时文件。

 ![Screen Shot 2016-08-27 at 8.49.04 AM](/Users/poodar/Desktop/Screen Shot 2016-08-27 at 8.49.04 AM.png)

 **重要结构体**：

 将Buffiles分解为GB大小的segment，而不是RELSG_SIZE的大小![Screen Shot 2016-08-27 at 8.50.48 AM](/Users/poodar/Desktop/Screen Shot 2016-08-27 at 8.50.48 AM.png)



