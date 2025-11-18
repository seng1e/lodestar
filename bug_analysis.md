# Bug Analysis: ProposerSlashing Type Comparison Issue

## Commit Information
- **Commit**: fd5870fcfed12f259053ec56cd65bbc29ed4e70e
- **PR**: https://github.com/ChainSafe/lodestar/pull/3977
- **Title**: "Optimize uint"
- **Date**: 2022-05-05

## Bug Description

### The Problem
The bug was in the `assertValidProposerSlashing` function in `processProposerSlashing.ts`. The code was using the wrong SSZ type (`BeaconBlockHeader`) to compare two beacon block headers when it should have been using `BeaconBlockHeaderBigint`.

### Root Cause
`ProposerSlashing` contains two `SignedBeaconBlockHeaderBigint` objects where the `slot` field is a `bigint` (not bounded by the clock). However, the comparison was using `BeaconBlockHeader.equals()` which expects regular `number` types for the slot field, leading to incorrect equality checks.

**Before (buggy code):**
```typescript
if (BeaconBlockHeader.equals(header1, header2)) {
  throw new Error("ProposerSlashing headers are equal");
}
```

**After (fixed code):**
```typescript
if (ssz.phase0.BeaconBlockHeaderBigint.equals(header1, header2)) {
  throw new Error("ProposerSlashing headers are equal");
}
```

### Why This Matters
When comparing two beacon block headers for a proposer slashing:
- If both headers were actually **equal** (same data), the buggy code might fail to detect this due to type mismatch
- If both headers were **different**, the comparison could produce false positives
- This could lead to either rejecting valid slashings or accepting invalid ones

## Affected Function

### Primary Function
**Function**: `assertValidProposerSlashing()`
**Location**: `packages/state-transition/src/block/processProposerSlashing.ts`

This function validates a `ProposerSlashing` operation by checking:
1. Headers have matching slots
2. Headers have matching proposer indices  
3. Headers are different (this is where the bug was)
4. The proposer is slashable
5. Signatures are valid

## Call Chain from SDK Entry Point

Here's how downstream users would trigger this code:

```
User Code
    ↓
stateTransition() 
  └─ packages/state-transition/src/stateTransition.ts
    ↓
processBlock()
  └─ packages/state-transition/src/block/index.ts
    ↓
processOperations()
  └─ packages/state-transition/src/block/processOperations.ts
    ↓
processProposerSlashing() [loops through body.proposerSlashings]
  └─ packages/state-transition/src/block/processProposerSlashing.ts
    ↓
assertValidProposerSlashing() ← BUG WAS HERE
  └─ Uses wrong SSZ type for header comparison
```

## How Downstream Users Trigger This Bug

Any downstream user of the `@lodestar/state-transition` package who processes blocks containing ProposerSlashing operations would hit this bug. This includes:

1. **Beacon Node Operators** - When their node processes blocks with proposer slashings
2. **Block Producers** - When validating blocks before proposal
3. **State Transition Testing** - When running state transition tests with slashing scenarios
4. **Custom Ethereum Clients** - Any client using Lodestar's state transition library

## Demo: How to Trigger the Bug

### Setup
```bash
npm install @lodestar/state-transition @lodestar/types @lodestar/params
```

### Code Example

```typescript
import {stateTransition} from "@lodestar/state-transition";
import {ssz, phase0} from "@lodestar/types";
import {DOMAIN_BEACON_PROPOSER} from "@lodestar/params";
import {config} from "@lodestar/config/default";

// This example shows how a downstream user would trigger the bug
// when processing a block containing a ProposerSlashing

async function demonstrateBug() {
  // Assume we have a valid beacon state
  const state = /* ... your cached beacon state ... */;
  
  // Create two identical headers (should be detected as equal)
  // This is an INVALID proposer slashing because headers must be different
  const header1 = {
    slot: 1_800_000n, // BigInt slot (higher than current clock)
    proposerIndex: 12345,
    parentRoot: new Uint8Array(32).fill(0xaa),
    stateRoot: new Uint8Array(32).fill(0xbb),
    bodyRoot: new Uint8Array(32).fill(0xcc),
  };
  
  // Create identical header (should trigger "headers are equal" error)
  const header2 = {...header1};
  
  const proposerSlashing: phase0.ProposerSlashing = {
    signedHeader1: {
      message: header1,
      signature: new Uint8Array(96).fill(0), // dummy signature
    },
    signedHeader2: {
      message: header2,
      signature: new Uint8Array(96).fill(0), // dummy signature
    },
  };
  
  // Create a block with this proposer slashing
  const block: phase0.SignedBeaconBlock = {
    message: {
      slot: 1_800_001,
      proposerIndex: 67890,
      parentRoot: new Uint8Array(32),
      stateRoot: new Uint8Array(32),
      body: {
        randaoReveal: new Uint8Array(96),
        eth1Data: {
          depositRoot: new Uint8Array(32),
          depositCount: 0,
          blockHash: new Uint8Array(32),
        },
        graffiti: new Uint8Array(32),
        proposerSlashings: [proposerSlashing], // ← Bug triggered here
        attesterSlashings: [],
        attestations: [],
        deposits: [],
        voluntaryExits: [],
      },
    },
    signature: new Uint8Array(96),
  };
  
  try {
    // This should fail with "ProposerSlashing headers are equal"
    // BUT with the bug, it might not detect the headers are equal
    // because BeaconBlockHeader.equals() expects number slots, not bigint
    const newState = stateTransition(state, block, {
      verifySignatures: false, // Skip signature verification for demo
      verifyStateRoot: false,  // Skip state root verification for demo
    });
    
    console.log("Bug: Invalid slashing was accepted!");
  } catch (error) {
    console.log("Correct behavior: Invalid slashing rejected", error.message);
  }
}
```

### What Happens

**With the bug (before fix):**
- `BeaconBlockHeader.equals(header1, header2)` compares headers with mismatched types
- The slot field is `bigint` but the SSZ type expects `number`
- The comparison may fail to properly detect equality
- Invalid proposer slashing might be accepted

**After the fix:**
- `BeaconBlockHeaderBigint.equals(header1, header2)` correctly handles bigint slots
- Equal headers are properly detected
- Error is thrown: "ProposerSlashing headers are equal"
- Invalid slashing is correctly rejected

## Impact

### Severity: HIGH
- **Security Impact**: Could allow invalid slashings to be processed or reject valid ones
- **Network Impact**: Consensus issues if different clients handle this differently
- **Affected Versions**: All versions before the fix in PR #3977

### Who Is Affected
1. Beacon node operators running Lodestar
2. Applications using `@lodestar/state-transition` package
3. Any custom Ethereum consensus clients built on Lodestar libraries

## Related Files Changed in the Fix

The fix touched multiple files to ensure consistency:

1. `packages/state-transition/src/block/processProposerSlashing.ts` - Main fix
2. `packages/state-transition/src/block/processAttesterSlashing.ts` - Similar fix for attester slashings
3. `packages/types/src/phase0/sszTypes.ts` - Type definitions (BeaconBlockHeaderBigint)
4. Multiple test files - Updated to use correct types
5. Signature set files - Updated to use bigint variants

## Recommendations

If you're using an older version of Lodestar:
1. **Upgrade immediately** to a version containing this fix
2. **Review any blocks** processed with proposer slashings since deployment
3. **Test your integration** with the demo code above to ensure correct behavior
