---
title: 第一篇blog
date: 2023-03-07 11:33:57
tags: 测试
---

# raft
## 实现

[raft-rs](https://github.com/tikv/raft-rs) 提供了最核心的 `Consensus Module`，而其他的组件，包括 `Log`，`State Machine`，`Transport`，都是需要应用去定制实现。

- **Storage**  
  基于`trait Storage`实现`RaftStorage`
    - **RaftStorage**

    ``` 
    impl Storage for RaftStorage {
    fn initial_state(&self) -> raft::Result<RaftState> {
        Ok(self.initial_state())
    }

    fn first_index(&self) -> raft::Result<u64> {
        Ok(self.first_index())
    }

    fn last_index(&self) -> raft::Result<u64> {
        Ok(self.last_index())
    }

    fn term(&self, idx: u64) -> raft::Result<u64> {
        self.term(idx)
    }

    fn entries(
        &self,
        low: u64,
        high: u64,
        max_size: impl Into<Option<u64>>,
    ) -> raft::Result<Vec<Entry>> {
        self.entries(low, high, max_size)
    }

    fn snapshot(&self, request_index: u64) -> raft::Result<Snapshot> {
        self.snapshot(request_index)
    }
}
```

- **Log and State Machine**

  `raft`的运行原理如下图所示：

  ![raft](img/raft.png)

  `Raft`的模型是一个基于`Log`复制的状态机模型。客户端向服务端`Leader`发起写入数据操作，`Leader`将该操作添加到`Log`并复制给所有`Follower`，当超过半数节点确认就可以将这条操作应用到`State Machine`中。

  通过`Log`复制的方式保证所有节点`Log`顺序一致，其目的是**保证`State Machine`中数据状态的一致性**。随着数据量的积累`Log`会不断增大，实际应用中会在适当时机对日志进行压缩，对当前`State Machine`的数据状态进行快照，将其作为应用数据的基础，并重新记录日志。一般的`Raft`应用中`Log`的数据轻，而`State Machine`的数据重，做快照的开销大，不宜频繁使用。
  而本实现作为区块链系统中的共识模块，关注重点在于利用`Raft`的`Consensus Module`。`State Machine`的数据是`ConsensusConfig`，并非真正的区块链的状态，它是为`Consensus Module`的正常运行服务的，而`Log`的数据是`Proposal`，相比之下`Log`的数据过于沉重。充分利用这一实际应用特点和日志压缩的原理，这里的做法是：每个`Proposal`被应用之后都对`State Machine`的数据状态进行快照并本地保存，并不断清空已被应用`Proposal`，数据状态一致性（`Log`查询不到会用快照同步）和重启状态恢复（本地保存的快照）都通过快照来实现。

- **Transport**

  该能力由[network](https://cita-cloud-docs.readthedocs.io/zh_CN/latest/architecture.html#network) 实现


- **启动及运行流程**  
  ![setup](img/raft_setup.png)

  运行流程中的`handle ready`步骤按照[raft-rs文档](https://docs.rs/raft/latest/raft/#processing-the-ready-state) 实现
