
**写在前面**

一周的时间，匆匆赶出来了LightNVM解析的Version 1。这是很粗糙的版本，但了解LightNVM已经不成问题。Version 2可能要等比较久以后再更新了，可能几个月后，工作被push有点紧了。

一些临时跳过的地方，用TODO标记。

**LightNVM是什么**

LightNVM是一个基于OCSSD协议（1.2和2.0），在Linux上对OCSSD设备进行管理的FTL框架。在LightNVM框架下，可以开发用于OCSSD的任何具体FTL实现。最新的LightNVM中的FTL实现为pblk。但由于OCSSD协议并不完善，被后来的ZNS SSD所替代。从Linux 5.4版本以后，LightNVM已经被移除。但从科研角度，LightNVM + FEMU OCSSD的架构仍可以用于在SSD模拟方面。

**LightNVM处于IO栈的位置：**
以下两个图搬运自 <a href="https://blog.xiocs.com/archives/36/"> LightNVM 自顶向下的 IO 通路简析 </a>，鸣谢。
![[Pasted image 20241124154148.png]]

![[Pasted image 20241206165325.png]]

**LightNVM的主要模块：**
- pblk-rl.c rate limit 速率控制器
- pblk-read.c 读逻辑
- pblk-write.c 落盘写逻辑
- pblk-map.c l2p映射表
- pblk-cache.c 环形缓冲区写入
- pblk-rb.c 环形缓冲区管理
- pblk-core.c/pblk-init.c 其他一些初始化和核心的函数
- pblk-recovery.c 卸载后的恢复
- pblk-sysfs.c 调试层

<hr>

## 一、设备信息

### （1）设备容量

- `pblk_set_provision` 计算设备中用户可用的空间大小
- `pblk_capacity` 返回用户可见的设备大小（字节数）

### （2）几种地址结构
地址结构体
```c
struct ppa_addr {
	/* Generic structure for all addresses */
	union {
		/* generic device format */
		struct {
			u64 ch		: NVM_GEN_CH_BITS;
			u64 lun		: NVM_GEN_LUN_BITS;
			u64 blk		: NVM_GEN_BLK_BITS;
			u64 reserved	: NVM_GEN_RESERVED;
		} a;
		/* 1.2 device format */
		struct {
			u64 ch		: NVM_GEN_CH_BITS;
			u64 lun		: NVM_GEN_LUN_BITS;
			u64 blk		: NVM_GEN_BLK_BITS;
			u64 pg		: NVM_12_PG_BITS;
			u64 pl		: NVM_12_PL_BITS;
			u64 sec		: NVM_12_SEC_BITS;
			u64 reserved	: NVM_12_RESERVED;
		} g;
		/* 2.0 device format */
		struct {
			u64 grp		: NVM_GEN_CH_BITS;
			u64 pu		: NVM_GEN_LUN_BITS;
			u64 chk		: NVM_GEN_BLK_BITS;
			u64 sec		: NVM_20_SEC_BITS;
			u64 reserved	: NVM_20_RESERVED;
		} m;
		/* Cacheline format*/
		struct {
			u64 line	: 63;
			u64 is_cached	: 1;
		} c;
		u64 ppa;
	};
};
```
- 设备地址
从上述结构体中可以得出，OCSSD V1.2的设备地址结构 ：`| Channel | Die | Block | Plane | Page | Sector |`
- cacheline地址
表示在ring buffer中的索引。
- line上地址
将上述地址固定blk字段，得到的是一个物理页面在line上的地址。

### （3）一些设备字段 

**1.min_write_pgs和min_write_pgs_data**
- `min_write_pgs` 每次最少写的page数，ring buffer达到这个数目再向设备发起落盘写请求。这个数目通常为一个chip的planes数
- `min_write_pgs_data` 每次最少写的数据page数。有的时候设备不支持oob，需要额外的空间来存oob，又叫做pack meta来支持oob。如果支持的设备则`min_write_pgs`等`于min_write_pgs_data`。

### （4）OCSSD结构计算

```c
/* 1.2 device format */
struct {
	u64 ch		: NVM_GEN_CH_BITS;
	u64 lun		: NVM_GEN_LUN_BITS;
	u64 blk		: NVM_GEN_BLK_BITS;
	u64 pg		: NVM_12_PG_BITS;
	u64 pl		: NVM_12_PL_BITS;
	u64 sec		: NVM_12_SEC_BITS;
	u64 reserved	: NVM_12_RESERVED;
} g;
```

- SSD总容量：$cap$（可配置`ssd_size`）

以下三个和SSD内部并行有关：
- 通道数：$chs$（可配置`num_channels`）
- 每个通道的芯片数：$cps$（可配置`num_chips_per_channel`）
- 每个芯片的平面数：$pls$（可配置`num_planes_per_chip`）

（注意到这里的gap：每个芯片的块数）

以下两个和读写擦除粒度等有关：
- 每个块的页数：$pgs$（可配置`num_pages_per_block`）
- 页面大小：$pgsz$（可配置`hw_sector_size`和`num_sectors_per_page`）

基于上述，可以计算出上述缺失的gap：
- 块大小：$pgs\times pgsz$
- 总页数：$\frac{cap}{pgsz}$
- 总块数：$\frac{cap}{pgs\times pgsz}$
- 总芯片数：$chs\times cps\times pls$
- 每个芯片的块数：$\frac{cap}{pgs\times pgsz \times chs\times cps\times pls}$

那么可以很容易计算出：
- line的页数：$\frac{cap}{pgsz} \div \frac{cap}{pgs\times pgsz \times chs\times cps\times pls}=pgs \times chs\times cps\times pls$
- line的块数: $chs \times cps \times pls$
- line的大小：$\frac{cap}{pgsz} \div \frac{cap}{pgs\times pgsz \times chs\times cps\times pls} \times pgsz = pgs\times pgsz \times chs\times cps\times pls$
- line数等等

一个栗子：
```
> 16GB SSD 
> 4KB PAGE SIZE  --> 4,194,304 (4M个PAGE)
> 每个BLK 512个PAGE --> 总共8K个BLK
> 2 CHANNEL 每个CHANNEL 4个CHIP --> 8KBLK / (2 * 4)，也就是确定CHANNEL和CHIP后的BLK字段有1024
> 每个CHIP一个PLANE，也就是一个PLANE有1K个BLK  -->
综上:
| BLK0 (512PAGE) | BLK0 (512PAGE) | ... | BLK0 (512PAGE) | 
...
| BLK1024 (512PAGE) | BLK1024 (512PAGE) | ... | BLK1024 (512PAGE) | 
故不同channel和chip上的相同BLK ID的PAGE有512 * 2 * 4=4096个
一共1024个line(ch_nums * lun_nums * pl_nums * blk_nums)
> line布局（4096个PAGE）：smeta（1个），data（4085个），emeta（10个）
> 一个line多大？4096 * 512 * 4KB = 16MB，实际是能存4085个
```

### （5）lun/chip的互斥访问

**SSD的lun结构** lun又成为chip/die，是一个存储芯片。一个存储芯片同一个时刻只能处理一个写入命令。即对同一个lun执行多个写操作，该lun上的其他写操作会对应被阻塞。

**pblk的lun维护** pblk通过`pblk_lun`对SSD的lun结构进行模拟。维护：1.一个芯片的起始物理地址，2.一个用于互斥访问lun的信号量，如下：
```c
struct pblk_lun {
	struct ppa_addr bppa;
	struct semaphore wr_sem;
};
```

**有关函数**：
- `__pblk_down_chunk` 将对应编号的chip/lun上锁  模拟实现互斥访问
	- 传入要访问的luns的下标pos
	- 然后将其上锁，并设置30000ms的超时时间
- `pblk_down_chunk` 给ppa对应的chip/lun进行上锁
	- 获取ppa对应的chip/lun
	- `__pblk_down_chunk` 将对应编号的chip/lun上锁
- `pblk_down_rq` 给请求对应的chip/lun进行上锁 返回上锁位图
	- 获取ppa对应的chip/lun
	- 将对应的lun_bitmap置1
	- 将对应的chip/lun上锁
- `pblk_up_chunk` 将ppa对应的chip/lun解锁
- `pblk_up_rq` 将落盘写请求保存的lun_bitmap对应的chip/lun解锁
在提交任何落盘写请求（包括emeta和数据）时，都会将设备物理地址对应的芯片上锁。当落盘写请求完成以后，end_io时候会将对应的芯片解锁。


TODO Ingore Me

pblk的内核模块注册
- pblk_init: pblk_module_init
- pblk_init: pblk_module_exit
pblk的初始化和退出
- TODO？

<hr>

## 二、Ring Buffer (Write Cache)

### （1）基本介绍

ring buffer用于缓冲用户写入和GC写入的数据。ring buffer的用户写入和GC写入的配额，由rate limit控制。ring buffer还会维护数据持久化、缓存替换等指针信息。

### （2）条目信息 与 维护指针

buffer中的条目后续会进行落盘写入/持久化，以及发生缓存替换将条目逐出缓冲区。因此ring buffer对其条目维护了如下的属性信息：
- **是否进行了落盘写入/持久化**。ring buffer中的条目，每达到`min_pages`的页数，就会发起一次落盘写入操作。因此ring buffer内的条目可能有的已经落盘写入了，有的还没落盘写入，需要维护有关指针。
- **是否将其从缓存中淘汰**。缓存会发生条目的驱逐，被驱逐条目的位置就可以用来存其他新条目。因此ring buffer内存在有效和无效条目，需要维护有关指针。

ring buffer中有关的指针，包括`sync`、`mem`、`subm`和`l2p_update`。注：以下指针的更新均通过`smp_store_release`函数进行。
- `mem`：写入指针（缓存准入有关）。表示当前写入到ring buffer的哪个位置，在用户数据的缓存准入时在`pblk_rb_may_write_user`和垃圾回收的缓存准入时在`pblk_rb_may_write_gc`中会更新`mem`的新位置。
- `subm`：提交同步指针（持久化有关）。发起写请求时在`pblk_submit_write`中会更新`subm`的新位置
- `sync`：完成同步指针（持久化有关）。写盘成功了在`pblk_end_w_bio`中会更新`sync`的新位置。
- `l2p_update`：驱逐指针（缓存替换有关）。从该指针往后的才是有效条目。在逐出缓存时在`__pblk_rb_update_l2p`会更新`l2p_update`的新位置。（参考缓存的驱逐/替换小节）。


### （3）缓存的驱逐/替换

ring buffer中的条目，其l2p表中维护的是cacheline，而非设备物理地址。通过cacheline能够快速索引到ring buffer中对应的条目。l2p当其从缓存中替换的时候，l2p表中的cacheline会被替换为设备物理地址，cacheline供给其他的lba使用。
和缓存替换有关的函数是：
- `pblk_rb_update_l2p` 对缓存的可用空间(mem sync)，进行缓存替换，腾出本次写入的空间
	- 如果满足写入需求直接返回
	- 计算所需要驱逐的条目数
	- `__pblk_rb_update_l2p` 从缓冲区中逐出对应数量的条目
		- 将对应的l2p表从cacheline置为设备地址
		- 更新`l2p_update`

### （4）缓存的数据拷贝

- `pblk_rb_read_to_bio` 通过pos从ring buffer 读取数据并将其添加到传入的 `bio` 中，以增加引用/共享方式实现
	- 获取ring buffer的内存数据
	- 以共享的方式add到写bio中
	- 更新ring buffer entry 的 flags
- `pblk_rb_copy_to_bio` 通过地址lba和cacheline查询，将缓存中的条目数据复制到指定bio中，内存拷贝的方式实现，lba过期后将不拷贝
	- 检查lba是否发生过期
	- 以数据拷贝的方式写bio

### （5）缓存的flush操作

和普通的写入只在以下几点不同：1.缓存准入判断时增加flush指针的维护；2.缓存准入时，添加条目的flush flag位，立即唤醒写线程；3.只有落盘结束了才能通知bio_endio（非sync的话只用写到缓存即可bio_endio）。

有关字段：
- `flush_point`，flush操作要求至少flush到的位置。

有关函数：
- `pblk_write_to_cache`
	- `pblk_rb_may_write_flush`
		- `pblk_rb_flush_point_set` 增加flush指针的维护。将bio push到对应rb entry的bios list下。
	- 设置entry的flush flag位，立即唤醒写线程
- `pblk_end_w_bio` 落盘结束。将bios list的bio pop出来进行bio_endio。取消entry的flush flag位。


<hr>

## 三、Rate Limit

### （1）基本介绍

rate limit和ring buffer、GC逻辑紧密相关，是ring buffer的管理器，也是GC的启动器。rate limit会根据整个盘的剩余空闲块数，分配用户写入和GC写入分别能够占用多少ring buffer（user IO窗口、gc IO窗口），同时

两种IO窗口相互调整。无需GC的时候，gc IO窗口为0，全部都是user IO窗口；当需要GC的时候，分配部分gc IO窗口，user IO窗口减少。

### （2）重要字段

rate limit中几个重要字段：
- `rl->rb_user_cnt` User I/O buffer counter，记录了当前ring buffer中用户缓存条目的数量。
	- **修改：** 对用户数据进行缓存准入时候增加，从缓存驱逐的时候减少。
	- **使用：** 用来检查是否超过下面所述的`rl->rb_user_max`，以此来决定用户数据还能否插入缓冲区。
- `rl->rb_gc_cnt` GC I/O buffer counter，和上面类似，记录了当前ring buffer中GC缓存条目数。
	- **修改：** 对GC数据进行缓存准入时候增加，从缓存驱逐的时候减少。
	- **使用：** 用来检查是否超过下面所述的`rl->rb_gc_max`，以此来决定GC数据还能否插入缓冲区。
- `rl->rb_space`， Space limit in case of reaching capacity，有两种取值，负数和0；初始值为-1。负数表示无限制，0表示进行限制。（参考下面，还是不太能捋清楚这个结构体含义。没下一个line，然后置零，等用户数据写入？）
	- **修改：** （1）`pblk_line_replace_data`和`pblk_line_get_first_data`当下一个line无法准备时（即`if (!l_mg->data_next)`），会调用`pblk_set_space_limit`，将其置为0；（2）有用户数据写入缓存时，如果其为0，则将其进行sub操作。
	- **使用：**`pblk_line_close_meta`中会调用`pblk_rl_is_limit`判断是否为0选择是否sync meta。
- `rl->rb_user_max`，Max buffer entries available for user I/O，由rate limit根据整个盘剩余的`free_blocks`决定的用户可以使用的ring buffer大小。
- `rl->rb_gc_max`，Max buffer entries available for GC I/O，由rate limit根据整个盘剩余的`free_blocks`决定的GC可以使用的ring buffer大小。

### （3）窗口调整和GC启动

- `__pblk_rl_update_rates` 控制GC和user write的窗口。==TODO：代码注释写完了，找时间更新上来==

**执行时机**：
（1）pblk初始化的时候；
```
pblk_init -> pblk_gc_should_kick -> pblk_rl_update_rates -> __pblk_rl_update_rates
```
（2）line被put的时候/GC完成的时候；
```
__pblk_line_put -> pblk_rl_free_lines_dec -> __pblk_rl_update_rates
```
（3）line被申请使用的时候；
```
pblk_line_replace_data -> pblk_rl_free_lines_inc -> __pblk_rl_update_rates
```

### （4）ring buffer准入

- `pblk_rl_user_may_insert` rate limit检查用户缓冲区使用情况`rb_user_cnt`，判断用户写入是否准入ring buffer。
- `pblk_rl_user_in` 允许用户写入准入ring buffer后，更新用户缓冲区使用情况`rb_user_cnt`
- `pblk_rl_gc_may_insert` 同上，判断`rb_gc_cnt`决定是否准入ring buffer
- `pblk_rl_gc_in` 同上，更新`rb_gc_cnt`


<hr>


## 四、L2P映射表
- l2p表大小：`pblk_trans_map_size`
- l2p表分配：`pblk_l2p_init`


### （1）l2p表的更新

- `pblk_update_map` 普通的l2p表更新
	- 获取l2p表的表项地址，==将该旧地址失效==
	- 更新l2p表项
- `pblk_update_map_cache` 将l2p表表项更新为user cahcheline
- `pblk_update_map_gc` 将l2p表表项更新为gc cacheline，和user不同之处在于会再次判断gc页面还是否有效
- `pblk_update_map_dev` 将l2p表对应的表项从cacheline替换为设备地址
	- 检查当前的l2p表的地址是否为最新的cacheline。**==如果是旧的，进行页面失效==**
	- 更新l2p表对应表项的cacheline为设备地址
### （2）l2p表的查询

- `pblk_trans_map_get` 直接返回l2p表的对应表项
- `pblk_lookup_l2p_seq` 从blba开始连续查询nr_secs个lba的ppa。返回类型连续的截段：要么都from cache，要么都不from cache（为读设计）。
- `pblk_lookup_l2p_rand` 离散查询l2p表，返回lba_list对应的ppa_list
TODO：完善这个小节
TODO：思考为什么map gc时候没有失效操作


<hr>


## 五、line结构

### （1）定义与结构

#### 1. line的定义

 一个line结构为相同`Block`字段（由 每个芯片上的块数决定）的所有页面构成的集合。line是一种内部并行的条带化结构，一个line上的pages会依次出现在不同并行单元上，不断往复。
 
 每次写入时，按line的条带化结构，顺序分配和写入。擦除时，以line为单位进行擦除。

#### 2. line的结构

line的布局结构大致为：`| smeta | data | emeta |`

与line结构有关的一些字段：
- `geo->all_luns`/`lm->blk_per_line`: line能够支持的并行单元数/块数
- `lm->sec_per_line`: 每个line上拥有的总页数（包括smeta区、data区和emeta区）
- `line->sec_in_line`: 每个line上可以使用的有效页数（只包含data区，同时去除坏块）

#### 3. line的元数据管理

emeta的磁盘布局
- 第一段/First sector：`| struct line_emeta | bb_bitmap　| struct wa_counters |`
- 第二段/Mid sectors：`L2P portion`
- 第三段/Last sectors：`vsc` (for all lines)（嗯？TODO为啥要放全部lines的vsc）

有关结构体
- `struct line_header` line的标志符，emeta和smeta中均存有该标识符
- `struct line_emeta` （相对动态的字段？只有line close时才确定）
- `struct line_smeta` （相对固定的字段？line open的时候就可以确定和写入？）
- `struct pblk_line_meta` 公用的一些line属性。规定line的smeta的大小、emeta的大小（其中emeta包括多段，分别记录多段的大小），line上有多少个块和页面，各种位图的长度，和line进行GC的阈值等
- `struct pblk_emeta` pblk的line emeta的写入/同步维护结构体。维护line emeta的内存数据buf、写入指针mem、同步指针sync和总页面数nr_entries。
- `struct pblk_smeta` 与上述类似，pblk的line smeta的维护结构体，由于smeta固定为一页，且line open时就持久化写入，只需要维护line smeta的内存数据buf即可。
- `struct pblk_line_mgmt` 对所有的line进行管理的结构体。
	- `struct list_head emeta_list` 需要进行同步的emeta列表
	- `struct pblk_emeta *eline_meta[]` 最多同时支持`PBLK_DATA_LINES`个line，并维护这些line的所有emeta buf。
	- `struct pblk_smeta *sline_meta[]` 同上。
	- `unsigned long meta_bitmap` 记录当前上述`eline_meta`和`sline_meta`数组的使用情况。

有关函数：
- `pblk_line_mg_init` 根据lm中维护的emeta的大小，在l_mg中分配对应的emeta buf给所有line进行使用
```c
static int pblk_line_mg_init(struct pblk *pblk){
	...
    for (i = 0; i < PBLK_DATA_LINES; i++) {
        struct pblk_emeta *emeta;
        emeta = kmalloc(sizeof(struct pblk_emeta), GFP_KERNEL);
		...
        emeta->buf = kvmalloc(lm->emeta_len[0], GFP_KERNEL);
		...
        emeta->nr_entries = lm->emeta_sec[0];
        l_mg->eline_meta[i] = emeta;
    }
    ...
}
```
- `pblk_line_setup_metadata` 从l_mg的meta buf池中找一个可用的meta buf给当前line
	- 从`l_mg->meta_bitmap`找到可用的meta buf index
	- 将对应的meta buf挂到line下
	- 重置该meta buf的维护结构体
- `pblk_line_close_meta` 关闭一个line所对meta进行的同步操作
	- emeta准备
	- `list_add_tail` 加入到meta待同步列表l_mg->emeta_list中
	- `pblk_line_should_sync_meta` 根据rate limit的rb_space决定是否同步元数据
- `pblk_should_submit_meta_io` 判断现在是否要进行元数据emeta同步
	- 检查emeta提交列表是否为空
	-  获取需要进行meta写入的line，判断是否需要元数据同步
	- `pblk_valid_meta_ppa`，进一步检查meta io和data io的冲突情况（==TODO：没看懂这部分啊啊啊==）
- `pblk_submit_meta_io` 提交meta_line对应的emeta持久化
	- 申请一个设备write请求，指定写入大小和写入数据
	- 为该请求分配页面，生成设备地址
	- chip上锁，并提交emeta的write I/O请求
	- 更新emeta的同步/维护指针

### （2）类型和状态

line的状态变化为：new -> (free -> open -> closed -> gc) -> free -> open -> ...
- new
在pblk初次挂载，进行line的初始化，即`pblk_lines_init`时，会将其状态都置为new（TODO：重挂载也是所有的line都置为new吗）。
- open
一个line进入预备队列`l_mg->data_next`的时候，在`pblk_line_get`会将其状态从free置为open。（注：如果line的状态为new，会将其先变为free，在进一步变为open）
- closed
当一个line的emeta全部写入完毕时，会将一个line关闭，在`pblk_line_close` 会将其状态从open变为closed。
- gc
当将一个full line直接回收，或者对victim line从gc队列摘除进行gc迁移时，在`pblk_gc_run` 会将其状态从closed变为gc。
- free
当对一个line完成了垃圾回收后（即，将有效页全部迁移），会最终释放这个line的引用，进入line put的逻辑，在 `__pblk_line_put` 中会其状态从gc变为free。

### （3）页面分配和line的替换

#### 1. 页面分配

pblk按照line的逻辑结构管理物理页面。物理页面分配时，也会按照line上的顺序依次分配物理页，这能够达到最大的并行。
line上与之有关的字段包括：
- `cur_sec` Sector map pointer，分配指针，记录当前line上分配到哪一个页面了
- `map_bitmap` Bitmap for mapped sectors in line，line上的
- `left_msecs` Sectors left for mapping，
有关函数如下：
- `pblk_map_page_data` 按line申请物理页 并为lba分配ppa
	- 如果当前的line满了 换下一个line
		- `pblk_line_replace_data` 申请新的line写入，所进行的必要准备工作
		- `pblk_line_close_meta` 
	- 按照line结构顺序分配设备物理地址
		- `pblk_alloc_page` 在line的`map_bitmap`上找到下一个为0的地方开始分配（起始地址），设置`map_bitmap`，更新`cur_sec`指针，并返回起始地址
		- `addr_to_gen_ppa` 由line上地址和line id生成设备地址
	- 填写write context的ppa字段，交付ppa给ring buffer；填写落盘写请求的oob字段和ppa_list字段，构造完整写请求。
	- 对lun上锁，一次只能一个请求访问
		- `pblk_down_rq` 给请求对应的chip/lun进行上锁，模拟实现互斥访问，返回上锁位图`lun_bitmap`

#### 2. line的关闭

line关闭时会写入emeta，emeta元数据写入部分见：line的元数据管理
- `pblk_line_is_full` 通过`line->left_msecs`判断一个line是否写满了
- `pblk_end_io_write_meta` 在`pblk_submit_meta_io`时候指定的回调函数。完成line的全部emeta写入后，关闭这个line。
	- 将chip解锁，并更新emeta的同步指针sync
	- `pblk_line_close_ws`/`pblk_line_close` 全部emeta sync完成后，提交的工作线程。用于关闭一个line。
		- 将line的状态从open置为closed
		- 置空line的有关字段，如位图、meta等
		- 将line上的所有块的状态置为关闭


#### 3.下个line的准备

在上述页面分配的过程中，如果当前的line分配满了，会切换到一个新的line进行地址分配。
- `pblk_line_replace_data` 申请新的line写入，所进行的必要准备工作
	- 获取当前line，获取新的line，更新当前line
	- `pblk_line_setup_metadata` 给当前line分配一个可用的meta buf（从l_mg的meta buf池中）
	- `pblk_line_erase` 新的line的块如果没完全擦除，则进行擦除，等待新的line上所有的块都被擦除
	- `pblk_line_alloc_bitmaps` 为line分配map_bitmap和invalid_bitmap
	- `pblk_line_init_metadata` 初始化line的元数据，smeta和emeta，并判断和标记坏line。（TODO：lun_bitmap有什么用？）
	- `pblk_line_init_bb` 对line上的坏块进行标记管理，初始化line与之有关的字段
	- `pblk_rl_free_lines_dec` 减少总空闲块，并进行速率调整
	- `pblk_line_get` 准备好下一个line
		- 从line的free_list中取一个line出来
		- 判断坏line
		- `pblk_line_prepare` 完成line未擦除块的初始化 将line的状态置为open
			- 处理line，确保状态都是free，且均完成未擦除块的统计
			- 在line的预备时，将line的状态置为open，并增加line的引用
- `pblk_line_get_first_data` 变体版的`pblk_line_replace_data`，区别在于准备下一个line的时机。

left_eblks

几种bitmap？

- `map_bitmap` Bitmap for mapped sectors in line，记录已经被映射过lba的sector，1表示被映射过了。
	- **标记**：（1）`pblk_dealloc_page`清空，写失败时候将alloc的sector恢复；（2）`__pblk_alloc_page`设置，标记某个sector已经被分配；（3）`pblk_line_init_bb`设置，将smeta设置为已经被映射；（4）`pblk_recov_l2p_from_emeta`设置，从ssd恢复原来line的bitmap；（5）`pblk_map_remaining`设置，用于写异常的处理使用

### （4）有效页维护与invalidate操作

#### 1. 有效页维护

- `invalid_bitmap` 失效位图，每个line通过失效位图来记录line上被失效的页。位图中为1表示就line上对应的页面被失效。
-  `vsc`，valid sector count，有效页面数，表示这个line中有效页面的数量。如果`vsc`为0，代表line上都是无效的页面。如果vsc为`sec_in_line`，则line上全是有效页面。
- `nr_valid_lbas`，有效逻辑地址数，用来记录当前的line上有多少有效的lba。

#### 2. invalidate操作

主要涉及函数
- 获取ppa对应的line，及其line上地址`paddr`。（函数`pblk_map_invalidate`中）
- 进一步标记失效位图、减少有效页面数（函数`__pblk_map_invalidate`中）
- 将状态为关闭的line放入到指定的回收队列中


### （5）坏块标记与块的擦除

==TODO：line的（不是请求的）lun_bitmap到底有什么用==
**有关字段：**
- `blk_bitmap` Bitmap for valid/invalid blocks，记录坏块，1表示坏块。
	- **标记**：（1）在`pblk_mark_bb`函数中会标记坏块；（2）当`__pblk_end_io_erase`擦除失败的时候会将某个块标记为坏块
- `erase_bitmap` Bitmap for erased blocks，记录已经被擦除过的sector，1表示被擦除过。
	- **标记**：（1）初始化，`pblk_line_prepare`时会将`erase_bitmap`设置为`blk_bitmap`；（2）`pblk_line_erase`逐块擦除一个line的时候，设置`erase_bitmap`并`pblk_blk_erase_sync`；（3）是边写边擦的时候在`pblk_map_erase_rq`时，设置`erase_bitmap`并`pblk_blk_erase_sync`
- `left_eblks` Blocks left for erasing，line上剩余还未提交擦除的块数。在提交擦除的时候进行减一。
- `left_seblks` Blocks left for sync erasing，line上剩余还未完成擦除的块数。在完成擦除的时候才进行减一。

有关函数：
- `pblk_line_init_bb` 对line上的坏块进行标记管理
	- TODO 根据`blk_bitmap`处理/计算坏块标记?
	- 将smeta和emeta标记为坏块
	- 持久化存储smeta
	- 初始化与之有关的line的其他字段

### （6）引用与释放

**line的引用**。line要被占用，以免被GC错误释放掉（TODO）。ring buffer中的条目会缓存line中对应的页面。以下是line中有关的两个字段：
- `line->sec_to_update`，Outstanding L2P updates to ppa，line中位于ring buffer的页面数（也就是l2p地址为cacheline但是实际物理地址ppa在该line上的页面数）。
	- **增加**：初始值为0。每当为lba分配了一个物理页面device addr时增加（注：数据一定会预先写入ring buffer）
	- **减少**：每当进行一次ring buffer替换，进行条目驱逐和l2p表替换，在调用`__pblk_rb_update_l2p`时减少。
	- **作用**：垃圾回收gc选择victim line的时候，只选择`sec_to_update`为0的line。
- `line->ref`，Write buffer L2P references，是line的引用计数。ring buffer会占用line，增加line引用。读取line也会占用line，增加line引用。（`line->sec_to_update`类似于其拆解出来的子集）。
	- **增加**：初始值为1。（1）同上。分配物理页增加。（2）每次读的时候，遇到非cache读，也就是设备读，将对应line的line->ref加一（读是随机的）。（3）对line做gc，发起一次有效页的读/写时，line->ref加一。
	- **减少**：（1）同上，缓存驱逐减少。（2）设备读取结束后，end_io中将对应line的line->ref减一。（3）完成一次有效页的读-写时，释放该line->ref减一。（4）最终完成整个line的gc时，释放最后一次引用（对于full line是直接回收时，对于victim line是完成其全部有效页读-写的提交时）。
	- **作用**：如果line的对应line->ref减为0，则对line进行`__pblk_line_put`操作。

只要不被gc的line，line->ref一定大于等于1，也就是正常来说写过且没被gc的line的line->ref都是大于等于1的。完成GC的line会解除最后的`line->ref`占用，该引用变为0后，就会执行下面的line put逻辑：
-  `pblk_line_put_wq` / `pblk_line_put_ws`/ `pblk_line_put` / `__pblk_line_put`  将完成垃圾回收的line放入空闲链表
	- 将line的状态从gc置为free
	- 置空line的相关结构 vsc、map/invlid bitmap、s/emeta
	- 将其放入空闲链表 更新空闲line数量


<hr>


## 六、GC逻辑

### （1）垃圾回收队列

有关函数：
- `pblk_line_gc_list` 将某个line放入到垃圾回收队列的逻辑：
	- 根据line的有效页数`vsc`，将line的`gc_group`设置为不同的组别（例如，vsc等于0放入gc_full_list，vsc较大放入gc_low_list等）
	- 返回对应的垃圾回收队列的头节点
	- 然后将这个line插入到头节点对应链表中

**调整时机** 何时执行将line放入垃圾回收队列？
- 当line写满时，将一个line关闭的时候
```c
pblk_submit_meta_io -> pblk_end_io_write_meta -> pblk_line_close_ws -> pblk_line_close -> pblk_line_gc_list
```
- 对一个line上的页面进行失效的时候，如果一个line已经关闭了会调整其垃圾回收队列
```c
pblk_map_invalidate -> __pblk_map_invalidate -> pblk_line_gc_list
```

特殊地，PBLK_LINEGC_NONE是错误队列？

### （2）垃圾回收逻辑

![[Pasted image 20241206142044.png]]

**概述**：pblk在进行垃圾回收的时候，gc主线程会从上述的GC队列中取出victim line进行垃圾回收。gc读线程构造gc_rq读取line上全部有效页。gc写线程将全部gc_rq都写入ring buffer。

**线程模型**：每个线程执行完都会进行sleep，等待下一次唤醒。
- set_current_state
设置当前线程的状态为可中断等待状态
- io_schedule
让线程进入睡眠状态，等待某些I/O事件的发生
当某些条件满足时，线程会被唤醒并继续执行

#### 0. GC启动/停止时机

gc的启动逻辑为`pblk_gc_should_start`和`pblk_gc_start`，这两个函数均在`__pblk_rl_update_rates`中调用。 也就是GC的启动和终止时机，就是rate limit调整窗口的时机：（1）line init；（2）line prepare；（3）line put。
- `pblk_gc_should_start` / `pblk_gc_start` GC开始函数，通过将gc->gc_active置为1
- `pblk_gc_should_stop` GC结束函数，通过将gc->gc_active置为0

#### 1. GC主线程

- `pblk_gc_ts` / `pblk_gc_run` 主要负责：1.直接回收full_list，2.选择victim line加入到r_list，并启动pblk_gc_reader_ts线程
	- `pblk_gc_free_full_lines`  将全是无效块的line进行回收
		- 遍历gc_full_list
		- 将line从closed置为gc状态
		- 释放line的最后引用 执行line put操作
	- `pblk_gc_should_run` 判断gc线程是否需要运行（do while循环直到不需要为止）
		- 获取当前空闲块和gc阈值
		- 如果gc启动了，且满足运行条件
	- `pblk_gc_get_victim_line` 从gc队列中选择一个line进行gc
		- 遍历gc队列，选择sec_to_update为0 且 vsc最小的line进行gc
	- 将victim line的状态从closed置为gc
	- 将line从gc队列摘除 插入到`gc->r_list`中
	- `pblk_gc_reader_kick` 唤醒`gc_reader_ts`线程
	- 当前gc_gourp遍历完了 换下一个gc_group

`pblk_gc_ts`线程 -> 启动`pblk_gc_reader_ts`线程 -> 自己睡眠

#### 2. GC读线程

- `pblk_gc_reader_ts` / `pblk_gc_read` 取下line，构造gc_rq读取line上全部有效页。
- 从`gc->r_list`摘下一个line
- `pblk_gc_line` / `pblk_gc_line_prepare_ws` 对一个line进行gc/回收
	- 从line结构体中拷贝invalid bitmap、lba_list、vsc供后续gc迁移使用
	- do while创建gc_rq请求（寻找line上所有有效块）。gc_rq装填lba和line上地址，并将对应的line挂在gc_rq上。
	- 对victim line做gc要增加victim line的引用。
	-                     ` 根据gc_rq，唤醒读线程读取有效块并唤醒gc写线程写入有效块。
		- `pblk_submit_read_gc` 根据gc_rq构造并提交读请求，阻塞读。
			- `read_ppalist_rq_gc` / `read_rq_gc` 得到`gc_rq`所要读取的物理页地址，进一步判断待回收页面是否有效
			- 构造完整的读请求，对设备进行阻塞读，读取结果存在`gc_rq->data`
		- `gc->w_list` 将完成读取的gc_rq，插入到gc->w_list队列
		- `pblk_gc_writer_kick` 唤醒写线程
	- 将line上的全部有效页提交后，释放line的最后引用

`pblk_gc_reader_ts`线程 -> 启动`pblk_gc_writer_ts`线程 -> 自己睡眠

#### 3. GC写线程

- `pblk_gc_writer_ts` / `pblk_gc_write` 将全部gc_rq都写入ring buffer。
	- 从gc->w_list获取一个gc_rq
	- `pblk_write_gc_to_cache` 和用户写类似，这个是将GC读取到的页面写到ring buffer中，大体流程与之类似。
		- `pblk_rb_may_write_gc` 决定本次gc写入是否可以写入ring buffer，并完成ring buffer管理信息更新，返回写入指针
			- `pblk_rl_gc_may_insert` 判断gc写入是否准入ring buffer
			- `pblk_rb_may_write` 进行缓存替换，往环形缓冲区ring buffer中写入指定数量的条目
			- `pblk_rl_gc_in` 更新gc缓冲区使用情况rb_gc_cnt
		- `pblk_rb_write_entry_gc` 真正往ring buffer写入gc条目(装填write context)，并修改l2p表（注: gc修改l2p表会再进行一次gc页面有效检查）
	- 完成后从gc->w_list摘除并释放这个gc_rq；完成了一次落盘gc读-写，将`line->ref`减1

`pblk_gc_writer_ts`线程 -> 自己睡眠


<hr>




## 七、IO逻辑

支持用户传来的：失效、读、写三种操作（参考`pblk_make_rq`）

### （1）失效操作

- `pblk_discard` 失效操作
	- 根据discard命令，获取bio中的lba和nr_secs
	- `pblk_invalidate_range` 
		- 依据lba和nr_secs，获取对应的ppa
		- 如果ppa不在cache或不空，就将其在line结构上标记失效（见有效块的维护与invalidate操作小节）
		- 然后将lba对应的l2p表表项置空

### （2）写操作

主线程只完成ring buffer和l2p表的维护，达到可写入条件会唤醒额外的线程`pblk_write_ts`来提交一个写请求。

#### 1. 相关结构体

- `struct pblk_w_ctx *w_ctx` write context，ring buffer中的每个entry都维护一个write context。其维护有关落盘/刷回的必要信息，是ring buffer和nand flash页面分配写入这两阶段的通信桥梁。
```c
struct pblk_w_ctx {
    struct bio_list bios;  // 原始的bio 用于REQ_FUA和REQ_FLUSH情况
    u64 lba;  //与该条目关联的逻辑地址
    struct ppa_addr ppa;  // 与该条目关联的物理地址
    int flags;  // 该条目操作的标志位
};
```
- `struct pblk_c_ctx *c_ctx` write buffer completion context，为每个请求维护一个completion context，从ring buffer中拷贝数据到落盘请求的bio中后，会对应填写好`sentry`、`nr_valid`和`nr_padded`等字段。
```c
struct pblk_c_ctx {
    struct list_head list;  // 用于处理顺序完成的链表
    unsigned long *lun_bitmap;  // 当前请求中使用的 LUN 位图
    unsigned int sentry;  // 条目所在的位置
    unsigned int nr_valid;  // 有效条目的数量
    unsigned int nr_padded;  // 填充条目的数量
};
```
- `struct nvm_rq *rqd` 最终发往设备的NVMe读/写请求
```c
struct nvm_rq {
    struct nvm_tgt_dev *dev;   // 目标设备
    struct bio *bio;  // 与此请求相关联的bio，存储数据等
    union {
        struct ppa_addr ppa_addr;  // 物理页地址 todo
        dma_addr_t dma_ppa_list;  // DMA 地址列表 todo
    };
    struct ppa_addr *ppa_list;  // PPA 地址列表 todo
    void *meta_list;  // 页面oob
    dma_addr_t dma_meta_list;  // 元数据列表 todo
    nvm_end_io_fn *end_io;  // 执行完该请求所进行的回调函数
    uint8_t opcode;  // 操作码，读/写等
    uint16_t nr_ppas; // 物理页地址的数量
    uint16_t flags;  // 请求的标志
    u64 ppa_status;  // 物理页地址的状态（例如，读取、写入等）
    int error;  // 错误码
    int is_seq;  // 是否是顺序请求的提示标志
    void *private;  // 私有数据
};
```
- `struct pblk_sec_meta *meta` 页面的oob字段，16B。：
```c
struct pblk_sec_meta {
    u64 reserved;
    __le64 lba;
};
```

#### 2. 相关流程

![[Pasted image 20241204112817.png]]

**流程概述** 数据传入，首先会写入ring buffer，在写缓存中l2p只记录cacheline（即`pblk_write_to_cache`），达到`min_write_pgs`，就会向device发起一次落盘写请求，在此过程中分配设备物理地址（即`pblk_submit_write`）。新来的写请求会进一步导致ring buffer的替换，缓存驱逐时会将l2p表的cacheline换成设备地址device ppa。

**流程分析** （参考`pblk_make_rq`）：
- 如果超过rate limit的max_io，将写请求bio进行拆分
- `pblk_write_to_cache` 将用户的写入写到ring buffer中（和ring buffer打交道）
	- `pblk_rb_may_write_user` 决定本次用户写入是否可以写入ring buffer，并完成ring buffer管理信息更新，返回写入指针
		- `pblk_rl_user_may_insert` rate limit检查用户缓冲区使用情况`rb_user_cnt`，判断用户写入是否准入ring buffer
		- `pblk_rb_may_write_flush` 准备往ring buffer写入条目，并更新写入指针
			- `__pblk_rb_may_write`  进行缓存替换，往ring buffer中写入指定数量的条目
			- 进行缓存准入，更新`rb->mem`指针，返回写入指针
			- `pblk_rb_flush_point_set` 如果是FLUSH有关的命令，设置flush_point。
		- `pblk_rl_user_in`  更新用户的缓冲区使用情况`rb_user_cnt`
	- `pblk_rb_write_entry_user` 真正往ring buffer写入用户数据条目(装填write context)，并修改l2p表
	- 判断是否要唤醒写线程进行落盘（连接和通往持久化的桥梁）
- `pblk->writer_ts/pblk_write_ts/pblk_submit_write`，执行持久化落盘写入的写线程（和line管理、物理页等打交道，和`pblk_write_to_cache`通过write context构建桥梁）
	- `pblk_prepare_resubmit` TODO 失败写请求的重新提交
	- 分配写bio和创建设备写请求
	- `pblk_rb_read_to_bio` 将页面数据从ring buffer中拷贝到写bio中，共享的方式
	- `pblk_submit_io_set` 构造一个完成的写请求，分配物理地址，并完成写请求的提交（可能会触发元数据的写入和边写边擦）。
		- `pblk_setup_w_rq` 构造一个完成的写请求，并为lba分配一个ppa
		- `pblk_should_submit_meta_io` 判断现在是否要进行元数据emeta同步
		- `pblk_submit_io` 提交I/O请求，统计历史I/O数（注：`pblk_submit_io`可以指定I/O请求完成后的要执行的回调函数`xxx_end_io`）
		- `pblk_blk_erase_async` 对下一块数据线的擦除操作，边写边擦
		- `pblk_submit_meta_io` 提交meta_line对应的emeta持久化

**特殊case** 对于“空bio”的处理。在使用sync同步的时候，系统通常会往底层发送空的bio请求（即bio中的secs为0）。对于空的bio，rb更新输入0 secs，数据处理时啥也不干直接return。总结是啥也不干。

### （3）读操作 

读请求直接在主线程提交。读请求会从ring buffer和nand flash两个位置中读取lba对应的数据。然而一个bio请求可能同时涵盖以上两种情况。因此在处理读请求的过程，采用了bio切割的方式。bio会不断从头切割成要么满足ring buffer，要么只能从nand flash读的很多子bio。具体分析如下：
- 同样的请求超过大小做拆分
- `pblk_submit_read` 
	- 构造读请求
	- `pblk_read_ppalist_rq` / `pblk_read_rq` 读取lba对应的物理地址，读取ring buffer的缓存数据。截断读取，要么全在cache，要么全在flash。
		- `pblk_lookup_l2p_seq` 截断读取lba对应的物理地址。
		- 如果截断出来的子段全是flash读取，直接返回。
		- `pblk_read_from_cache` 如果截断出来的子段全是ring buffer读取，则从ring buffer拷贝数据到bio，并更新bio数据指针。
	- 将bio进行split，剩下的bio重新提交一个读请求（递归）。
	- split出来的子bio要么已经从ring buffer中读取成功，结束bio和rqd。
	- `pblk_submit_io` 要么要从flash读取。同写操作。


<hr>



## 八、sysfs和LightNVM调试

在/sys/block/myblk/pblk上可以查看相关的文件，文件是实时更新和变化的
```
femu@fvm:/sys/block/myblk/pblk$ ls
errors    lines              padding_dist  write_amp_mileage  write_luns
gc_force  lines_info         ppa_format    write_amp_trip
gc_state  max_sec_per_write  rate_limiter  write_buffer
```



<hr>


## 九、reference
- <a href="http://lightnvm.io/pblk-tools/"> pblk-tools | lightnvm.io</a>
- <a href="https://www.usenix.org/conference/fast17/technical-sessions/presentation/bjorling"> LightNVM | FAST'17</a>
- <a href="https://blog.xiocs.com/archives/36/"> LightNVM 自顶向下的 IO 通路简析 </a>