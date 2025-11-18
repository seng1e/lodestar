# 调用链详细图示

## 完整调用链: 从 SDK 入口到 Bug 位置

```
┌────────────────────────────────────────────────────────────────────┐
│                         用户应用代码                                  │
│                                                                    │
│  import {stateTransition} from "@lodestar/state-transition";     │
│  const newState = stateTransition(state, signedBlock, options);  │
│                                                                    │
└──────────────────────────┬─────────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────────────┐
│  stateTransition()                                                │
│  📁 packages/state-transition/src/stateTransition.ts:84            │
│                                                                    │
│  • Clone state                                                    │
│  • Process slots (including epoch transitions)                   │
│  • Verify proposer signature                                     │
│  • ⬇️  Call processBlock()                                         │
│  • Commit state changes                                           │
│  • Verify state root                                              │
│                                                                    │
└──────────────────────────┬─────────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────────────┐
│  processBlock()                                                   │
│  📁 packages/state-transition/src/block/index.ts:32                │
│                                                                    │
│  • Process block header                                           │
│  • Process execution payload (if applicable)                      │
│  • Process RANDAO                                                 │
│  • Process ETH1 data                                              │
│  • ⬇️  Call processOperations()                                    │
│  • Process sync aggregate (Altair+)                               │
│  • Process blob KZG commitments (Deneb+)                          │
│                                                                    │
└──────────────────────────┬─────────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────────────┐
│  processOperations()                                              │
│  📁 packages/state-transition/src/block/processOperations.ts:29    │
│                                                                    │
│  for (const proposerSlashing of body.proposerSlashings) {        │
│    ⬇️  Call processProposerSlashing(proposerSlashing)             │
│  }                                                                 │
│                                                                    │
│  for (const attesterSlashing of body.attesterSlashings) {        │
│    processAttesterSlashing(attesterSlashing)                     │
│  }                                                                 │
│                                                                    │
│  • Process attestations                                           │
│  • Process deposits                                               │
│  • Process voluntary exits                                        │
│  • Process BLS to execution changes (Capella+)                    │
│  • Process execution requests (Electra+)                          │
│                                                                    │
└──────────────────────────┬─────────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────────────┐
│  processProposerSlashing()                                        │
│  📁 packages/state-transition/src/block/processProposerSlashing.ts:15│
│                                                                    │
│  • ⬇️  Call assertValidProposerSlashing()                          │
│  • Slash the validator                                            │
│                                                                    │
└──────────────────────────┬─────────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────────────┐
│  assertValidProposerSlashing()                                    │
│  📁 packages/state-transition/src/block/processProposerSlashing.ts:26│
│                                                                    │
│  const header1 = proposerSlashing.signedHeader1.message;         │
│  const header2 = proposerSlashing.signedHeader2.message;         │
│                                                                    │
│  ✓ Check: header1.slot === header2.slot                          │
│  ✓ Check: header1.proposerIndex === header2.proposerIndex        │
│                                                                    │
│  🐛 BUG LOCATION (Line 47):                                       │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ ❌ BEFORE:                                                   │ │
│  │ if (BeaconBlockHeader.equals(header1, header2)) {           │ │
│  │   throw new Error("ProposerSlashing headers are equal");    │ │
│  │ }                                                            │ │
│  │                                                              │ │
│  │ ✅ AFTER:                                                    │ │
│  │ if (ssz.phase0.BeaconBlockHeaderBigint.equals(             │ │
│  │   header1, header2                                          │ │
│  │ )) {                                                         │ │
│  │   throw new Error("ProposerSlashing headers are equal");    │ │
│  │ }                                                            │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
│  ✓ Check: isSlashableValidator(proposer, state.epochCtx.epoch)   │
│  ✓ Check: verifySignatureSet() for both headers                  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

## 数据流

```
SignedBeaconBlock (from network/API)
    ↓
    {
      message: {
        slot: number,
        proposerIndex: number,
        body: {
          proposerSlashings: [
            {
              signedHeader1: {
                message: {
                  slot: bigint,          ← ⚠️  BigInt! 不是 number!
                  proposerIndex: number,
                  parentRoot: Uint8Array,
                  stateRoot: Uint8Array,
                  bodyRoot: Uint8Array,
                },
                signature: Uint8Array
              },
              signedHeader2: {
                message: {
                  slot: bigint,          ← ⚠️  BigInt! 不是 number!
                  proposerIndex: number,
                  parentRoot: Uint8Array,
                  stateRoot: Uint8Array,
                  bodyRoot: Uint8Array,
                },
                signature: Uint8Array
              }
            }
          ],
          // ... other operations ...
        }
      },
      signature: Uint8Array
    }
    ↓
    传递到 assertValidProposerSlashing()
    ↓
    比较 header1 和 header2
    ↓
    ❌ BeaconBlockHeader.equals()    → 期望 slot: number
       但实际数据是 slot: bigint
       → 类型不匹配! 比较结果可能错误
    
    ✅ BeaconBlockHeaderBigint.equals() → 期望 slot: bigint
       实际数据是 slot: bigint
       → 类型匹配! 比较结果正确
```

## 触发场景时序图

```
┌─────────┐   ┌──────────┐   ┌────────────┐   ┌──────────────┐   ┌─────────────┐
│  Client │   │   P2P    │   │   Beacon   │   │    State     │   │   Process   │
│   Node  │   │ Network  │   │    Node    │   │  Transition  │   │   Slashing  │
└────┬────┘   └────┬─────┘   └─────┬──────┘   └──────┬───────┘   └──────┬──────┘
     │             │               │                  │                   │
     │  Produce    │               │                  │                   │
     │  malicious  │               │                  │                   │
     │  slashing   │               │                  │                   │
     │─────────────┼───────────────┼─────────────────>│                   │
     │             │               │                  │                   │
     │             │  Broadcast    │                  │                   │
     │             │  slashing     │                  │                   │
     │             │───────────────>│                  │                   │
     │             │               │                  │                   │
     │             │               │  Include in      │                   │
     │             │               │  block           │                   │
     │             │               │───────────────────>                   │
     │             │               │                  │                   │
     │             │               │  stateTransition()                   │
     │             │               │  processBlock()  │                   │
     │             │               │  processOperations()                 │
     │             │               │──────────────────────────────────────>│
     │             │               │                  │                   │
     │             │               │                  │  assertValid...() │
     │             │               │                  │  ┌──────────────┐ │
     │             │               │                  │  │ 🐛 Bug here! │ │
     │             │               │                  │  │ Wrong type   │ │
     │             │               │                  │  │ comparison   │ │
     │             │               │                  │  └──────────────┘ │
     │             │               │                  │<──────────────────│
     │             │               │                  │                   │
     │             │               │  ❌ May accept   │                   │
     │             │               │  invalid slash   │                   │
     │             │               │  OR reject valid │                   │
     │             │               │<─────────────────                   │
     │             │               │                  │                   │
     │             │  ⚠️  Consensus │                  │                   │
     │             │  divergence!  │                  │                   │
     │             │<──────────────│                  │                   │
```

## 函数调用栈深度

```
Level 0: User Application Code
    ↓
Level 1: stateTransition() [Entry Point]
    ↓
Level 2: processBlock()
    ↓
Level 3: processOperations()
    ↓
Level 4: processProposerSlashing()
    ↓
Level 5: assertValidProposerSlashing() [Bug Location]
    ↓
Level 6: ssz.phase0.BeaconBlockHeaderBigint.equals()
```

**Bug 深度**: 5 层调用栈
**入口点**: `stateTransition()` - 这是用户直接调用的主要 API

## 受影响的代码路径

```
packages/
├── state-transition/         ← 主要受影响的包
│   └── src/
│       ├── stateTransition.ts         (Level 1: 入口)
│       └── block/
│           ├── index.ts               (Level 2: processBlock)
│           ├── processOperations.ts   (Level 3: 操作处理)
│           └── processProposerSlashing.ts  ← 🐛 BUG 在这里
│                                          (Level 4-5)
├── types/                    ← 类型定义
│   └── src/
│       └── phase0/
│           ├── sszTypes.ts            (BeaconBlockHeaderBigint 定义)
│           └── types.ts               (类型导出)
│
└── beacon-node/              ← 使用 state-transition 的节点
    └── src/
        └── chain/
            ├── blocks/
            └── regen/
```

## 关键概念

### BigInt vs Number

```typescript
// 普通区块头 (当前区块)
type BeaconBlockHeader = {
  slot: number,        // 受时钟限制，当前 slot
  proposerIndex: number,
  parentRoot: Bytes32,
  stateRoot: Bytes32,
  bodyRoot: Bytes32,
}

// Slashing 中的区块头 (可能是过去/未来的)
type BeaconBlockHeaderBigint = {
  slot: bigint,        // 不受时钟限制，可以是任意值
  proposerIndex: number,
  parentRoot: Bytes32,
  stateRoot: Bytes32,
  bodyRoot: Bytes32,
}
```

### 为什么需要两种类型？

1. **当前区块** (`BeaconBlockHeader`):
   - Slot 总是接近当前时间
   - 可以安全地使用 `number` 类型
   - 更高效的内存使用

2. **Slashing 引用** (`BeaconBlockHeaderBigint`):
   - Slot 可以是任意历史或未来值
   - 需要 `bigint` 避免溢出
   - 确保长期的准确性

### Bug 的本质

```
使用的类型:  BeaconBlockHeader
实际数据:    BeaconBlockHeaderBigint
结果:        类型不匹配 → 比较逻辑错误
```

这就像用尺子（meter）测量用英里（mile）标记的距离 - 单位不匹配！
