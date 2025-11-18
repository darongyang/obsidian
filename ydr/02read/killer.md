chao1 intro

- contributions
  - file operations bottlenecks
  - write once WOFS
  - implement KILL and evaluation

chap2 bg & moti

- PM I/O'stack strong crash consistency
  - PM into & 2 key advantages
  - Fast I/O advantages on PM
  - Strong crash consistency on PM
  - Two main challanges:
        1.I/O amplification
        2.Squander I/O bandwith

- Existing works
  - main tech: disaggregated meta
  - JFS
    - joural area + workflow: D->JM->JC->M1..Mn
    - PMFS write / create workflow
    - PMFS drawback: long ording point
    - SplitFS parallel the JM and JC
    - SplitFS drawback: double meta with EXT4
  - LFS
    - log-structured layout + GC + workflow: D->(GC)->JM->JC
    - NOVA use inode log, wp, dram index, lightweight trans.
    - NOVA drawback: unorchestrated I/Os from disaggregated meta and GC
    - disaggregated meta waste bandwidth
  - Other
    - shadow paging: costly
    - soft update: disaggregated meta
    - hardware transactional memory
  - Summary
    - ALL: Fail to fully align with PM I/O characteristics

- crash consistency overhead
  - I/O time can even exceed the data I/O time
  - theoretical analysis
  - motivation: how to minimize meta I/O
  
chap3 WOFS Design

- overview and principles:
  - pkg conponent: meta + commit
  - wofs workflow
  - principle:
    - avoid data checksum
    - minimal pkgs -> pkgs design
    - effcient pkg abstraction -> pkgs trans. layer
    - avoid logging -> layout and soft GC
    - fast failure recovery -> coarse persistence
- pkgs design
  - inspired by CRUD
  - atomic pkgs
    - create
    - unlink
    - write
    - attr
  - compand pkgs
    - be composed of atomic pkgs
    - forward pointer
    - concurrently persist
- pkgs trans. layer
  - compatible realizations
  - component: Inode table, File/Dir Abstraction
- layout and soft GC
  - two reasons why not non-log layout
  - mentioned the soft GC without details
  - the time of invalidation of different pkgs (main)
- coarse persistence
  - centralized normal bitmap: induces ramdom I/O
  - straightforward method
    - dump-restore: dump at umount
    - DR optimize: tag and skip unallocated blocks
  - coarse persistence
    - pkg-group
    - how the JM|JC|Mcp works

chap4 Implementation

- data layout
  - 4 KiB granularity
  - superblock
  - data and pkgs
  - reservation for bitmap

- space management
  - data blocks aollcator: on each CPU
  - pkgs aollcator: <group addr, mem node> mem node(bitmap) -> pkg-group -> pkgs in group

- PTL and soft GC
  - per-core manner
  - soft GC : on verhead

- Concurrency
  - casual-ordering: independent in parallel
  - causally related with serialization

- Atomicity and Recovery
  - CoW for atomicity
  - Revovery workflow
    - create and unlink pkgs: inode table and dent list
    - write pkgs: data lists
    - attr pkgs: attributes

- File Operation
  - Dir
    - 128 hash table
    - create/unlink workflow
  - File
    - data list and write/read op. workflow

Other
- append reuse avoid recalculate
- huge alloc for append to reduce pkgs
- prefetch reading

chap5 Evaluation

- concern
  - robust crash consistency
  - release I/O performance (metadata)
  - how affect
  - other PM discussion
  
- setup
  - 16-core 128GiB-dram 256GiB-PM

- Reliability
  - test methods (functional testing)
  - the diff between ideal and recovery in vary crash points

- I/O performance
  - significant benefits  when per I/O size less than 16KiB/IO
  - drop after 16KiB/IO and solution
  -  excellent read performance due to RA

-  Tail Latecys
   -  xxxx

- xxx


