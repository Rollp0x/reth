# CallX Feature Migration Guide

**版本**: v1.8.3  
**创建日期**: 2025-11-07  
**功能**: 实现 `eth_callX` RPC 方法  
**用途**: 记录所有代码改动细节，便于未来版本 merge 后快速恢复

---

## 目录
1. [功能概述](#功能概述)
2. [文件改动清单](#文件改动清单)
3. [详细改动内容](#详细改动内容)
4. [升级后恢复步骤](#升级后恢复步骤)
5. [测试验证](#测试验证)

---

## 功能概述

`eth_callX` 是 `eth_call` 的扩展版本，提供更详细的执行信息：
- ✅ 返回执行过程中的事件日志
- ✅ 返回 Gas 使用量
- ✅ 返回执行状态 (成功/失败)
- ✅ 返回 Revert 错误信息
- ✅ 支持性能优化参数 (ignoreLogs)

**与 Geth 的区别**:
- Reth 在 Revert 时也返回 logs（Geth 返回 nil）
- 使用 Reth 原生的 `transact_call_at`，代码更简洁

---

## 文件改动清单

### 新增文件
1. `crates/rpc/rpc-eth-types/src/call_x.rs` - 类型定义 (93 行)
2. `CALLX_FEATURE.md` - 功能文档
3. `CALLX_MIGRATION_GUIDE.md` - 本文档

### 修改文件
1. `crates/rpc/rpc-eth-types/src/lib.rs` - 导出新模块
2. `crates/rpc/rpc-eth-api/src/core.rs` - 添加 API 定义和 handler
3. `crates/rpc/rpc-eth-api/src/helpers/call.rs` - 实现核心逻辑

---

## 详细改动内容

### 1. 新增文件: `crates/rpc/rpc-eth-types/src/call_x.rs`

**完整内容**:

```rust
//! CallX RPC response types - extended call response with logs and execution details

use alloy_primitives::{Bytes, B256};
use alloy_rpc_types_eth::Log;
use serde::{Deserialize, Serialize};

/// Extended call result that includes logs and execution status
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct LogOrRevert {
    /// Block number where the call was executed
    pub block_number: u64,
    /// Block hash where the call was executed
    pub block_hash: B256,
    /// Execution status (1 = success, 0 = failure)
    pub status: u64,
    /// Gas used by the call
    pub used_gas: u64,
    /// Logs generated during execution
    #[serde(skip_serializing_if = "Option::is_none")]
    pub logs: Option<Vec<Log>>,
    /// Return data from the call
    pub returns: Bytes,
    /// Revert error message if call reverted
    #[serde(skip_serializing_if = "Option::is_none")]
    pub revert_error: Option<String>,
}

impl LogOrRevert {
    /// Creates a new successful call result
    pub fn new_success(
        block_number: u64,
        block_hash: B256,
        used_gas: u64,
        logs: Vec<Log>,
        returns: Bytes,
    ) -> Self {
        Self {
            block_number,
            block_hash,
            status: 1, // success
            used_gas,
            logs: Some(logs),
            returns,
            revert_error: None,
        }
    }

    /// Creates a new failed call result
    pub fn new_failure(
        block_number: u64,
        block_hash: B256,
        used_gas: u64,
        logs: Option<Vec<Log>>,
        revert_error: Option<String>,
    ) -> Self {
        Self {
            block_number,
            block_hash,
            status: 0, // failure
            used_gas,
            logs,
            returns: Bytes::new(),
            revert_error,
        }
    }
}

/// Optional parameters for CallX
#[derive(Debug, Clone, Default, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct CallXArgs {
    /// Whether to ignore logs in the response (for performance optimization)
    #[serde(skip_serializing_if = "Option::is_none")]
    pub ignore_logs: Option<bool>,
    /// Whether to evaluate gas usage
    #[serde(skip_serializing_if = "Option::is_none")]
    pub eval_gas: Option<bool>,
}

impl CallXArgs {
    /// Returns true if logs should be ignored
    pub fn should_ignore_logs(&self) -> bool {
        self.ignore_logs.unwrap_or(false)
    }

    /// Returns true if gas should be evaluated
    pub fn should_eval_gas(&self) -> bool {
        self.eval_gas.unwrap_or(false)
    }
}
```

**关键点**:
- `#[serde(rename_all = "camelCase")]` 确保 JSON 字段名正确转换
- `LogOrRevert::new_success` 和 `new_failure` 简化创建
- `CallXArgs` 提供可选参数

---

### 2. 修改文件: `crates/rpc/rpc-eth-types/src/lib.rs`

**位置**: 文件开头的模块声明部分

**原代码** (大约第 11 行):
```rust
pub mod block;
pub mod builder;
pub mod cache;
pub mod error;
pub mod fee_history;
```

**新增代码**:
```rust
pub mod block;
pub mod builder;
pub mod cache;
pub mod call_x;  // ← 新增这一行
pub mod error;
pub mod fee_history;
```

**位置**: 文件中部的导出部分

**原代码** (大约第 28 行):
```rust
pub use cache::{
    config::EthStateCacheConfig, db::StateCacheDb, multi_consumer::MultiConsumerLruCache,
    EthStateCache,
};
pub use error::{EthApiError, EthResult, RevertError, RpcInvalidTransactionError, SignError};
```

**新增代码**:
```rust
pub use cache::{
    config::EthStateCacheConfig, db::StateCacheDb, multi_consumer::MultiConsumerLruCache,
    EthStateCache,
};
pub use call_x::{CallXArgs, LogOrRevert};  // ← 新增这一行
pub use error::{EthApiError, EthResult, RevertError, RpcInvalidTransactionError, SignError};
```

---

### 3. 修改文件: `crates/rpc/rpc-eth-api/src/core.rs`

#### 3.1 添加 API 方法定义

**位置**: `EthApi` trait 中，`eth_call` 方法之后

**查找标记**:
```rust
    /// Executes a new message call immediately without creating a transaction on the block chain.
    #[method(name = "call")]
    async fn call(
        &self,
        request: TxReq,
        block_number: Option<BlockId>,
        state_overrides: Option<StateOverride>,
        block_overrides: Option<Box<BlockOverrides>>,
    ) -> RpcResult<Bytes>;
```

**在其后添加**:
```rust
    /// Extended version of `eth_call` that returns execution logs and detailed status.
    /// 
    /// This is similar to `eth_call` but returns additional information including:
    /// - Event logs generated during execution
    /// - Gas used
    /// - Execution status
    /// - Revert error messages
    ///
    /// Useful for debugging and analyzing smart contract executions.
    #[method(name = "callX")]
    async fn call_x(
        &self,
        request: TxReq,
        block_number: Option<BlockId>,
        state_overrides: Option<StateOverride>,
        block_overrides: Option<Box<BlockOverrides>>,
        call_args: Option<reth_rpc_eth_types::CallXArgs>,
    ) -> RpcResult<reth_rpc_eth_types::LogOrRevert>;
```

#### 3.2 添加 Handler 实现

**位置**: `EthApiServer` impl 块中，`call` handler 之后

**查找标记**:
```rust
    /// Handler for: `eth_call`
    async fn call(
        &self,
        request: RpcTxReq<T::NetworkTypes>,
        block_number: Option<BlockId>,
        state_overrides: Option<StateOverride>,
        block_overrides: Option<Box<BlockOverrides>>,
    ) -> RpcResult<Bytes> {
        trace!(target: "rpc::eth", ?request, ?block_number, ?state_overrides, ?block_overrides, "Serving eth_call");
        Ok(EthCall::call(
            self,
            request,
            block_number,
            EvmOverrides::new(state_overrides, block_overrides),
        )
        .await?)
    }
```

**在其后添加**:
```rust
    /// Handler for: `eth_callX`
    async fn call_x(
        &self,
        request: RpcTxReq<T::NetworkTypes>,
        block_number: Option<BlockId>,
        state_overrides: Option<StateOverride>,
        block_overrides: Option<Box<BlockOverrides>>,
        call_args: Option<reth_rpc_eth_types::CallXArgs>,
    ) -> RpcResult<reth_rpc_eth_types::LogOrRevert> {
        trace!(target: "rpc::eth", ?request, ?block_number, ?state_overrides, ?block_overrides, ?call_args, "Serving eth_callX");
        Ok(EthCall::call_x(
            self,
            request,
            block_number,
            EvmOverrides::new(state_overrides, block_overrides),
            call_args,
        )
        .await?)
    }
```

---

### 4. 修改文件: `crates/rpc/rpc-eth-api/src/helpers/call.rs`

**位置**: `EthCall` trait 中，`call` 方法之后

**查找标记**:
```rust
    /// Executes the call request (`eth_call`) and returns the output
    fn call(
        &self,
        request: RpcTxReq<<Self::RpcConvert as RpcConvert>::Network>,
        block_number: Option<BlockId>,
        overrides: EvmOverrides,
    ) -> impl Future<Output = Result<Bytes, Self::Error>> + Send {
        async move {
            let res =
                self.transact_call_at(request, block_number.unwrap_or_default(), overrides).await?;

            ensure_success(res.result)
        }
    }
```

**在其后添加**:
```rust
    /// Executes the call request (`eth_callX`) and returns extended output including logs
    fn call_x(
        &self,
        request: RpcTxReq<<Self::RpcConvert as RpcConvert>::Network>,
        block_number: Option<BlockId>,
        overrides: EvmOverrides,
        call_args: Option<reth_rpc_eth_types::CallXArgs>,
    ) -> impl Future<Output = Result<reth_rpc_eth_types::LogOrRevert, Self::Error>> + Send {
        use reth_rpc_eth_types::LogOrRevert;

        async move {
            let block_id = block_number.unwrap_or_default();
            
            // Execute the transaction
            let res = self.transact_call_at(request.clone(), block_id, overrides.clone()).await?;
            
            // Get block hash
            let block_hash = if let BlockId::Hash(hash) = block_id {
                hash.block_hash
            } else {
                self.provider().block_hash_for_id(block_id)
                    .map_err(Self::Error::from_eth_err)?
                    .ok_or(EthApiError::HeaderNotFound(block_id))?
            };
            
            // Get block header for block number and timestamp
            let header = self.cache().get_header(block_hash).await
                .map_err(Self::Error::from_eth_err)?;
            
            let block_num = header.number();
            let gas_used = res.result.gas_used();

            // Check if we should ignore logs
            let should_ignore_logs = call_args.as_ref().map(|args| args.should_ignore_logs()).unwrap_or(false);
            
            // Get logs from the execution result (if available and not ignored)
            let logs = if !should_ignore_logs {
                res.result.logs().iter().map(|log| {
                    alloy_rpc_types_eth::Log {
                        inner: alloy_primitives::Log {
                            address: log.address,
                            data: log.data.clone(),
                        },
                        block_hash: Some(block_hash),
                        block_number: Some(block_num),
                        block_timestamp: Some(header.timestamp()),
                        transaction_hash: None, // eth_call doesn't have a transaction hash
                        transaction_index: None,
                        log_index: None,
                        removed: false,
                    }
                }).collect::<Vec<_>>()
            } else {
                Vec::new()
            };

            match &res.result {
                ExecutionResult::Success { output, .. } => {
                    Ok(LogOrRevert::new_success(
                        block_num,
                        block_hash,
                        gas_used,
                        logs,
                        output.clone().into_data(),
                    ))
                }
                ExecutionResult::Revert { output, .. } => {
                    let revert_error = RevertError::new(output.clone()).to_string();
                    Ok(LogOrRevert::new_failure(
                        block_num,
                        block_hash,
                        gas_used,
                        if should_ignore_logs { None } else { Some(logs) },
                        Some(revert_error),
                    ))
                }
                ExecutionResult::Halt { reason, .. } => {
                    let halt_error = format!("execution halted: {:?}", reason);
                    Ok(LogOrRevert::new_failure(
                        block_num,
                        block_hash,
                        gas_used,
                        if should_ignore_logs { None } else { Some(logs) },
                        Some(halt_error),
                    ))
                }
            }
        }
    }
```

**关键实现细节**:
1. 使用 `transact_call_at` 执行交易
2. 智能获取 `block_hash`：如果是 Hash 直接提取，否则查询
3. 从 `ExecutionResult` 提取 logs
4. 区分三种情况：Success、Revert、Halt
5. Revert 和 Halt 时也返回 logs（比 Geth 更好）

---

## 升级后恢复步骤

### 场景：从 upstream 合并新版本后

假设您从 `paradigmxyz/reth` 的 v1.9.0 合并到您的 fork：

#### 步骤 1: 合并代码
```bash
git fetch upstream
git checkout dev-v1.9.0  # 创建新分支
git merge upstream/v1.9.0
```

#### 步骤 2: 处理冲突（如果有）

**可能的冲突文件**:
- `crates/rpc/rpc-eth-types/src/lib.rs` - 模块导出列表
- `crates/rpc/rpc-eth-api/src/core.rs` - API trait 定义
- `crates/rpc/rpc-eth-api/src/helpers/call.rs` - 实现逻辑

**解决策略**:
1. 保留新版本的所有改动
2. 按照本文档第 3 节的指引，重新添加 CallX 相关代码

#### 步骤 3: 恢复 CallX 功能

**3.1 复制类型定义文件**:
```bash
# 从旧分支复制 call_x.rs
git show dev-v1.8.3:crates/rpc/rpc-eth-types/src/call_x.rs > crates/rpc/rpc-eth-types/src/call_x.rs
```

**3.2 修改 lib.rs**:
参考第 2 节，添加：
- 模块声明: `pub mod call_x;`
- 导出: `pub use call_x::{CallXArgs, LogOrRevert};`

**3.3 修改 core.rs**:
参考第 3 节，添加：
- API 方法定义（在 `EthApi` trait 中）
- Handler 实现（在 `EthApiServer` impl 中）

**3.4 修改 call.rs**:
参考第 4 节，添加 `call_x` 方法实现

#### 步骤 4: 验证编译
```bash
cargo check --package reth-rpc-eth-api
cargo check --package reth-rpc-eth-types
```

#### 步骤 5: 运行测试（如果有）
```bash
cargo test --package reth-rpc-eth-api
```

---

## 快速检查清单

恢复功能后，使用此清单确保所有改动都已应用：

- [ ] **文件存在**: `crates/rpc/rpc-eth-types/src/call_x.rs`
- [ ] **lib.rs**: 包含 `pub mod call_x;`
- [ ] **lib.rs**: 包含 `pub use call_x::{CallXArgs, LogOrRevert};`
- [ ] **core.rs**: `EthApi` trait 包含 `call_x` 方法定义
- [ ] **core.rs**: `EthApiServer` impl 包含 `call_x` handler
- [ ] **call.rs**: `EthCall` trait 包含 `call_x` 实现
- [ ] **编译通过**: `cargo check` 无错误
- [ ] **JSON 序列化**: 确认 `#[serde(rename_all = "camelCase")]` 存在
- [ ] **logs 返回**: 确认 Revert/Halt 时返回 logs（不是 None）

---

## 测试验证

### 手动测试

**启动 Reth 节点后**，使用 curl 测试：

```bash
# 测试成功调用
curl -X POST http://localhost:8545 \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "eth_callX",
    "params": [{
      "to": "0x合约地址",
      "data": "0x函数签名"
    }, "latest"],
    "id": 1
  }'

# 预期返回:
# {
#   "jsonrpc": "2.0",
#   "id": 1,
#   "result": {
#     "blockNumber": 12345678,
#     "blockHash": "0x...",
#     "status": 1,
#     "usedGas": 21000,
#     "logs": [...],
#     "returns": "0x..."
#   }
# }
```

### 自动化测试（TODO）

创建 `crates/rpc/rpc-eth-api/tests/call_x_tests.rs`:

```rust
#[tokio::test]
async fn test_call_x_success() {
    // TODO: 实现测试
}

#[tokio::test]
async fn test_call_x_revert_with_logs() {
    // TODO: 验证 revert 时返回 logs
}

#[tokio::test]
async fn test_call_x_ignore_logs() {
    // TODO: 验证 ignoreLogs 参数
}
```

---

## 依赖的 Reth 版本特性

此功能依赖以下 Reth v1.8.3 的特性：

1. **`transact_call_at`**: 执行交易的核心方法
2. **`BlockId`**: 区块标识符（Hash/Number/Tag）
3. **`ExecutionResult`**: EVM 执行结果（包含 logs）
4. **`alloy_rpc_types_eth::Log`**: 日志类型
5. **`EthApiError`**: 错误类型

如果新版本修改了这些接口，需要相应调整实现。

---

## 版本兼容性说明

### v1.8.3 → v1.9.x

**预期兼容性**: 高

**可能的变化**:
- Trait 方法签名可能增加新参数（使用默认值）
- 错误类型可能重构（需要调整错误处理）
- EVM 执行接口可能优化（核心逻辑不变）

### v1.8.3 → v2.0.x

**预期兼容性**: 中等

**可能的重大变化**:
- RPC 框架可能升级（jsonrpsee 版本）
- 类型系统可能重构（Alloy 版本升级）
- 执行层接口可能改变（需要重写部分逻辑）

**建议**:
- 仔细阅读 CHANGELOG
- 逐步测试每个修改点
- 保留此文档作为参考

---

## 相关资源

- **Geth 参考实现**: 查看 `api_ext.go` 中的 `CallX` 方法
- **Reth 文档**: https://paradigmxyz.github.io/reth/
- **Alloy 文档**: https://alloy.rs/
- **原始 Issue/PR**: (如果有，在此记录链接)

---

## 维护记录

| 日期 | 版本 | 修改内容 | 修改人 |
|------|------|---------|--------|
| 2025-11-07 | v1.8.3 | 初始实现 | - |

---

## 附录：完整 diff 参考

如需查看完整 diff，使用：

```bash
git diff upstream/v1.8.3 dev-v1.8.3 -- \
  crates/rpc/rpc-eth-types/src/call_x.rs \
  crates/rpc/rpc-eth-types/src/lib.rs \
  crates/rpc/rpc-eth-api/src/core.rs \
  crates/rpc/rpc-eth-api/src/helpers/call.rs
```

---

**文档版本**: 1.0  
**最后更新**: 2025-11-07  
**状态**: ✅ 已验证并提交到 dev-v1.8.3
