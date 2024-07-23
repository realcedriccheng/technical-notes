[https://blog.csdn.net/jinking01/article/details/106490467](https://blog.csdn.net/jinking01/article/details/106490467)
[https://blog.csdn.net/u010039418/article/details/115773253](https://blog.csdn.net/u010039418/article/details/115773253)

- 文件的地址空间用来管理文件映射到内存的页面，将文件系统中 file 的数据与内存 page cache 或者 swap cache 相关页面绑定到一起。这样就可以将盘上实际不连续的数据以页面为单位连续呈现出来。
- 表示地址空间的数据结构是 address_space 结构体，表示地址空间操作表的数据机构是 address_space_operations（a_ops）。
- 地址空间的属主（host）是 inode。对于块设备文件是其主 inode。
- 文件的地址空间以页面为单位组织成基数树。
- 地址空间的操作一般是对具体 page 的操作。包括 writepage、readpage 等。将内存中的某个 page 落盘或者将盘上某个 page 的数据读取到内存中。
   - 注意，i_fop 中那些操作都是针对内存的。只有在这里才是真正的 IO。
- 读取文件时
   - 以文件的f_mapping为参数，通过find_get_page查找page cache
   - 查找到page cache且数据是最新的，就通过copy_page_to_iter，将数据拷贝到用户空间
   - 没找到，就通过page_cache_alloc分配一个新页面，并将其加入page cache和LRU链表
   - 然后调用对应的readpage函数，从磁盘中读入文件数据
   - 最后还是通过copy_page_to_iter，将数据拷贝到用户空间
   - ![image.png](https://cdn.nlark.com/yuque/0/2024/png/22949753/1721700552847-a877056b-996f-4d4d-a63d-9574ff62a0d8.png#averageHue=%23ededed&clientId=u8844a054-2e92-4&from=paste&height=410&id=u9f982889&originHeight=451&originWidth=497&originalType=binary&ratio=1.100000023841858&rotation=0&showTitle=false&size=49674&status=done&style=none&taskId=ued043d3f-069c-496f-977b-c4d0db7a220&title=&width=451.8181720252867)
