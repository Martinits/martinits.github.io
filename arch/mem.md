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

- Sequential Consistency

  - 最简单的mem model

  - 如果一个多核cpu执行指令的任何一种可能顺序都保证其结果如同所有核的所有指令按某种顺序执行后的结果，那么这个cpu就是sequentially consistent的

    - 也就是说，每个核的指令顺序都在多核的指令顺序中得到保证

  - 一个符合SC的执行要满足以下两点

    - 每个核把自己指令流插入到多核系统的执行流中，形成一个合并的执行顺序，并保证自己的指令顺序不变
    - 每个load得到的结果等于上一个同地址的store

  - SC的执行顺序就是保证load/store的任意排列都能保证实际执行顺序就等于指令顺序

    - 下表中true意为保证该情况（op1 - op2）下的执行顺序

    - atomic RMW：原子操作Read-Modify-Write

    - |                  | op2 - load | op2 - store | op2 - atomic RMW |
      | ---------------- | ---------- | ----------- | ---------------- |
      | op1 - load       | true       | true        | true             |
      | op1 - store      | true       | true        | true             |
      | op1 - atomic RMW | true       | true        | true             |

- Total Store Order / x86
