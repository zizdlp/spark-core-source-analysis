# MemoryStore

`MemoryStore` 类用于在内存中管理和存储数据块。它有多种内存管理策略，包括处理数据块的序列化和反序列化。以下是该类主要功能和方法的简要说明：

1. **成员变量和构造函数**：
   - `entries`: 使用 `LinkedHashMap` 存储所有块的信息。
   - `onHeapUnrollMemoryMap` 和 `offHeapUnrollMemoryMap`: 分别用于记录堆内存和堆外内存中用于解卷的内存量。
   - `unrollMemoryThreshold`: 解卷块之前请求的初始内存量。

2. **主要方法**：
   - `putBytes`: 尝试将一个字节数组放入内存。如果内存足够，就将块存储为字节；否则返回 `false`。
   - `putIterator`: 尝试将一个迭代器中的数据放入内存。这是一个逐步解卷的过程，以避免内存溢出。
   - `putIteratorAsValues` 和 `putIteratorAsBytes`: 分别将数据块以值或字节形式存储到内存中。
   - `getBytes` 和 `getValues`: 根据块 ID 获取存储的字节或值。
   - `remove`: 移除指定块的内存条目，并释放相关内存。
   - `clear`: 清除所有块，释放所有内存。
   - `evictBlocksToFreeSpace`: 尝试通过逐出其他块来释放指定空间。

3. **内存管理**：
   - `reserveUnrollMemoryForThisTask` 和 `releaseUnrollMemoryForThisTask`: 分别用于请求和释放用于解卷的内存。
   - `currentUnrollMemory` 和 `currentUnrollMemoryForThisTask`: 获取当前内存使用情况。

4. **日志记录**：
   - 类中大量使用日志记录（`logInfo`, `logWarning` 等）来跟踪内存使用情况和操作结果。

## 代码中的一些关键点

- **线程安全**：在处理内存和块操作时，使用 `synchronized` 确保线程安全。
- **内存策略**：支持堆内存和堆外内存两种存储模式，并且根据内存情况动态调整内存使用策略。
- **解卷策略**：为了防止 OOM 异常，`putIterator` 方法会逐步解卷数据，同时动态请求内存。

!!! tip unrolling

    在Spark的内存管理中，“unrolling”指的是将数据从磁盘或其他存储介质中读取到内存中的过程。这通常涉及到将数据从序列化格式转化为对象格式，以便在计算中使用。在Spark中，“unrolling”是处理存储中还没完全处理的“溢出”数据时发生的，这些数据被存储在磁盘上而不是内存中。

    因此，你提到的 `blocksMemoryUsed` 函数计算的是当前被用来缓存数据块的内存量，不包括用于“unrolling”过程的内存。它通过从总的内存使用量中减去“unrolling”过程所使用的内存来获得这个值。这样可以更准确地反映出用于缓存数据块的实际内存量。
