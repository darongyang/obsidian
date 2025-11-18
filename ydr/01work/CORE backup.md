> paper: [[CORE paper]]


## Design
% 思考问题&挑战：如何进行高效的页面数据管理？
% 1.会发生旧数据的覆盖，原先数据会被放大，如何让横向空间存下更多的数据成为一个问题；2.有限的页面和变长的小数据会带来内部碎片化的问题，如何处理？借助：cycle，弹性编码方案；3.压缩场景，非编码区和编码区的取舍，什么东西放编码区？
% 关键的challange：这个二维的空间的高效数据管理和小数据更新表示？这需要面临和考虑以下的问题：（1）空间占用的放大和旧数据的覆盖：更新数据的无状态化，一个区间表示一份数据；（2）确定空间和变长数据带来的页面碎片化问题：弹性的编码，借助Cycle的循环，数据和元数据的分离，数据横向append避免过早的空间缩减，元数据纵向update便于数据定位，借助vmax完成定位。

% 小数据更新的无状态化。
% 如图六所示，传统的小数据更新采用状态化的链式结构进行数据组织，即页面当前状态的构建依赖上一次的状态。
% 和传统的一维横向数据表示不同，二维数据平面具备空间放大和迭代覆盖两个重要特点。
% 基于上述特点，具备状态化的数据追加方式在二维数据空间上无法奏效。
% 多级电压的利用将数据的存储空间进行了放大，使其一次只能同时容纳更少的数据状态。
% 其次，这种方式下，数据的恢复需要依赖前置的数据完成，形成一个链式的数据存储结构。而在二维数据平面中，多级电压的纵向更新和旧数据覆盖必然会破坏这种链式结构。
% 因此在二维数据空间的管理上，一个简单但关键的设计在于让这个空间只用于管理一份无状态的更新数据。

% \textbf{Stateless updates of small data.} 
% As shown in Figure \ref{fig:tradition-fail}, traditional small data updates use a state-dependent chained structure for data organization, meaning the construction of the current page state relies on the previous state. Unlike traditional one-dimensional horizontal data representation, the two-dimensional data plane has two key features: spatial amplification and iterative coverage. Due to these characteristics, the state-dependent data append method is ineffective in a two-dimensional data space. In this approach, data recovery relies on the completion of the preceding data, forming a chained data storage structure. However, in a two-dimensional data plane, the vertical updates and overwriting of old data with multi-level voltage inevitably disrupt this chained structure. Furthermore, the utilization of multi-level voltage amplifies the storage space, reducing the capacity to simultaneously accommodate multiple data states. Therefore, in managing a two-dimensional data space, a simple yet critical design consideration is to use this space exclusively for managing stateless update data.


% 元数据的分离和空间的隐式划分。无状态的热更新数据的最新状态会在二维数据平面进行动态更新。相应地，我们对热更新数据的定位和解码也维护了一个对应的动态热更新的元数据。每次数据的更新，都会带来无状态的数据及其元数据的同时热更新。两者都被维护在支持热更新的二维数据平面内。为了实现元数据的定位，我们将其与灵活更新的数据进行解耦。在二维数据平面的布局上，我们采用隐式划分的布局对两者的空间进行拆分，将元数据维护在平面内的一侧，而数据则从平面的另一侧开始更新。当两者发生不可避免的碰撞的时候，则意味着该二维数据平面表示数据能力达到上限。
\noindent \textbf{Separation of metadata and implicit spatial partitioning.} The latest state of stateless hot-updated data is dynamically refreshed within the dual-dimensional data plane. Correspondingly, we maintain a dynamic hot-updated metadata for locating and decoding the data. Each data update results in simultaneous hot updates of both the stateless data and its metadata. Both are maintained within a dual-dimensional data plane that supports hot updates. To facilitate metadata location, we decouple it from the flexibly updated data. In the layout of the dual-dimensional data plane, we employ an implicit partitioning scheme to separate the spatial areas of metadata and data, with metadata maintained on one side of the plane and data updated from the opposite side. When collisions between the two inevitably occur, it indicates that the data capacity of the dual-dimensional data plane has reached its limit.


% 数据的二维更新策略。
% 元数据字段会包括数据定位的必要信息，例如起始位置和长度。关于其开销，对于一个4KB的物理页面，对其做编码前，元数据大小仅为3B。在不同等级的编码中，其进一步被放大为4B、6B和12B，这是可接受的。
% 和数据的变长特点不同，元数据的长度是固定的。借助高密度闪存的多级电压特点，可以很好的对这些元数据进行定位管理。如图x所示，我们从闪存页面的一侧开始记录元数据。定长元数据以其长度为一个滑动窗口。元数据会采用纵向覆盖优先的方式进行元数据的更新。当该滑动窗口内部的最大电压达到Vmax的时候。向迭代的方式。先后版本的元数据可以通过cover。
% 元数据的字段，编码与大小开销。元数据的编码管理，先纵向后横向。元数据定长+可cover+Vmax的有界性，纵向更新。将最大电压Vmax作为数据单元失效标志。
% 元数据更新详细描述：元数据窗口。纵向更新达到Vmax。按窗口滑动。滑动会落到old data上。
% 解码描述：窗口滑动，直到窗口不存在Vmax，该窗口有效。
% However, 这会带来Valid Page空间的缩小。

% 数据变长+编码空间翻倍，为了避免碎片化的问题，应该尽可能避免Valid Page空间的缩小。
% 为了避免编码空间过早出现存储单元达到最大电压Vmax而缩小编码空间，数据在编码空间的拓展方式和编码元数据的方式正好相反。
% 数据在编码空间中 采用的拓展方式是先横向后纵向的可更新拓展方式。数据的定位和解码受编码元数据的维护，编码元数据中记录着当前编码空间中存储的数据的起始位置和结束位置。新的数据会接着当前旧的数据的结束位置继续横向拓展，直到抵达当前元数据的位置处停止。如图xxx所示，完成一次完整的横向拓展后，数据会进行一次纵向拓展，然后重新开始新的一轮横向拓展。

% Elastic Code和比特填充。如图xx所示，输入的原始数据会具有不确定的长度。
% 元数据的编码，

% Cycle的概念


% 


% 编码元数据的可更新存储：本文方案的编码元数据包含差量在编码空间中的定位和编码类型等信息。在4KB大小的NAND闪存页面中，本文方案的编码元数据总大小仅为3B。在不同等级的方案下，分别占用NAND闪存页面的4B、6B和12B，是完全可接受的。由于差量数据动态存在于编码空间中，起始位置和其大小均不固定，因此需要编码元数据的维护。但编码元数据也会在空间中进行滑动，起始位置并不固定。编码元数据的定位和解码是编码空间管理的一个关键问题。在我们的方案中，对编码元数据采用了先纵向后横向的可更新拓展方式来解决这个问题。和差量数据有一点不同之处，在于编码元数据的大小是固定的，本文方案充分利用了这一点。如图xxx所示，该方案将最大电压Vmax作为编码元数据单元失效的标志。该方案会对编码元数据先尝试进行纵向更新，直到纵向更新达到最大电压Vmax。此时，编码元数据会更新失败，进行横向移动一个编码元数据的大小的距离，重新进行纵向更新，以此往复。值得注意的是，横向移动并重新纵向更新未必一定要从空白的存储单元上开始，可以建立在被旧的差量数据编程过的存储单元上。在编码元数据的定位和解码时，会从编码空间的编码元数据存储一侧开始，每次滑动大小为编码元数据的大小的窗口，检查该窗口内是否出现最大电压Vmax，如果出现了最大电压，则会继续滑动，直到没有出现最大电压的窗口，该窗口内是当前有效的编码元数据。本质上，该策略是将横向移动前达到最大电压Vmax的编码元数据单元内的全部存储单元从编码空间中进行丢弃，等效于编码空间在不断减小。

\textbf{Updatable Storage of Encoding Metadata.} The encoding metadata in our approach includes information such as the positioning of deltas within the encoding space and the encoding types. In a 4KB NAND flash memory page, the total size of encoding metadata in our scheme is only 3B. Under different configurations, it occupies 4B, 6B, and 12B of the NAND flash page, which is entirely acceptable. Since delta data is dynamically present in the encoding space with variable starting positions and sizes, maintenance of encoding metadata is necessary. However, encoding metadata also undergoes spatial shifts, with its starting position not being fixed. The positioning and decoding of encoding metadata are key issues in managing the encoding space. In our approach, we address this problem by employing a vertical-first, then horizontal updatable expansion method for encoding metadata. Unlike delta data, encoding metadata has a fixed size, which is fully leveraged in our scheme. As illustrated in Figure xxx, this approach uses the maximum voltage Vmax as a failure indicator for encoding metadata units. The scheme attempts vertical updates of encoding metadata until it reaches Vmax. At this point, when vertical updates fail, it horizontally shifts by the size of one encoding metadata unit and resumes vertical updates, repeating this process. Notably, horizontal shifting and subsequent vertical updates do not necessarily start from blank storage cells; they can be based on cells previously programmed with old delta data (as discussed in the next section). During encoding metadata positioning and decoding, a sliding window of the size of one encoding metadata unit is moved across the encoding metadata storage side of the encoding space. The window checks for the presence of Vmax; if Vmax is detected, the window continues sliding until a window without Vmax is found, indicating the current valid encoding metadata. Essentially, this strategy discards all storage cells within encoding metadata units that have reached Vmax before horizontal shifting, effectively reducing the size of the encoding space. 


% 差量数据的可更新存储：差量数据的原始长度是不固定的，在WOM-v(3,4) 方案中，每3比特会编码为4比特，因而需要考虑差量数据的长度和 对应的编码单位保持对齐的问题。在本文方案中，不对齐的差量数据首先会进行零填充，直到原始的差量数据的字节数和对应编码的编码单位的比特数保持对齐。为了避免编码空间过早出现存储单元达到最大电压Vmax而缩小编码空间，差量数据在编码空间的拓展方式和编码元数据的方式正好相反。差量数据在编码空间中 采用的拓展方式是先横向后纵向的可更新拓展方式。差量数据的定位和解码受编码元数据的维护，编码元数据中记录着当前编码空间中存储的差量数据的起始位置和结束位置。新的差量数据会接着当前旧的差量数据的结束位置继续横向拓展，直到抵达当前编码元数据的位置处停止。如图xxx所示，完成一次完整的横向拓展后，差量数据会进行一次纵向拓展，然后重新开始新的一轮横向拓展。不断迭代更新的输入到编码空间的差量会以这种方式在空间中进行不断拓展更新。


\textbf{Updatable Storage of Delta Data.} The original length of delta data is variable. In the WOM-v(3,4) scheme, every 3 bits are encoded as 4 bits, thus requiring alignment between the length of delta data and the corresponding encoding units. In our approach, misaligned delta data is first zero-padded until the byte count of the original delta data aligns with the bit count of the corresponding encoding units. To prevent the encoding space from prematurely shrinking due to storage cells reaching the maximum voltage Vmax, the expansion method for delta data within the encoding space is designed to be the opposite of that for encoding metadata. Delta data is expanded using a horizontal-first, then vertical updatable expansion method. The positioning and decoding of delta data are managed by the encoding metadata, which records the starting and ending positions of the delta data stored in the current encoding space. New delta data extends horizontally from the end position of the current old delta data until it reaches the position of the current encoding metadata. As shown in Figure xxx, after completing a full horizontal expansion, delta data undergoes vertical expansion before restarting a new round of horizontal expansion. The continuously iterating delta data inputs into the encoding space are updated and expanded in this manner.

% 编码空间的可更新极限：如图xxx所示，图的横坐标代表了编码空间的全部存储单元，左侧的纵坐标代表了差量数据可以进行的完整横向拓展的次数，称为轮数。右侧的纵坐标代表了存储单元不同的电压级别。基于前面分析，在该方案中，元数据采用先纵向后横向的拓展方式，差量数据采用先横向后纵向的拓展方式。编码空间中的存储单元可更新的极限只看存储单元的纵向拓展，当纵向拓展程度达到最大编程电压 Vmax时，存储单元无法再进行页面更新，等价于可用于更新的编码空间的长度减小。当可用于更新的编码空间的长度减小到0时，编码空间达到其可更新的极限。

\textbf{Updatable Limits of the Encoding Space.} As illustrated in Figure xxx, the horizontal axis represents all storage cells within the encoding space, the left vertical axis indicates the number of complete horizontal expansions that delta data can undergo, referred to as the number of iterations, and the right vertical axis denotes different voltage levels of the storage cells. Based on previous analysis, in this scheme, metadata expands vertically first and then horizontally, while delta data expands horizontally first and then vertically. The updatable limit of storage cells in the encoding space is determined solely by vertical expansion. When vertical expansion reaches the maximum programming voltage Vmax, the storage cells can no longer be updated, effectively reducing the length of the encoding space available for updates. When the length of the encoding space available for updates is reduced to zero, the encoding space reaches its updatable limit.