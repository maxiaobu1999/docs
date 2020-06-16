[TOC]

将缓存数据持久化到磁盘上，并使用 LRU 淘汰算法来维持缓存保持在一个稳定的大小。 

LinkedHashMap

默认实现了LRU 、访问时会将item移动到队尾、双向链表



## 日志文件

日志文件记录了 `DiskLruCache` 的各种活动，是纯文本文件，格式诸如：

```
libcore.io.DiskLruCache
1
100
2

CLEAN 3400330d1dfc7f3f7f4b8d4d803dfcf6 832 21054
DIRTY 335c4c6028171cfddfbaae1a9c313c52
CLEAN 335c4c6028171cfddfbaae1a9c313c52 3934 2342
REMOVE 335c4c6028171cfddfbaae1a9c313c52
DIRTY 1ab96a171faeeee38496d8b330771a7a
CLEAN 1ab96a171faeeee38496d8b330771a7a 1600 234
READ 335c4c6028171cfddfbaae1a9c313c52
READ 3400330d1dfc7f3f7f4b8d4d803dfcf6
```

第一行充当了 **"magic number"** 的角色，说明了此文件为 `DiskLruCache` 的日志，第二行则是类的版本号，第三行是客户端应用的版本号，第四行为每个缓存条目的文件数。

接下来的所有行就记录了之前 `DiskLruCache` 的一系列行为，在初始化的时候通过读取日志文件，就可以重建整个条目索引了。

日志中总共会出现四种状态：

- **DIRTY:** 记录了某一个条目开始被编辑，有可能是新增条目，也有可能是修改已有条目。接下来需要有 **CLEAN** 或 **REMOVE** 状态来平衡 **DIRTY** 状态，否则这就是无效的一条记录。
- **CLEAN:** 表明之前的某一 **DIRTY** 记录已经修改完毕（通常是执行 `Editor#commit()` 的结果），这一行结尾会有条目的各个文件的长度。
- **READ:** 记录了某一条的读取行为。记录读取行为的目的是为了在重构条目索引时将对应的节点移动到链表的尾部，以便于 LRU 算法的执行。
- **REMOVE:** 记录了某一条目的移除行为。





## Snapshot



Entry



Editor






