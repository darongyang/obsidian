总结。
原因是line被释放，line的bitmap为空，但是invalidate的时候还在使用bitmap导致的。
有效块被迁移，新的写入沿用了旧的页面地址，新的写入发现页面无效要失效，失效到了已经无效的line上。
gc到底触发了没有？==待确定==
pblk_gc_line_ws -> pblk_gc_write
gc写cache写了没有？==没有==
gc w list都是空的，line是怎么被put的
gc的写请求落盘了没有？分配物理地址了没有？l2p表修改了没有？
gc和用户写是可以并行的吗？
fio跑的大小超出磁盘大小了吗？

原因是：read_ppalist_rq_gc读取有效块的时候失败了，导致没有成功进行有效块的迁移，为什么呢？可能原因是：line0选择为gc_line -> 发起对line0有效块的写请求 -> line0存在有效块引用 -> gc失败

没有用好line->sec_to_update，修改l2p表为cacheline的时候就得get

现在的逻辑：
更新l2p表的cacheline -> 为该lba申请实际物理地址，此刻获取line -> sec_to_update -> 从缓存淘汰后，持久化成功，释放line->ref 


核心问题：
1.为什么原来逻辑中cacheline中有line的块，就无法进行gc？
- cacheline中存着最新数据，并将旧的disk数据进行invalidate了
- l2p的ppa记录的是cacheline，同时应该分配了一个新地址ppa，hold住了line的sec_to_update，可能落盘或没落盘
- gc不能选择还有数据备份在cacheline中的那些line？
- gc读取读请求、更新l2p表还会额外检查当前的lba对应的ppa是不是要gc的ppa
- 假设可以gc，gc会迁移这个cacheline对应的数据块，创建一个新的cacheline2，缓存中会同时存在这个数据块的两个缓存，同时gc会更新l2p表指向这个cacheline。此外gc可能会选中正在用于open写的line。
- 感觉没有什么一定不能gc的理由
2.如果cacheline中有line的块，该如何进行gc？
- 去掉sec_to_update变量，可以选择位于缓存中的页面所在的line作为擦除单元
- 去掉读请求、更新l2p表的检查，pblk_ppa_comp
- 注意把，地址替换为计算出来的gc地址
- 在naive测试一下，跑通fio
- 然而，没有跑通，为什么呢？
- 顺序读写寄了，没有任何反应，
- 随机读写呢


<span style="background:#fff88f">构建一套全面的GC方案。</span>




```
[  345.863334] BUG: kernel NULL pointer dereference, address: 00000000000001e8
[  345.864494] #PF: supervisor write access in kernel mode
[  345.865305] #PF: error_code(0x0002) - not-present page
[  345.866122] PGD 0 P4D 0 
[  345.866535] Oops: 0002 [#1] SMP PTI
[  345.867094] CPU: 4 PID: 4393 Comm: pblk-writer-t Tainted: G        W  OE     5.4.0-64-generic #72-Ubu
[  345.868566] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS rel-1.16.2-0-gea1b7a073390-p4
[  345.870348] RIP: 0010:__pblk_map_invalidate+0x39/0x120 [pblk]
[  345.871252] Code: 55 4c 8d ae b8 00 00 00 41 54 49 89 fc 4c 89 ef 53 48 89 f3 e8 58 a9 6f d5 83 7b 10
[  345.874149] RSP: 0018:ffffb6b1c0c5bc90 EFLAGS: 00010246
[  345.874967] RAX: 0000000000000000 RBX: ffff8f696ddc8000 RCX: 0000000000000000
[  345.876092] RDX: 0000000000000001 RSI: ffff8f696ddc8000 RDI: ffff8f696ddc80b8
[  345.877208] RBP: ffffb6b1c0c5bcb0 R08: 0000000000000f7d R09: 0000000000000000
[  345.878315] R10: 0000000000000000 R11: ffff8f69fffd3000 R12: ffff8f69ee853000
[  345.879432] R13: ffff8f696ddc80b8 R14: 0000000000000f7d R15: ffffb6b1c04412f0
[  345.880554] FS:  0000000000000000(0000) GS:ffff8f69fbb00000(0000) knlGS:0000000000000000
[  345.881812] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[  345.882721] CR2: 00000000000001e8 CR3: 0000000133a5a001 CR4: 00000000007606e0
[  345.883845] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
[  345.884951] DR3: 0000000000000000 DR6: 00000000fffe0ff0 DR7: 0000000000000400
[  345.886067] PKRU: 55555554
[  345.886517] Call Trace:
[  345.886914]  pblk_map_invalidate+0xe1/0xf0 [pblk]
[  345.887650]  pblk_map_page_data+0x42d/0x560 [pblk]
[  345.888399]  pblk_map_erase_rq+0x146/0x310 [pblk]
[  345.889131]  pblk_write_ts+0xe03/0xe60 [pblk]
[  345.889813]  kthread+0x104/0x140
[  345.890328]  ? pblk_submit_meta_io+0x310/0x310 [pblk]
[  345.891119]  ? kthread_park+0x90/0x90
[  345.891700]  ret_from_fork+0x35/0x40
[  345.892267] Modules linked in: pblk(OE) binfmt_misc dm_multipath scsi_dh_rdac scsi_dh_emc scsi_dh_aly
[  345.901798] CR2: 00000000000001e8
[  345.902324] ---[ end trace 13207bfeccdc9c60 ]---
[  345.903059] RIP: 0010:__pblk_map_invalidate+0x39/0x120 [pblk]
[  345.903970] Code: 55 4c 8d ae b8 00 00 00 41 54 49 89 fc 4c 89 ef 53 48 89 f3 e8 58 a9 6f d5 83 7b 10
[  345.906924] RSP: 0018:ffffb6b1c0c5bc90 EFLAGS: 00010246
[  345.907761] RAX: 0000000000000000 RBX: ffff8f696ddc8000 RCX: 0000000000000000
[  345.908884] RDX: 0000000000000001 RSI: ffff8f696ddc8000 RDI: ffff8f696ddc80b8
[  345.910004] RBP: ffffb6b1c0c5bcb0 R08: 0000000000000f7d R09: 0000000000000000
[  345.911134] R10: 0000000000000000 R11: ffff8f69fffd3000 R12: ffff8f69ee853000
[  345.912266] R13: ffff8f696ddc80b8 R14: 0000000000000f7d R15: ffffb6b1c04412f0
[  345.914037] FS:  0000000000000000(0000) GS:ffff8f69fbb00000(0000) knlGS:0000000000000000
[  345.915919] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[  345.917423] CR2: 00000000000001e8 CR3: 0000000133a5a001 CR4: 00000000007606e0
[  345.919152] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
[  345.920854] DR3: 0000000000000000 DR6: 00000000fffe0ff0 DR7: 0000000000000400
[  345.922567] PKRU: 55555554
```


```
[  258.475803] BUG: kernel NULL pointer dereference, address: 00000000000001e8
[  258.476952] #PF: supervisor write access in kernel mode
[  258.477775] #PF: error_code(0x0002) - not-present page
[  258.478577] PGD 0 P4D 0 
[  258.478977] Oops: 0002 [#1] SMP PTI
[  258.479518] CPU: 1 PID: 2333 Comm: pblk-writer-t Tainted: G        W  OE     5.4.0-64-generic #u
[  258.480956] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS rel-1.16.2-0-gea1b7a0734
[  258.482712] RIP: 0010:__pblk_map_invalidate+0x39/0x120 [pblk]
[  258.483611] Code: 55 4c 8d ae b8 00 00 00 41 54 49 89 fc 4c 89 ef 53 48 89 f3 e8 58 f9 a4 dd 830
[  258.486489] RSP: 0018:ffffa12a40d0bc90 EFLAGS: 00010246
[  258.487304] RAX: 0000000000000000 RBX: ffff909736928000 RCX: 0000000000000000
[  258.488404] RDX: 0000000000000001 RSI: ffff909736928000 RDI: ffff9097369280b8
[  258.489519] RBP: ffffa12a40d0bcb0 R08: 0000000000000f7e R09: 0000000000000000
[  258.490605] R10: 0000000000000000 R11: ffff90973ffd3000 R12: ffff909730bfc000
[  258.491712] R13: ffff9097369280b8 R14: 0000000000000f7e R15: ffffa12a4048d338
[  258.492826] FS:  0000000000000000(0000) GS:ffff90973ba40000(0000) knlGS:0000000000000000
[  258.494086] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[  258.494974] CR2: 00000000000001e8 CR3: 000000007440a006 CR4: 00000000007606e0
[  258.496066] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
[  258.497178] DR3: 0000000000000000 DR6: 00000000fffe0ff0 DR7: 0000000000000400
[  258.498294] PKRU: 55555554
[  258.498739] Call Trace:
[  258.499152]  pblk_map_invalidate+0xe1/0xf0 [pblk]
[  258.499903]  pblk_map_page_data+0x42d/0x560 [pblk]
[  258.500664]  pblk_map_erase_rq+0x146/0x310 [pblk]
[  258.501418]  pblk_write_ts+0xe03/0xe60 [pblk]
[  258.502115]  kthread+0x104/0x140
[  258.502641]  ? pblk_submit_meta_io+0x310/0x310 [pblk]
[  258.503442]  ? kthread_park+0x90/0x90
[  258.504035]  ret_from_fork+0x35/0x40
[  258.504614] Modules linked in: pblk(OE) binfmt_misc dm_multipath scsi_dh_rdac scsi_dh_emc scsi_i
[  258.514100] CR2: 00000000000001e8
[  258.514650] ---[ end trace 34df00e75df0cef6 ]---
[  258.515384] RIP: 0010:__pblk_map_invalidate+0x39/0x120 [pblk]
[  258.516289] Code: 55 4c 8d ae b8 00 00 00 41 54 49 89 fc 4c 89 ef 53 48 89 f3 e8 58 f9 a4 dd 830
[  258.519182] RSP: 0018:ffffa12a40d0bc90 EFLAGS: 00010246
[  258.520006] RAX: 0000000000000000 RBX: ffff909736928000 RCX: 0000000000000000
[  258.521117] RDX: 0000000000000001 RSI: ffff909736928000 RDI: ffff9097369280b8
[  258.522232] RBP: ffffa12a40d0bcb0 R08: 0000000000000f7e R09: 0000000000000000
[  258.523348] R10: 0000000000000000 R11: ffff90973ffd3000 R12: ffff909730bfc000
[  258.524461] R13: ffff9097369280b8 R14: 0000000000000f7e R15: ffffa12a4048d338
[  258.526166] FS:  0000000000000000(0000) GS:ffff90973ba40000(0000) knlGS:0000000000000000
[  258.527987] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[  258.529466] CR2: 00000000000001e8 CR3: 000000007440a006 CR4: 00000000007606e0
[  258.531141] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
[  258.532802] DR3: 0000000000000000 DR6: 00000000fffe0ff0 DR7: 0000000000000400
[  258.534469] PKRU: 55555554
^Cqemu-system-x86_64: terminating on signal 2
```