# sort

![image-20230322154035690](C:\Users\xiasui\Desktop\cmu14-445\lec10\image-20230322154035690.png)

排序算法:在磁盘中排序，无法加载全部到内存:外部排序

1：轮流读B个page到内存，在内存中进行排序。

2：归并两个page ，双指针，每当写满一个page，写回到磁盘，然后再写。

3：将两个page视作一个page去除，两个two page归并.....

![image-20230322162143288](C:\Users\xiasui\Desktop\cmu14-445\lec10\image-20230322162143288.png)



可以采用预取的方式来优化io

利用B+ tree提升排序操作的速度，如果我们想要排序的key与B+tree 上的key相同，就复用B+ tree

clustered B+ tree(聚簇索引)指page中tuple的物理位置与索引中定义顺序匹配(付出更多插入代价)

uncluster B+ tree (叶节点不保存完整tuple数据，只保存索引对应字段数据)用于排序非常糟糕，因为对于每次记录都要io

# aggregations

聚合操作(group by)

可以用sort或hash，一般hash更好

![image-20230322164902590](C:\Users\xiasui\Desktop\cmu14-445\lec10\image-20230322164902590.png)

先处理where中的过滤

去除不需要的sid，grade行

执行order by 进行排序

最后distinct ，由于之前已经order by了，所以直接去除







