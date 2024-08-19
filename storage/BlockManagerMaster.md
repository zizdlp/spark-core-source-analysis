# BlockManagerMaster

`BlockManagerMaster` 类是 Apache Spark 的一个内部组件，用于在集群中管理和协调块管理器（Block Manager）。它负责以下主要功能：

1. **块管理器的注册与管理**：
   - **`registerBlockManager`**：注册一个新的块管理器到驱动端，获取块管理器的完整信息，包括拓扑信息。
   - **`removeExecutor`** 和 **`removeExecutorAsync`**：从驱动端移除死掉的执行器，并可以选择异步操作。

2. **块和数据的管理**：
   - **`updateBlockInfo`**：更新块的信息，包括存储级别、内存大小和磁盘大小。
   - **`removeBlock`**：从所有存储端点中移除指定的块。
   - **`removeRdd`**、**`removeShuffle`** 和 **`removeBroadcast`**：移除指定的RDD、shuffle或广播相关的所有块。

3. **块的可见性和状态**：
   - **`isRDDBlockVisible`**：检查RDD块是否可见。
   - **`getBlockStatus`**：获取块在所有块管理器上的状态，包括可能的未来状态。
   - **`getLocations`** 和 **`getLocationsAndStatus`**：获取块的位置信息和状态。

4. **集群状态和配置**：
   - **`getMemoryStatus`** 和 **`getStorageStatus`**：获取块管理器的内存和存储状态。
   - **`getPeers`**：获取集群中其他块管理器的ID。
   - **`getShufflePushMergerLocations`** 和 **`removeShufflePushMergerLocation`**：管理shuffle推送合并的位置。

5. **其他功能**：
   - **`stop`**：停止驱动端点，通常在驱动程序停止时调用。
   - **`getExecutorEndpointRef`**：获取指定执行器的RPC端点引用。

这个类的主要作用是通过 RPC（远程过程调用）与集群中的块管理器进行通信，管理块的存储、状态更新、内存和其他相关操作，确保 Spark 的数据处理任务能够高效地运行。

```mermaid
classDiagram
    class BlockManagerMaster {
        +removeExecutor(execId: String) // 从驱动端移除一个死掉的执行器
        +decommissionBlockManagers(executorIds: Seq[String]) // 停用对应一组执行器的块管理器
        +getReplicateInfoForRDDBlocks(blockManagerId: BlockManagerId) // 获取指定块管理器中所有RDD块的复制信息
        +removeExecutorAsync(execId: String) // 异步请求从驱动端移除一个死掉的执行器
        +registerBlockManager(id: BlockManagerId, localDirs: Array[String], maxOnHeapMemSize: Long, maxOffHeapMemSize: Long, storageEndpoint: RpcEndpointRef, isReRegister: Boolean = false) BlockManagerId // 注册块管理器的ID到驱动端
        +updateBlockInfo(blockManagerId: BlockManagerId, blockId: BlockId, storageLevel: StorageLevel, memSize: Long, diskSize: Long) Boolean // 更新块信息
        +updateRDDBlockTaskInfo(blockId: RDDBlockId, taskId: Long) // 更新RDD块的任务信息
        +updateRDDBlockVisibility(taskId: Long, visible: Boolean) // 更新RDD块的可见性
        +isRDDBlockVisible(blockId: RDDBlockId) Boolean // 检查块是否可见
        +getLocations(blockId: BlockId) Seq[BlockManagerId] // 获取块ID的位置信息
        +getLocationsAndStatus(blockId: BlockId, requesterHost: String) Option[BlockLocationsAndStatus] // 获取块ID的位置信息和状态
        +getLocations(blockIds: Array[BlockId]) IndexedSeq[Seq[BlockManagerId]] // 获取多个块ID的位置信息
        +contains(blockId: BlockId) Boolean // 检查块管理器主控是否有指定块
        +getPeers(blockManagerId: BlockManagerId) Seq[BlockManagerId] // 获取集群中其他节点的ID
        +getShufflePushMergerLocations(numMergersNeeded: Int, hostsToFilter: Set[String]) Seq[BlockManagerId] // 获取成功注册的shuffle服务的位置信息
        +removeShufflePushMergerLocation(host: String) // 从候选列表中移除shuffle推送合并位置
        +getExecutorEndpointRef(executorId: String) Option[RpcEndpointRef] // 获取执行器的RpcEndpointRef
        +removeBlock(blockId: BlockId) // 从存储端点中移除一个块
        +removeRdd(rddId: Int, blocking: Boolean) // 移除属于指定RDD的所有块
        +removeShuffle(shuffleId: Int, blocking: Boolean) // 移除属于指定shuffle的所有块
        +removeBroadcast(broadcastId: Long, removeFromMaster: Boolean, blocking: Boolean) // 移除属于指定广播的所有块
        +getMemoryStatus Map[BlockManagerId, (Long, Long)] // 返回每个块管理器的内存状态
        +getStorageStatus Array[StorageStatus] // 获取存储状态
        +getBlockStatus(blockId: BlockId, askStorageEndpoints: Boolean = true) Map[BlockManagerId, BlockStatus] // 返回块在所有块管理器上的状态
        +getMatchingBlockIds(filter: BlockId => Boolean, askStorageEndpoints: Boolean) Seq[BlockId] // 返回匹配给定过滤器的块ID列表
        +stop() // 停止驱动端点
    }

    BlockManagerMaster : -timeout // RpcUtils.askRpcTimeout(conf)
    BlockManagerMaster : -driverEndpoint // RpcEndpointRef
    BlockManagerMaster : -driverHeartbeatEndPoint // RpcEndpointRef
    BlockManagerMaster : -conf // SparkConf
    BlockManagerMaster : -isDriver // Boolean

    class BlockManagerMaster {
        +tell(message: Any) // 发送单向消息到主控端
    }
```
