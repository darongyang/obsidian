### 页面alloc和使用
（1）alloc/free一个页面
```c
struct page *page = alloc_page(GFP_KERNEL); // 分配一页内存
__free_page(page); // 释放页面
```
但这个page无法直接用于修改page的内容。因为 `struct page` 本身并不包含页面的实际数据内容。`struct page` 只是描述页的元数据，如页的状态、引用计数、物理地址等。
（2）页面访问
```c
void *page_ptr = kmap(page);
kunmap(page);
```
将该页面映射到虚拟内存空间，能够实现页面内存内容的访问

### bio有关

bio结构体用于Linux中的块设备的读写操作，主要如下图所示：
![[Pasted image 20241207162949.png]]
（1）申请bio结构体
```c
struct bio *bio_alloc(gfp_t gfp_mask, unsigned int nr_iovecs);
// eg
struct bio *bio = bio_alloc(GFP_KERNEL, secs_to_sync + packed_meta_pgs);
```
- `gfp_mask` 内存分配标志
- `nr_iovecs` I/O 向量的数量