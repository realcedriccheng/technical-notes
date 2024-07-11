# 孤儿节点orphan node

orphan node 是无主的 node，orphan inode 是指一个文件不在任何目录中，不能被用户访问到。这样的文件仍然被某些进程占用，当没有进程使用时，该文件就被删除了。