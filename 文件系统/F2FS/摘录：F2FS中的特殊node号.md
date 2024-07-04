# 转载！
参看：[https://www.cnblogs.com/liuchao719/p/some_special_node_id_in_F2FS.html](https://www.cnblogs.com/liuchao719/p/some_special_node_id_in_F2FS.html)<br />为防止丢失，现摘录如下。
<a name="aAFKe"></a>

# [F2FS 中一些特殊的 node id](https://www.cnblogs.com/liuchao719/p/some_special_node_id_in_F2FS.html)

在 mkfs 中定义了一些特殊的 node id，比如 node，meta，root 和 compress。<br />实际 F2FS 可分配的 node id 是从 4 开始的。

```c
set_sb(node_ino, 1);
set_sb(meta_ino, 2);
set_sb(root_ino, 3);
c.next_free_nid = 4;

#define F2FS_COMPRESS_INO(sbi)	(NM_I(sbi)->max_nid)
```

<a name="node-inode"></a>

# node inode

1 号 inode，data page 里保存了其他 node 的数据，顺序更新。根据 nid 可以从 node inode 的 mapping 里获取对应的 page，之后通过 f2fs_get_node_info 从 NAT 获取 nid 对应的 blkaddr，最后提交 io 获取 page。接口为 f2fs_get_node_page。<br />在 inode 创建时，通过 f2fs_new_inode_page 创建 node page。另外在保留 data block 时，如果现有 node 空间不足以存放 data block addr，则需要通过 ALLOC_NODE 模式调用 f2fs_get_dnode_of_data，在内部创建 node page。
<a name="meta-inode"></a>

# meta inode

2 号 inode，data page 里保存了 CP/NAT/SIT/SSA/POR 数据，随机更新。在获取 meta page 时要指定 meta 类型，如 META_CP、META_NAT 等，f2fs_ra_meta_pages 会根据 meta 类型把 start 加上对应类型的偏移量。<br />meta inode 是不加/解密的，所以还用于 GC 的时候临时做 block 的暂存地。读加密 page 出来，然后再分配新的地址，并写入；最后把地址更新到原 inode 里。相当于借了 meta mapping 的一个 page 和对应的 aops，blkaddr 还是原来 inode 的 blkaddr。参考

- 4375a33664de (f2fs crypto: add encryption support in read/write paths)。
- 6aa58d8 (f2fs: readahead encrypted block during GC)<br />另外 f2fs_wait_on_block_writeback 这个奇怪的函数也是为了这个创造出来的。
  <a name="516c71be"></a>

# 为什么 node id 是从 1 开始的？

因为 nid = 0 被视作未分配，或者无效。参考 __get_node_page。<br />另外在 truncate_dnode 里，nid == 0 即视作已经 truncate。