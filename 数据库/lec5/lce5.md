# buffer池管理器

1:用malloc获取内存

2:将内存分为固定大小的chunk，称frame(对应slot，buffer池内存区域中的区块或chunk，可以将page放到里面)

3:此外还需要一层，想要某个内存中的特定page，通过这一层就能知道在哪个frame

追踪一些数据:

dirty flag:用于判断当我们从磁盘读取到这个page后，这个page是否被修改。(之后也要记录是谁改的)

pin count:引用计数，跟踪想要使用该page的当前线程数量或查询该page的数量。(用于防止被移除或交换回磁盘)

当想要读取一个不在内存中的数据，就要给page表加锁(写锁)，获取该page，然后更新所指向的page

page目录必须持久化:知道每个page的信息

page表不需要:只是个临时映射

# allocation polices

1:全局策略

试着做出的决策使得workload都受益:找到最近最少使用的page将其移除

2:局部策略

针对单个查询或单个事务进行

# buffer池实现

需要page表，将page id映射到frame

根据程序所分配的内存中的offset值，回得到所查找到的page位置

(可以尝试，多buffer池:)让一个buffer处理索引，另一个buffer处理表

buffer池通过将数据库对象的record id维护到一个列表中，可以通过每个id找到对应的对象条目。

pre-fetching(预取):当获取一个磁盘中的page时，同时返回该位置的指针(磁盘中的)给上层系统。当上层系统要进行遍历时，就可以通过指针快速获取下一个

scan sharing(扫描共享):利用彼此结果的查询，复用从磁盘中读到的数据。这是在buffer层面实现的，当允许多个查询附加到单个游标上时(将这些查询注册到游标数据结构管理的一个集合中)。像知道是否拿到一个新page，可以通知再等待这个page的那个线程。(将自己附加到此前的游标上)

buffer pool by pass():不想让一次循序扫描花精力再page table中找，并且不想污染缓存，思路:分配一小块内存给执行查询的线程。读page时，如果page不在buffer中，就必须从磁盘拿到page，但不会放到buffer中，而是放到本地内存。查询玩就抛弃。

所有磁盘操作都是通过os api实现，fopen,fread,rwrite。os也会维护一个page副本，buffer也有一个副本。所以数据库会通过direct I/O实现，让操作系统不进行任何缓存。

不过postgreSQL利用os page cache 。可以减少维护工作。

# buffer替换策略

不同数据库厂，替换策略很复杂。

这里只需要最终最晚的page

LRU:最近最少使用，Clock是其的一个近似算法，使得无需追踪单个page的时间戳，只需要追踪每个page的标志位(reference bit) 用于知道:自此你上次检查page后，这个page是否被访问。把page组织位环形buffer，像时钟一样。再某个时间段内，即一圈时钟。如果标致位未变化，即可从buffer pool移除该page

LRU-K:K是指需要对单个page对应缓存数据的访问进行计数的次数。(将最近使用过1次的判断标准拓展为最近使用K次，没有到达K次访问的数据不会被缓存)

# 处理dirty page

flag:用于判断当我们从磁盘读取到这个page后，这个page是否被修改。(之后也要记录是谁改的)

当要判断那个page该被移除时，最快方法是找到一个未被标记未dirty的page，直接删掉就行了，而不需重新刷会磁盘。但dirty page可能是常用page。需要在两者间抉择



工作：如果都被修改，就选择frame_id最小的进行移除

跟踪page是否可用，哪些page是dirty，哪些page是pinned
