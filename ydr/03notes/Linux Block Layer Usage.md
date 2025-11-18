## 页面alloc和使用
### （1）alloc/free一个页面

```c
struct page *page = alloc_page(GFP_KERNEL); // 分配一页内存
__free_page(page); // 释放页面
```

但这个page无法直接用于修改page的内容。因为 `struct page` 本身并不包含页面的实际数据内容。`struct page` 只是描述页的元数据，如页的状态、引用计数、物理地址等。

### （2）页面访问

```c
void *page_ptr = kmap(page);
kunmap(page);
```

将该页面映射到虚拟内存空间，能够实现页面内存内容的访问

## bio结构

### （0）bio概述

bio结构体用于Linux中的块设备的读写操作，主要如下图所示：

![[Pasted image 20241207175348.png]]

### （1）申请bio结构体

```c
struct bio *bio_alloc(gfp_t gfp_mask, unsigned int nr_iovecs);
// eg
struct bio *bio = bio_alloc(GFP_KERNEL, secs_to_sync + packed_meta_pgs);
```
- `gfp_mask` 内存分配标志
- `nr_iovecs` 最大支持的I/O 向量的数量/页面数

`bio_alloc` 初始化一个bio结构体，限定其能够支持的最大页面数。

### （2）添加物理页面

```c
int bio_add_pc_page(struct request_queue *q, struct bio *bio, struct page *page, unsigned int len, unsigned int offset)
// eg
bio_add_pc_page(q, bio, page, rb->seg_size, 0)
```

- `bio_add_pc_page` 将页面通过尾插法添加到bvec数组中，并更新`bi_vcnt`。 

（3）获取物理页面
```c
// eg
struct page *page = bio_page(bio);
void *data = bio_data(bio);

// eg 完成一次bio所有数据的遍历 待研究
struct bvec_iter iter;
struct bio_vec bvec;
bio_for_each_segment(bvec, bio, iter) {
    struct page *page = bvec.bv_page;
    unsigned int offset = bvec.bv_offset;
    unsigned int len = bvec.bv_len;
    void *data = kmap(page) + offset; // 映射页面并获取数据指针
    // 使用数据...
    kunmap(page); // 解除映射
}

// eg 直接通过bio结构体结构访问某个特定页面
int index = 0;
struct bio_vec bvec = (bio->bi_io_vec)[index]; 
struct page *page = bvec.bv_page; 
void *data = kmap(page) + offset; 
// 使用数据... 
kunmap(page);
```
- `bio_page` 获取 `bi_iter` 当前指向的 `bio_vec` 中的 **页面结构体**（`struct page`）
- `bio_data` 获取 `bi_iter` 当前指向的虚拟地址（数据指针）

## Reference
- <a href="https://blog.csdn.net/geshifei/article/details/119959905"> linux block layer 子系统数据结构及初始化 | CSDN</a>
- <a href="https://aliez22.github.io/posts/11537/"> 【块设备】通用块层 struct bio 详解 </a>

