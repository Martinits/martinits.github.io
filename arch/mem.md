## Theory of Memory and Cache

### Book: A primer on memory consistency and cache coherence

- Consistency是指多cpu共用一块内存（地址空间）时的行为
  - 目的是保证同一块内存不出现data race
  - 也叫memory (consistency) model
  - 硬件提供保证consistency的机制，但是需要软件去合理的使用
- Coherency是指内存数据在别处有拷贝时，如何保证大家都读到最新的数据
  - 比如cache，DMA等
  - 完全有硬件保证，软件无需关心
  - coherency的目标是让多核系统的cache（或类似的东西）完全透明，就像单核系统一样
  - 是optional的，因为可以没有cache，有cache也可以没有coherency