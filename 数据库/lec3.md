# 数据库的结构

Query planning 查询规划

Operator execution 运算符执行

access methods 访问方法

buffer pool manager 缓冲池管理器

disk manager 磁盘管理器

易失存储:支持随机寻址(字节可寻址)

非易失存储:块可寻址，只能得到数据块或数据页。同时具有更强的顺序访问能力，更有效率读取一段连续块的内容

非易失内存(傲腾)，具有可字节寻址能力，并且能持久

速度

![image-20230310105456743](C:\Users\xiasui\Desktop\cmu14-445\image-20230310105456743.png)

在数据库中，通过缓冲区来获取数据。

在os中，用mmap来映射地址

mmap问题:操作系统不知道某些page必须要在其他page执行前，先从内存刷到磁盘中。

有的数据库需要奖描述page中内容的元数据，和这些内容数据一起保存在page中。这样丢失一个page，也不影响其他page

大部分操作系统一个page只会存储一类数据

数据库的page通常是512B-16KB

os的page通常是4KB

Hardware Page 通常4KB

Hardware Page是执行原子写入存储设备的最低底层的东西(原子)。

## heap file organization

heap文件是无序page集合，能存储tuple数据，不保证顺序。

链表可以实现(但不好)，header两个指针，分别指向空余page和数据page的双向链表。



每个page里有一个header，包含page大小,checksum，dbms版本。

(糟糕的方法:)通过存储的tuple数量，当想要插入一个新tuple就能计算出想要跳转的偏移量是多少(长度固定的情况下)。当想要删除时就要移动所有数据，或产生内存碎片并且如果长度不固定，就不能直接插入到中间位置。

(好方法:slotted page)在page顶部有一个slot数组，底部用于存储数据。slot将一个特定slot映射到page的某个偏移量上。由此，可以将page内的tuple移动到想要的地方。slot从前往后添加，数据从后往前添加。

![image-20230310120047287](C:\Users\xiasui\Desktop\cmu14-445\image-20230310120047287.png)

如何知道tuple1保存在page1:识别tuple的方式是通过record id或tuple id实现，它是一个唯一标识符，标识一个tuple的逻辑地址。page id+offset值或slot标识。程序通过page id，再从slot中获取具体偏移。这使得page内部更新(整理碎片)的情况下，外部依旧能通过相同值访问。

在postgres 中1,2,3三个位置上有三个数据。删除2，再插入一个，此时插入的是位置4。因为postgres有vaccum。相当于垃圾回收器

在sql server中，当更新page时，如果有可用空间，就会把page变得紧凑，将数据写出到page上。1,2,3删除2添加4后，编程1,3,4。1,3没有空袭

oracle 与postgres 相同

 
