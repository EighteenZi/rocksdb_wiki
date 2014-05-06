### When to need it
When the DB uses tons of gigabytes of memory, there is a higher chance that when accessing data in memory, the program will get data TLB miss, as well as cache misses to retrieve the mapping. When using hash table based indexes and bloom filters, provided by some mem tables and table readers, users are more prone to it, because the data locality is worse. Those indexes and blooms are perfect candidates to be allocated in huge page TLB, since you get relatively worse data locality in those data structures. When you see a high data TLB overheads in data structures supporting huge pate TLB, Consider to give this feature a try.

Currently, it is only supported in Linux.

### How to use it
#### Requisition 
* You need to pre-reserve huge pages in linux
* Find out the huge page size to allocate
See Linux Documentation/vm/hugetlbpage.txt for details.

#### Configure
Here are where the feature is available to use and how:
1. mem table's bloom filter: set Options.memtable_prefix_bloom_huge_page_tlb_size to be the huge page size
2. hash linked list mem table's indexes and bloom filters: when calling NewHashLinkListRepFactory() to create a factory object for the mem table, pass huge page size into the parameter huge_page_tlb_size.
3. PlainTableReader's indexes and bloom filters. When creating table factory for plain tables using NewPlainTableFactory() or NewTotalOrderPlainTableFactory(), set the parameter huge_page_tlb_size to be the huge page size.