

---

## Score Overview

**#63 Update Your High-density SSD Like a Tetris Game: Balance Lifetime, I/O, and Space Efficiency on High-Density SSD with TetrisSSD**

|            | OveMer | RevExp |
| ---------- | ------ | ------ |
| Review 63A | 2      | 3      |
| Review 63B | 2      | 3      |
| Review 63C | 3      | 3      |

---

## Review#63A

​**​Overall merit​**​  
2. Weak reject

​**​Reviewer expertise​**​  
3. Knowledgeable
### Paper summary

The paper proposes TetrisSSD, a high-density SSD architecture designed to improve the lifetime, I/O efficiency, and space utilization of QLC/PLC SSDs by leveraging redundancy codes (specifically WOM-v) more intelligently. TetrisSSD introduces two techniques—Tetris Updater, which densely packs only the updated data within pages, and Tetris Reshaper, which adaptively selects encoding strategies based on I/O patterns. This selective and adaptive encoding minimizes unnecessary writes and space overhead. Experimental results using benchmarks and real-world traces show that TetrisSSD outperforms existing WOM-v-based SSDs, extending lifetime by 25-58% while also improving I/O and space efficiency.
### ​Comments for authors​

Thanks for submitting the paper to FAST'26. The paper addresses the challenge of using WOW (Write-Once Memory) codes and proposes a solution aimed at further improving SSD lifetime. While the proposed approach is interesting, the key concern is its practicality in real-world deployments due to potentially high management overhead (e.g., mapping table overhead) and complications introduced by dynamic SSD capacity. Below are detailed comments:
- The overall writing quality of the paper needs improvement. Several explanations lack depth, and notations used in figures are often poorly defined. For instance, Section 3 presents conclusions without adequate derivation, making it difficult to follow. In Figure 6, the sudden introduction of ElasticSSD and DeltaSSD results is confusing. The paper does not clarify how these schemes relate to WOW codes or why their results are relevant to the discussion.
- The paper lacks sufficient detail on how logical page addresses (LPAs) are updated and indexed. For example, in the case of a small update on the bits (e.g., 1st, 3rd, 5th,... bits), it remains unclear how these bits are stored in delta pages and how their positions are tracked. How are these delta updates generated, and what is the typical size? The update mechanism is difficult to understand. Additionally, it is unclear how the delta update headers are obtained and used.
- It is also unclear how data split between base and delta pages is recovered or read. Does the system require a bit-level mapping mechanism to determine the latest values? The reconstruction logic for a logical page composed of a base and multiple delta pages should be more thoroughly discussed.
- There is a conceptual gap in the description of the transition from traditional page writes to encoded pages. The paper mentions that initially, new data is written in a traditional format. However, when updates arrive, it does not explain how the system handles them in relation to the previously unencoded data. It appears the authors assume the data has already been encoded, which may not always be the case in practice.
- Regarding reshape updates, different WOM codes result in variable space consumption. This implies that SSD capacity may change dynamically, which is problematic for filesystem-level management. The paper does not address how this dynamic capacity is handled or mitigated at the system level.
- The methodology for lifetime evaluation is unclear. What formula or criteria is used to compute lifetime? Is it based solely on erase counts, or are other factors considered? A clear and formal definition of the lifetime metric is necessary.
- The experimental setup raises concerns. The SSD capacity used in the evaluation is only 16GB, which is not representative of typical QLC SSDs, which are generally much larger. Moreover, using a 4KB page size seems not reasonable, as larger page sizes are more realistic and better suited for QLC.

## Review#63B

​**​Overall merit​**​  
2. Weak reject

​**​Reviewer expertise​**​  
3. Knowledgeable
​
### ​Paper summary​
​  
This paper proposes TetrisSSD, a technique designed to balance lifetime, space efficiency, and write efficiency for high-density SSDs. To better leverage redundancy codes (R-codes) such as WOM-v, TetrisSSD introduces two key components—Tetris Updater and Tetris Reshaper—which are implemented within the SSD controller. To balance lifetime with space and I/O efficiency, TetrisSSD dynamically selects different strategies (e.g., various versions of WOM-v or no WOM-v) depending on data hotness and update size when writing data. For small-sized updates, TetrisSSD mitigates redundancy-induced overhead by generating a compressed delta from the base page and the updated portion, which is then stored in a delta page. In this manner, it can optimize write efficiency for fine-grained updates. While prior approaches focused primarily on extending SSD lifetime—often at the cost of space and write efficiency—TetrisSSD achieves a more holistic optimization. Compared to WOM-v-based SSDs, it improves space efficiency by up to 1.87x and write efficiency by up to 2.52x. In terms of endurance, TetrisSSD outperforms both raw SSDs and WOM-v-based SSDs.

### Comments for authors​​  
Thank you for submitting your work to FAST'26. This paper provides a solid background and a clear explanation of prior techniques, which helps readers understand the motivation behind TetrisSSD. The experimental results also highlight that endurance is a critical issue for high-density SSDs, thereby supporting the motivation well. Overall, the paper is well-written and easy to follow. However, several critical issues must be addressed before it can be considered for acceptance.

I do not have significant concerns regarding the design of TetrisSSD—it is described in sufficient detail and appears to be reasonable. My main concerns lie in the experimental environment, system configurations, and the feasibility of the proposed approach. TetrisSSD relies on various data structures and operations (such as XORing and encoding) and even assumes the availability of hardware compressors. However, all evaluations are conducted using software-based emulation with FEDU, without any analysis of the associated overheads in realistic SSD platforms. 32GB DRAM with eight cores is too fast and rich to emulate an SSD with wimpy ARM cores. This makes it difficult to assess how feasible TetrisSSD would be in a real-world SSD controller.

Some assumptions in the paper also appear outdated. For example, the page size is assumed to be 4KB throughout the paper, which is no longer typical in modern SSDs. A page size can significantly impact the efficiency of TetrisSSD, yet the paper lacks any in-depth analysis on this aspect. Furthermore, it is unclear whether employing hardware accelerators or adopting computational storage devices (CSD) solely to support WOM codes—given their limited benefits (as discussed below)—would be a practical design choice, especially when considering hardware cost.

Beyond feasibility, the practical benefits of TetrisSSD seem limited. First, the paper does not report read latency. Since the WOM-v technique increases the number of pages read per request—and TetrisSSD adds delta pages on top of that—read latency could be negatively impacted. Second, while TetrisSSD shows improved space and write efficiency compared to other WOM-based approaches, it still performs worse than Raw-SSD in these metrics. While extending SSD lifetime is important, I am not convinced it is reasonable to sacrifice space efficiency, write efficiency, and (potentially) cost-effectiveness for endurance alone.

The paper mentions the use of an extended L2P table, but does not explain how it is implemented. In a typical L2P table, the LBA is used as an index to map to a single PPA. But, in the proposed scheme, a single LBA can be mapped to multiple PPAs, which likely requires a different data structure. Storing the PPA for each delta page would also require additional space. Maybe, I missed something here. Could the authors clarify what kind of structure was used? Also, the paper says the additional elements increase the L2P table size by 7%. Does that include the overhead from having multiple PPAs per LBA?

The explanation of R-Code could also be improved for readers unfamiliar with the concept. It took me a while to interpret Figure 3 and fully understand how incremental updates are achieved using R-Code.

The efficiency of R-code depends on bit patterns of input data. How could you emulate those patterns? Do your benchmarks contain actual data in addition to I/O traces?

Finally, it appears that the left and right figures in Figure 4 are identical. Could the authors confirm if this is an error?

## Review#63C

​**​Overall merit​**​  
3. Weak accept

​**​Reviewer expertise​**​  
3. Knowledgeable

​
### ​Paper summary​
​  
A QLC flash cell can normally store 4 bits, and sustain some number (call it N) program-erase cycles; if using a WOM(2,4) code it can store 2 bits, but sustain 5N program/erase cycles. TetrisSSD uses a novel method of packing compressed updates "horizontally"— across cells in a page— and "vertically", across WOM voltage states— to achieve a better lifetime/capacity tradeoff than prior work.

### ​Comments for authors​

**Strengths:**
- Interesting combination of WOM coding and delta compression

**Weaknesses:**
- Lack of comparison to delta-FTL, SLC cache
- No mention of LDPC, read retry
- No discussion of additional wear due to use of full V_th range

Thank you for submitting your paper to FAST 2026.

I've always been interested in WOM codes, and was pleased to read this paper; however I worry that such codes are an idea that is more attractive in theory than in practice. In particular, this paper does not (a) evaluate against anything other than a naive strawman baseline, nor does it (b) evaluate the additional wear per P/E cycle incurred when using WOM coding

(a) The primary problem with this paper is that it is difficult to figure out its improvements over prior art, as all comparisons are to a hypothetical pure QLC unencoded SSD.

I'm not aware of any pure QLC (or TLC) SSDs on the market— my understanding is that standard practice is to reserve a pool of pages which are used in SLC mode as a cache, as described in [Jiminez2015], below. Delta-FTL [20] is an alternative approach, using similar placement and compression mechanisms to TetrisSSD but without the multi-write WOM coding; [26] combines both of them.

Without knowing whether this work performs better than either Delta-FTL or naive SLC cache it is not possible to determine whether this work improves on the state of the art.

[Jimenez2015] Xavier Jimenez, David Novo, and Paolo lenne. 2015. Libra: Software-Controlled Cell Bit-Density to Balance Wear in NAND Flash. ACM Trans. Embed. Comput. Syst. 14, 2, Article 28 (March 2015), 22 pages.(https://doi.org/10.1145/2638552)

(b) potential additional wear: one of the reasons that SLC-mode cache is used with MLC/TLC/QLC devices is the lower cell wear when doing LSB-only programming—e.g. see section 5 of the reference below. TetrisSSD minimizes program-erase cycles, but it does so by subjecting cells to the highest stress possible, so additional analysis will be needed to determine the tradeoff— it seems possible that it may do more harm than good.

Additional points:

Error correction/ read retry— it may be beyond the scope of this paper, or covered in one of the cited WOM papers, but I would like to see at least a brief mention of whether or not LDPC or similar strength codes with read retry may be implemented on top of WOM codes, as ECC strength plays a significant role in determining the number of feasible P/E cycles.

"Tetris"— It took me most of a full pass through the paper to understand why you were Tetris. You might want to describe the two dimensions early on (bits/cells= horizontal, coding level=vertical) and mention that your Tetris pieces are all of height 1, but various widths. This seems to be the sort of thing that— at least when I'm writing— has been in the author's head so long that one forgets to explain it to the reader.

Trace age: the FIU traces are a decade and a half old; however they're the only traces I know of that include content information, at 512B granularity for "homes", and 4KB granularity for other traces. It would be better to base the work on newer traces, but to my knowledge they don't exist.

V_th: There's a slight issue with terminology concerning cell states. V_th is the threshold voltage, i.e. the minimum gate-to-source voltage at which the transistor will conduct. It is not the "voltage on a cell", despite the frequent use of that term, and it cannot be measured directly; instead a single-read-out operation applies a specific voltage V to the gate, and measures a binary signal indicating whether V_th is lower or higher than V. What it's sensing can be accurately referred to as either the threshold voltage (what's being directly probed) or the charge on the cell, which is being measured indirectly. This also means that multi-level cell sensing is complicated and often slow, requiring binary probing to determine V_th levels. This is a major reason for exposing each bit on a different page— reading pages sequentially will read MSB first and LSB last, performing one readout per page instead of the log2(#levels) required to read an arbitrary page.

I'm mostly offering this for the authors' information; however I think there are a few places where the wording should be adjusted slightly for correctness.