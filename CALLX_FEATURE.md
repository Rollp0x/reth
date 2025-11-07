# CallX Feature - Extended eth_call Implementation

## 概述

`eth_callX` 是对标准 `eth_call` 的扩展版本，提供更详细的执行信息，包括事件日志、Gas 使用情况和执行状态。

这个功能是基于 Geth 的类似实现移植到 Reth 的，主要用于智能合约调试和执行分析。

## 功能对比

### eth_call (标准版本)
- 返回：执行结果的字节数据
- 失败时：返回错误

### eth_callX (扩展版本)
返回一个 `LogOrRevert` 结构，包含：
- `blockNumber`: 执行所在的区块号
- `blockHash`: 执行所在的区块哈希
- `status`: 执行状态 (1 = 成功, 0 = 失败)
- `usedGas`: 消耗的 Gas
- `logs`: 执行过程中产生的事件日志 (可选)
- `returns`: 返回数据
- `revertError`: Revert 错误信息 (如果有)

## API 定义

```json
{
  "jsonrpc": "2.0",
  "method": "eth_callX",
  "params": [
    {
      "from": "0x...",
      "to": "0x...",
      "data": "0x..."
    },
    "latest",
    null,  // state overrides (可选)
    null,  // block overrides (可选)
    {      // callX args (可选)
      "ignoreLogs": false,
      "evalGas": true
    }
  ],
  "id": 1
}
```

## 返回示例

### 成功执行
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "blockNumber": 12345678,
    "blockHash": "0x...",
    "status": 1,
    "usedGas": 21000,
    "logs": [
      {
        "address": "0x...",
        "topics": ["0x..."],
        "data": "0x...",
        "blockNumber": 12345678,
        "blockHash": "0x...",
        "transactionHash": null,
        "transactionIndex": null,
        "logIndex": null,
        "removed": false
      }
    ],
    "returns": "0x0000000000000000000000000000000000000000000000000000000000000001"
  }
}
```

### 执行失败 (Revert)
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "blockNumber": 12345678,
    "blockHash": "0x...",
    "status": 0,
    "usedGas": 23450,
    "logs": [],
    "returns": "0x",
    "revertError": "execution reverted: insufficient balance"
  }
}
```

## CallX 可选参数

### ignoreLogs
- 类型: `boolean`
- 默认: `false`
- 说明: 如果设置为 `true`，不会返回日志信息，可以提高性能

### evalGas
- 类型: `boolean`
- 默认: `false`
- 说明: 是否评估 Gas 使用（预留参数）

## 使用场景

### 1. 智能合约调试
查看合约执行过程中触发的所有事件：
```javascript
const result = await provider.send("eth_callX", [
  {
    to: contractAddress,
    data: encodedFunctionCall
  },
  "latest"
]);
console.log("Events:", result.logs);
```

### 2. Gas 估算
获取准确的 Gas 使用情况：
```javascript
const result = await provider.send("eth_callX", [
  { to: contractAddress, data: "0x..." },
  "latest",
  null,
  null,
  { evalGas: true }
]);
console.log("Gas used:", result.usedGas);
```

### 3. 执行追踪
分析复杂合约调用链：
```javascript
const result = await provider.send("eth_callX", [
  { to: contractAddress, data: "0x..." },
  "latest"
]);
// 检查所有涉及的合约
result.logs.forEach(log => {
  console.log("Contract:", log.address);
  console.log("Event:", log.topics[0]);
});
```

## 实现细节

### 代码位置

1. **返回类型定义**: `crates/rpc/rpc-eth-types/src/call_x.rs`
   - `LogOrRevert`: 返回结构
   - `CallXArgs`: 可选参数

2. **API 定义**: `crates/rpc/rpc-eth-api/src/core.rs`
   - `call_x` 方法签名

3. **实现逻辑**: `crates/rpc/rpc-eth-api/src/helpers/call.rs`
   - `EthCall::call_x` 实现

### 关键实现点

1. **日志收集**: 从 EVM 执行结果中提取日志
2. **错误处理**: 区分 Revert 和 Halt 两种失败情况
3. **性能优化**: 通过 `ignoreLogs` 参数可以跳过日志收集

## 与 Geth 的差异

| 特性 | Geth | Reth |
|------|------|------|
| 日志类型 | `types.Log` | `alloy_rpc_types_eth::Log` |
| 状态字段 | `status` (uint64) | `status` (u64) |
| Gas 字段 | `usedGas` (uint64) | `used_gas` (u64) |
| 区块号 | `blockNumber` (uint64) | `block_number` (u64) |

Reth 版本使用 Alloy 类型系统，但字段名称保持一致以确保 RPC 兼容性。

## 测试

TODO: 添加集成测试

```rust
#[tokio::test]
async fn test_call_x_with_logs() {
    // 测试带日志的 callX
}

#[tokio::test]
async fn test_call_x_revert() {
    // 测试 revert 情况
}
```

## 未来改进

1. 添加更多执行统计信息
2. 支持追踪内部调用
3. 优化大量日志的性能
4. 添加日志过滤功能
