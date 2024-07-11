# 文件系统中的扩展属性xattr

> 参考：《文件系统技术内幕》

# 什么是文件系统中的扩展属性 xattr

文件的基础属性包括 inode ID、创建时间和大小等，比较有限。扩展属性 xattr 是一种允许用户为文件添加自定义属性的方法。xattr 以键值对的方式储存在文件外部。

## 通过 API 使用 xattr

setfattr 设置属性，getfattr 获取属性。

# xattr 的实现

## NTFS

在 Windows 的 NTFS 文件系统中没有扩展属性的概念，而是有 ADS（Alternate Data Stream）的概念。

## Ext2

在 Linux 中，以 Ext2 为例。xattr 的内容存储在一个单独的逻辑块中，由描述头、扩展属性项和值组成。
![534520072.jpg](https://cdn.nlark.com/yuque/0/2024/jpeg/22949753/1720595918927-3071c430-0d77-4f6d-9d0a-74580ece67fc.jpeg#averageHue=%23adada8&from=url&id=utHTN&originHeight=1493&originWidth=1846&originalType=binary&ratio=1&rotation=0&showTitle=false&size=1443169&status=done&style=none&title=)
entry 存储了键和值的偏移量等信息。键向下生长、值向上生长。但是由于键和值的长度是可变的，因此 entry 和 value 的长度也是可变的，必须通过遍历 entry 才能找到键，通过 offs 才能找到 value。

## F2FS

为了搞清楚 F2FS 中 xattr 的实现方法，必须解答三个问题：xattr 怎样存储，怎样设置，怎样查询。

### 数据结构

f2fs 中也存在 xattr 的 header 和 entry。

```cpp
struct f2fs_xattr_header {
    __le32  h_magic;        /* magic number for identification */
    __le32  h_refcount;     /* reference count */
    __u32   h_reserved[4];  /* zero right now */
};

struct f2fs_xattr_entry {
    __u8    e_name_index;
    __u8    e_name_len;
    __le16  e_value_size;   /* size of attribute value */
    char    e_name[];      /* attribute name */
};
```

### 接口

xattr 是 linux 的文件系统中广泛支持的功能，VFS 通过一系列函数接口调用具体文件系统的实现。f2fs/xattr.c 中的f2fs_xattr_user_handler、f2fs_xattr_trusted_handler、f2fs_xattr_advise_handler 和f2fs_xattr_security_handler 是 f2fs 中针对 xattr 的接口。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/22949753/1720598373069-8146aff6-744c-4efd-a6fc-de73d2f71c50.png#averageHue=%23201f1f&clientId=u69f81bf4-140b-4&from=paste&height=664&id=u62b11fbe&originHeight=664&originWidth=589&originalType=binary&ratio=1.100000023841858&rotation=0&showTitle=false&size=25797&status=done&style=none&taskId=ue0e0d732-8575-42ab-9b89-1c9d2cc86e6&title=&width=589)
![](https://cdn.nlark.com/yuque/0/2024/jpeg/22949753/1720600596573-52b52bc4-9506-4ff8-a0f9-01f1af592050.jpeg)

