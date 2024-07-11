参考：https://github.com/RiweiPan/F2FS-NOTES/blob/master/Reading-and-Writing/%E5%86%99%E6%B5%81%E7%A8%8B.md
Yongmyung Lee 等： When F2FS Meets Address Remapping

ipu，in-place-update 就地更新：在原地更新数据。传统文件系统如 ext4 都采用。
opu，out-of-place-update 异地更新：将更新后的数据写在新的地址，修改映射到新地址。

1. 分配一个新的物理地址
2. 将数据写入新的物理地址
3. 将旧的物理地址无效掉，然后等GC回收
4. 更新逻辑地址和物理地址的映射关系

异地更新更适合闪存特性。
OPU 的缺点在于（1）产生无效块，造成 GC 开销；（2）更新元数据的开销；（3）数据碎片化