# BlockManager状态追踪

## BlockManagerMasterEndpoint

**功能概述**:
`BlockManagerMasterEndpoint` 是一个用于在 Spark 集群的主节点上追踪所有存储端点的 BlockManager 状态的类。它主要用于管理和协调集群中各个 BlockManager 的块存储和状态信息。

**主要方法及使用**:

- **`receiveAndReply(context: RpcCallContext)`**: 处理不同类型的 RPC 消息，并根据请求执行相应的操作。主要处理注册 BlockManager、更新块信息、获取块位置、获取内存状态等。
- **`isRDDBlockVisible(blockId: RDDBlockId)`**: 检查给定 RDD 块的可见性。
- **`updateRDDBlockVisibility(taskId: Long, visible: Boolean)`**: 更新 RDD 块的可见性状态。
- **`removeRdd(rddId: Int)`**: 删除指定 RDD 的所有块，包括从存储端点和外部 Shuffle 服务中删除。
- **`removeShuffle(shuffleId: Int)`**: 删除指定 Shuffle 的所有块，处理相关的块删除操作。

**使用示例**:

- 注册 BlockManager 时，通过 `RegisterBlockManager` 消息发送注册请求。
- 更新块信息时，通过 `UpdateBlockInfo` 消息发送更新请求。
- 获取块位置时，通过 `GetLocations` 消息查询块位置。
- 删除 RDD 时，调用 `removeRdd` 方法来删除相关的块。

该类在 Spark 的 BlockManager 管理和集群调度中扮演了重要的角色，确保了块的正确存储、更新和删除操作。

## `BlockStatus` 类

- **功能**：表示块的存储状态，包括内存和磁盘中的大小。
- **主要方法**：
  
  - `isCached`: 判断块是否被缓存（即是否在内存或磁盘中有数据）。
- **用法**：
  
  ```scala
  val status = BlockStatus(StorageLevel.MEMORY_ONLY, 1024L, 0L)
  val cached = status.isCached  // 返回 true，因为内存大小大于 0
  ```

### `BlockStatusPerBlockId` 类

- **功能**：管理块状态的映射，并在所有块被移除时清空内存。
- **主要方法**：
  
  - `get(blockId: BlockId)`: 获取指定块的状态。
  - `put(blockId: BlockId, blockStatus: BlockStatus)`: 添加或更新块的状态。
  - `remove(blockId: BlockId)`: 移除指定块的状态。
- **用法**：
  
  ```scala
  val blockStatusMap = new BlockStatusPerBlockId
  blockStatusMap.put(blockId, BlockStatus(StorageLevel.MEMORY_ONLY, 1024L, 0L))
  val status = blockStatusMap.get(blockId)
  blockStatusMap.remove(blockId)
  ```

### `BlockManagerInfo` 类

- **功能**：跟踪块管理器的信息，包括内存使用情况和块状态。
- **主要方法**：
  
  - `getStatus(blockId: BlockId)`: 获取指定块的状态。
  - `updateBlockInfo(blockId: BlockId, storageLevel: StorageLevel, memSize: Long, diskSize: Long)`: 更新块的状态信息。
  - `removeBlock(blockId: BlockId)`: 移除指定块的状态。
  - `remainingMem`: 返回剩余的内存大小。
- **用法**：
  
  ```scala
  val managerInfo = new BlockManagerInfo(...)
  managerInfo.updateBlockInfo(blockId, StorageLevel.MEMORY_ONLY, 1024L, 0L)
  val status = managerInfo.getStatus(blockId)
  val freeMemory = managerInfo.remainingMem
  managerInfo.removeBlock(blockId)
  ```

```mermaid
classDiagram
    class BlockStatus {
        +StorageLevel storageLevel // 存储级别
        +Long memSize // 内存中的块大小
        +Long diskSize // 磁盘上的块大小
        +Boolean isCached() // 检查块是否被缓存
        +static BlockStatus empty() // 返回一个空的BlockStatus对象
    }

    class BlockStatusPerBlockId {
        -JHashMap~BlockId, BlockStatus~ blocks // 块 ID 与状态的映射
        +Option~BlockStatus~ get(BlockId blockId) // 获取指定块的状态
        +void put(BlockId blockId, BlockStatus blockStatus) // 添加或更新块状态
        +void remove(BlockId blockId) // 移除指定块状态
    }

    class BlockManagerInfo {
        +BlockManagerId blockManagerId // 块管理器的唯一标识符
        +Long maxOnHeapMem // 最大堆内内存
        +Long maxOffHeapMem // 最大堆外内存
        +RpcEndpointRef storageEndpoint // 存储端点引用
        +Option~BlockStatusPerBlockId~ externalShuffleServiceBlockStatus // 外部 Shuffle 服务块状态
        -Long _lastSeenMs // 最后一次访问时间戳
        -Long _remainingMem // 剩余内存
        -JHashMap~BlockId, BlockStatus~ _blocks // 块 ID 与状态的映射
        +Option~BlockStatus~ getStatus(BlockId blockId) // 获取指定块的状态
        +void updateLastSeenMs() // 更新最后一次访问的时间戳
        +void updateBlockInfo(BlockId blockId, StorageLevel storageLevel, Long memSize, Long diskSize) // 更新块信息
        +void removeBlock(BlockId blockId) // 移除块
        +Long remainingMem() // 获取剩余内存
        +Long lastSeenMs() // 获取最后一次访问的时间戳
        +void clear() // 清除所有块状态
    }

    class BlockManagerMasterEndpoint {
        -JHashMap~BlockManagerId, BlockManagerInfo~ blockManagerInfo // 块管理器信息
        -JHashMap~String, BlockManagerId~ blockManagerIdByExecutor // 根据执行器 ID 获取块管理器 ID
        -JHashMap~BlockId, mutable.HashSet~BlockManagerId~ blockLocations // 块位置映射
        -mutable.HashSet~Int~ invisibleRDDBlocks // 隐藏的 RDD 块
        -JHashMap~String, BlockManagerId~ shuffleMergerLocations // Shuffle 合并器位置
        -Boolean externalShuffleServiceRddFetchEnabled // 外部 Shuffle 服务 RDD 提取启用状态
        +memoryStatus() // 返回内存状态
        +storageStatus() // 返回存储状态
        +blockStatus(blockId: BlockId, askStorageEndpoints: Boolean) // 获取块状态
        +getMatchingBlockIds(filter: BlockId => Boolean, askStorageEndpoints: Boolean) // 获取匹配块 ID
        +externalShuffleServiceIdOnHost(blockManagerId: BlockManagerId) // 获取外部 Shuffle 服务 ID
        +register(idWithoutTopologyInfo: BlockManagerId, localDirs: Array[String], maxOnHeapMemSize: Long, maxOffHeapMemSize: Long, storageEndpoint: RpcEndpointRef, isReRegister: Boolean): BlockManagerId // 注册块管理器
        +updateShuffleBlockInfo(blockId: BlockId, blockManagerId: BlockManagerId): Future[Boolean] // 更新 Shuffle 块信息
        +updateBlockInfo(blockManagerId: BlockManagerId, blockId: BlockId, storageLevel: StorageLevel, memSize: Long, diskSize: Long): Boolean // 更新块信息
        +getLocations(blockId: BlockId): Seq[BlockManagerId] // 获取块位置
        +getLocationsAndStatus(blockId: BlockId, requesterHost: String): Option[BlockLocationsAndStatus] // 获取块位置和状态
        +getLocationsMultipleBlockIds(blockIds: Array[BlockId]): IndexedSeq[Seq[BlockManagerId]] // 获取多个块位置
        +getPeers(blockManagerId: BlockManagerId): Seq[BlockManagerId] // 获取对等块管理器
        +getShufflePushMergerLocations(numMergersNeeded: Int, hostsToFilter: Set[String]): Seq[BlockManagerId] // 获取 Shuffle 推送合并器位置
        +removeShufflePushMergerLocation(host: String) // 移除 Shuffle 合并器位置
        +getExecutorEndpointRef(executorId: String): Option[RpcEndpointRef] // 获取执行器端点引用
        +onStop() // 关闭 Ask 线程池
    }

    BlockManagerInfo --> BlockStatusPerBlockId : externalShuffleServiceBlockStatus
    BlockManagerInfo --> BlockStatus : _blocks
    BlockStatusPerBlockId --> BlockStatus : blocks
    BlockManagerMasterEndpoint --> BlockManagerInfo : blockManagerInfo
    BlockManagerMasterEndpoint --> BlockStatus : blockLocations
    BlockManagerMasterEndpoint --> BlockStatusPerBlockId : externalShuffleServiceBlockStatus
    BlockManagerMasterEndpoint --> BlockManagerId : blockManagerIdByExecutor
    BlockManagerMasterEndpoint --> BlockId : blockLocations
    BlockManagerMasterEndpoint --> ShuffleMergerLocations : shuffleMergerLocations
    class BlockManagerId {
        - String executorId_ // 执行器的唯一标识符
        - String host_ // 块管理器所在的主机
        - Int port_ // 块管理器的端口号
        - Option[String] topologyInfo_ // 网络拓扑信息，用于数据块的复制
        + String executorId // 获取执行器 ID
        + String host // 获取主机名称
        + Int port // 获取端口号
        + Option[String] topologyInfo // 获取拓扑信息
        + Boolean isDriver // 检查是否为 Driver 节点
        + void writeExternal(ObjectOutput out) // 序列化对象
        + void readExternal(ObjectInput in) // 反序列化对象
        + String toString // 返回对象的字符串表示
        + Int hashCode // 返回对象的哈希码
        + Boolean equals(Any that) // 比较两个对象是否相等
        - Object readResolve() // 反序列化后的解决方法，返回缓存中的对象
    }

    class BlockManagerIdCompanion {
        + BlockManagerId apply(String execId, String host, Int port, Option[String] topologyInfo) // 创建新的 BlockManagerId 实例
        + BlockManagerId apply(ObjectInput in) // 从输入流创建 BlockManagerId 实例
        + BlockManagerId getCachedBlockManagerId(BlockManagerId id) // 从缓存中获取 BlockManagerId 实例
        - Cache[BlockManagerId, BlockManagerId] blockManagerIdCache // 缓存 BlockManagerId 实例
        + String SHUFFLE_MERGER_IDENTIFIER // shuffle 合并器标识符
        + String INVALID_EXECUTOR_ID // 无效执行器标识符
    }

    BlockManagerId --> BlockManagerIdCompanion : companion object

   
```
