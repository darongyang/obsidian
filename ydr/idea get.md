## motivation story
- topic: what happen when the ZNS-like SSDs meet the hot data compression?
> the LightNVM, the platform I use, is FTL of ZNS-like SSDs
- ZNS can make a data classification, aggregating the hots data together, called hot zones (seprate the cold data and the hot data). The data pages in the same zone have the similar update frequency
- hots zone suffer the frequent data update and need to earse the hots zone frequently
- data comppression in FTL encounters the fragementation problem and hot data compression encounters the severe WAF
- Using WOMv code directly encounters the space and the performance problem!

## CSD fragementation problems
### DataBase
Total Fragmentation: 463712 bytes free, 7.87% fragmentation overall
Total Compression Ratio: 16.17%

### DICOM pictures
Total Fragmentation: 877765 bytes free, 17.52% fragmentation overall
Total Compression Ratio: 41.44%

### HTML
Total Fragmentation: 4284296 bytes free, 20.75% fragmentation overall
Total Compression Ratio: 39.47%

### EXE
Total Fragmentation: 7996879 bytes free, 28.27% fragmentation overall
Total Compression Ratio: 39.62%


### Mixed
Total Fragmentation: 12995583 bytes free, 20.25% fragmentation overall
Total Compression Ratio: 34.95%

## zns的数据分类方案
![[Screenshot_2024-09-05-16-46-08-506_com.miui.gallery-edit.jpg]]
- 压缩率感知，减少页面内部的碎片化
	- 压缩盘内部的碎片化问题严重
> 粗粒度压缩率（>50%）避免放在一个zone内
- 生命周期感知，减少gc写放大
	- ai训练，实现data dream的中间件
	- 特征提取：基于lba，基于content，其他io特征
> csal已经基于VMs进行了数据分类

测试

Flash Cache和WOM-v
![[56381a9719535af6bde2aa04e3b712c.jpg]]
