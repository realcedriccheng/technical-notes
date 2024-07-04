# 转载！
参看：[https://www.cnblogs.com/liuchao719/p/annotation_of_F2FS_atomic_write.html](https://www.cnblogs.com/liuchao719/p/annotation_of_F2FS_atomic_write.html)<br />为防止丢失，现摘录如下。
<a name="UJs2B"></a>

# [F2FS atomic write 注解](https://www.cnblogs.com/liuchao719/p/annotation_of_F2FS_atomic_write.html)

最近看到了 FAST 2022 上的一篇论文 [exF2FS: Transaction Support in Log-Structured Filesystem](https://www.usenix.org/conference/fast22/presentation/oh)，在 F2FS 的基础上添加了事务支持。<br />An F2FS transaction [34] supports only the atomicity, neither isolation nor durability. The transaction in F2FS cannot span multiple files. Ironically, despite its barest minimum support for the transaction, F2FS is the only filesystem that successfully deploys its transaction support to the public. F2FS’s transaction support has a specific target application: SQLite. With atomic write of F2FS, SQLite can implement the transaction without the rollback journal file and can eliminate the excessive flush overhead [31, 64]<br />所以有了 Q1。在研究的途中发现 22 年 4 月，Google 提交了一笔 Change，改变了 atomic write 的实现方式 (3db1de0: f2fs: change the current atomic write way)，和论文描述的有所出入，新的方式实现更合理（见 commit message），所以本文分析的是新实现。<br />文章是边读代码边写的。代码是功能最详细的说明书，实现细节都可以通过读代码获得，所以此处多为代码流程之外的一些思考。代码版本为 v5.19。
<a name="d9fba37a"></a>

# Q1: atomic write 是为数据库，尤其是 SQLite 设计的，研究其 ACID 特性？

A1: 参考 [ACID - 维基百科，自由的百科全书](https://zh.wikipedia.org/zh-cn/ACID)。

- A: start atomic write 根据原 inode 大小创建了一个 cow inode，后续所有的写入都会写到 cow inode 里。在 commit 或 abort atomic write 之前，无法读取到 cow inode 的数据。writer commit 之前，reader 看到的是旧数据；之后看到的是新数据。回滚则依赖 abort。
- C: 这里只关注文件系统一致性。cow inode 被设置为 orphan inode，且只与内存 inode 关联。commit 之前断电，atomic write 的数据在 orphan inode 中不影响原 inode。commit之中断电，orphan inode 在开机后会被清除。
- I: atomic write 粒度是单个 inode。开启 atomic write 后，inode 会设置 ATOMIC flag，后续所有写入都会写到 cow inode 里。atomic write 不是为并发设置的，并发 start atomic write 之后，假设 inode 被一个进程 commit/abort，另一个进程并不会收到通知，并还认为其在 atomic write。
- D: commit 中有调用 fsync，保证数据回写完成。
  <a name="8a5d6965"></a>

# Q2: down_write i_gc_rwsem 以及 inode_lock 的意义？

<a name="034bfa5d"></a>

## Q2.1：i_gc_rwsem 的来历及用途？

i_gc_rwsem 原名 dio_rwsem（b2532c6: f2fs: rename dio_rwsem to i_gc_rwsem ），是为了防止并发时，DIO 写到已经被 GC 搬迁了的错误 block addr 中，或从错误 block addr 中读取数据（82e0a5a: f2fs: fix to avoid data update racing between GC and DIO）；buffer write 期间不会持有 i_gc_rwsem 锁。现在 i_gc_rwsem 的主要用途有：

- i_gc_rwsem READ 锁: 读者为 DIO read、DIO write（需要 DIO read）；写者为会搬移 data block 的 DATA GC。READ 锁好理解，DIO 读取 block 时，不希望 GC 搬移 block 从而从过期的地址读取数据。
- i_gc_rwsem WRITE 锁：读者为 DIO write；写者为会读取以及写入 data block 的 DATA GC 及其他修改 block 的 ioctl。WRITE 锁有些复杂，是两次改动分别引入的。
  1. 82e0a5a: f2fs: fix to avoid data update racing between GC and DIO。防止 DIO write 与 DATA GC 读写冲突。
  2. bb06664: f2fs: avoid race in between GC and block exchange。防止修改 block 的 ioctl 和 DATA GC 读冲突。
     <a name="b21fdcdb"></a>

## Q2.2：为什么 DIO 读写锁要分开？为什么读写锁都是 down_read？

用两个锁是因为防止 DIO read 内部对 inode lock，导致 ABBA 死锁 [[PATCH v3] f2fs: fix to avoid data update racing between GC and DIO](https://lore.kernel.org/all/20160708031916.GA94348@jaegeuk/T/)。最新代码已经从 __blockdev_direct_IO 切换到 iomap，我暂时还没有测试 read 路径是否还存在 inode_lock。<br />dio 读写都是 down_read 是因为 DIO 读、写都有可能有不同的进程，这些读写可以并发。但其实写因为 inode lock 不能并发，而且！当改为不公平 rwsem 之后，所有上述对 i_gc_rwsem WRITE 的 down_write 都可以在 DIO write 调用的 down_read 等待唤醒时插队，导致 DIO write 延迟（我认为可以优化为 down_write）。
<a name="50f4ed79"></a>

## Q2.3：为什么 DATA GC 读要用 i_gc_rwsem WRITE？而且还是 down_write？

WRITE 锁最初用于阻止 DIO write 和 DATA GC 并发，因为 DATA GC 需要搬移 block，dio write 会写到不正确的地方。在 i_gc_rwsem WRITE 锁的第二个引入 commit 里，因为 ioctl 会修改 blk addr，也要用 WRITE 锁；为了阻止 gc 读到不稳定的 blk addr，所以要用相同的 WRITE 锁控制。<br />至于为什么用 down_write…… 没想明白。（我认为各场景应该用 DIO write: down_read，DATA GC 搬移: down_write，DATA GC 读: down_read，ioctl: down_write）。
<a name="e7ae0670"></a>

## Q2.4：为什么 atomic write start 需要对 i_gc_rwsem WRITE 加写锁，start 又不涉及 blkaddr？

atomic write start 需要在 i_gc_rwsem WRITE 锁的保护下进行，因为我们希望回写之前的 dirty page 后再设置 ATOMIC 标记。回写之后一直到标记设置成功前，不能有 ditry page 产生，包括 GC。通过 i_gc_rwsem WRITE 锁可以避免这种情况。(c27753d: f2fs: flush dirty pages before starting atomic writes & 27319ba: f2fs: fix race in between GC and atomic open)
<a name="d22b4ca5"></a>

## Q2.5：为什么 atomic write commit 需要对 i_gc_rwsem WRITE 加写锁？

atomic write commit 也需要在 i_gc_rwsem WRITE 锁的保护下进行，因为会修改 block addr，类似于 i_gc_rwsem WRITE 锁的第二个引入 commit。
<a name="3e6550e1"></a>

## Q2.6：为什么 atomic write 的 start/commit/abort 操作都需要 lock inode？

inode_lock 会在写入 inode 期间持有，覆盖了 write_begin/write_end。start/commit/abort atomic write（以及其他 ioctl）持有 inode_lock 是为了不与正常写流程并行，这样 能保证操作的一直是 cow inode 的 page。
<a name="f1af63ee"></a>

## Q2.7：为什么本来要研究 atomic write，却研究了这么多 lock？

本来只是想研究一下为什么要 lock i_gc_rwsem。在研究 GC 时发现有 READ、WRITE 两把锁，研究途中又发现这两把锁还是读写信号量，而且用读锁还是写锁、以读者/写者访问的逻辑都很奇怪，所以就有了 Q2.1 - Q2.3。至于 inode lock 则是好奇如何控制数据精确地写入 cow inode page 里的。
<a name="53302519"></a>

# Q3: tmpfile 和 orphan inode 是怎么回事？abort 直接 put inode 后，inode 会何去何从？

cow_inode 不应该被用户访问到，所以 f2fs_get_tmpfile 生成临时 inode 时，没有把 inode 添加到原目录中，并且直接把 inode 挂到 orphan inode 链表中。<br />orphan inode 链表中的 inode 都是 nlink 为 0 （用户无法找到）但仍然有用户在使用的。在意外断电重启后，这些 inode 已经没有用了，遂删除。<br />当 nlink 为 0 时，iput 陆续会调用 super_operations 中的 drop_inode，evict_inode，destroy_inode，free_inode 等函数销毁 inode。
<a name="313e11ab"></a>

# Q4: atomic write 写流程分析？

- start：条件检查（权限，普通文件，非 DIO）。加锁、转换为非 inline、等待回写。创建 tmpfile，对 inode 加标记，结束。
- write：write_begin 时获取原 inode mapping 里的 page，并尝试获取 1.cow_inode 对应 index 的 blkaddr，2.原 inode 对应 index 的 blkaddr，3.在 cow_inode 里分配新的新 blkaddr，再读到 page 中。write_end 时同时更新 cow_inode 和原 inode 的 i_size。
- writeback：do_write 时拦截，获取 cow_inode 的 dnode info，将新 blkaddr 写入 cow_inode 中。原 inode 中的 block 保持正常访问。
- commit：条件检查（权限）。清理内存，加锁，用 cow inode 里的有效 block 替换原 inode 里的对应 block，解锁。sync file，abort atomic write。<br />abort：条件检查（权限）。加锁，清理标记，解锁。
  <a name="92630b05"></a>

# Q5: truncate/punch_hole 或者 append_write 怎么实现，怎么恢复？

atomic write 不处理 truncate/fallocate，这两个系统调用会直接作用于原 inode（所以不能在 atomic write 时 truncate/fallocate，不然如果 cow_inode 较大，多出来的部分无法转移到原 inode）。<br />append_write 时直接在 cow inode 里保留空间，并在 write_end 时为两个 inode 都记录新 i_size（这里存在不一致，未 atomic 但可以看到文件大小增长。最近 Daeho Jeong 已经发现了这个问题 [f2fs: correct i_size change for atomic writes](https://lore.kernel.org/linux-f2fs-devel/20220919160644.2219088-1-daeho43@gmail.com/)）。<br />恢复时，如果原地址是 NULL，就直接 truncate；如果原地址有效，就 replace_block。
<a name="5bc5f499"></a>

# Q6：为什么 atomic inode 不可以 inplace update，也不支持压缩？

opu 是因为要实际分配新的 blkaddr，并记录到 cow inode 中。要是 ipu 就会复用原来的 blkaddr，导致原 inode 可以访问到新的数据。<br />在场景上考虑，压缩是为了给一些不更新的文件节省空间用的，数据库文件更新频繁，没有压缩必要。
<a name="53d7a78e"></a>

# Q7：commit 完成之前断电，能恢复成调用 atomic write 之前的样子吗？

看了代码中没有重新提交 cow_inode 的 roll-forward 流程，也没有单独给 commit page 做标记然后恢复到 atomic write 之前的 roll-back 流程，所以应该是不行的。类比 fsync 过程，如果 fsync 完成之前断电，落盘也是修改的一部分，也不会恢复到 fsync 前
<a name="786ef30c"></a>

# Q8：revoke 之前如果 revoke list 里记录的 block 被 gc 搬走怎么办？

不存在，因为已经锁了 i_gc_rwsem
<a name="d4ad0835"></a>

# Q9：cow_inode 在 commit 前不 writeback 下吗？

目前只对原 inode writeback 的。我认为如果不针对 cow_inode writeback 会有问题：cow_inode 的 dirty page 为 NULL_ADDR 状态，commit 时会跳过，造成数据丢失；至少要 do_write_page 分配地址才可以。<br />那 gc 把 cow inode 数据搬走怎么办？只要数据落盘就 ok，commit 时是 valid blkaddr 状态就可以。