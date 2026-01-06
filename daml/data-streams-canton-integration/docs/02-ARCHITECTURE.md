# Architecture Overview

This document explains how Chainlink Data Streams verification works on Canton, key design decisions, and how it differs from EVM implementations.


## System Overview

The Data Streams Canton Integration enables cryptographic verification of Chainlink oracle reports on the Canton blockchain using Daml smart contracts.


**Key Components:**

1. **Chainlink OCR Network** - Decentralized oracle nodes that generate and sign reports off-chain
2. **Consumer Application** - Your Daml application that needs verified oracle data
3. **Verifier Contract** - Performs cryptographic signature verification
4. **VerifierConfig** - Holds oracle public keys and fault tolerance configuration that is issued by CLL
5. **MasterVerifierConfigFactory** - Admin contract for managing and issuing configurations

**Data Flow:**
- Reports are generated off-chain by Chainlink's OCR protocol
- Reports contain: payload data + f+1 cryptographic signatures from oracle nodes
- Canton contracts verify signatures before allowing data consumption
- Only verified data is returned to consumer applications

## Canton vs EVM: Key Differences

Understanding these differences is crucial for effective integration:

| Aspect | EVM (Solidity) | Canton (Daml) | Integration Impact |
|--------|---------------|---------------|-------------------|
| **Privacy** | Public state visible to all | Party-scoped visibility via observers | Only observers can see verification results. **Must manage observer relationships.** |
| **Execution Model** | Mutable state updates | Immutable contracts, archive & recreate | Contract IDs change on config updates. **Must handle ID updates.** |
| **Authorization** | `msg.sender` checks | Signatory/Observer/Controller model | More granular control. **Use correct authorization patterns.** |
| **Config Updates** | In-place state mutation | Contract archival & recreation | New ContractId on each update. **Track current IDs.** |
| **Transaction Costs** | Per-operation gas fees | Transaction-based pricing | Different cost model. **Consider batch operations.** |
| **Query Model** | View functions (free) | Observer queries only | Must be observer to query. **Admin must grant access.** |

### Key Canton Advantages

**1. Selective Disclosure**
- Only observers see verification transactions and results
- Enables compliance use cases requiring data privacy
- Different parties can verify same report privately

**2. Strong Authorization**
- Daml's signatory model prevents unauthorized access at language level
- No need for runtime access control checks (enforced by ledger)
- Authorization bugs are caught at compile time

**3. Immutable Audit Trail**
- All config changes create new contracts (old ones archived)
- Complete history of all configurations
- No risk of overwriting historical data

### Considerations for Integrators

**1. Observer Management**
- You must be explicitly added as an observer to query VerifierConfig
- Admin controls who can access configurations
- Cannot query configs you're not authorized for

**2. Contract ID Updates**
- Config updates create new VerifierConfig ContractIds
- Must handle ContractId changes in your application

**3. Privacy Implications**
- Verification results are only visible to the verifying party and contract observers
- Multi-party scenarios require careful observer design
- Canton node operators can see contracts they participate in

## Contract Hierarchy

### MasterVerifierConfigFactory

**Role:** Admin contract for centralized configuration management

**Responsibilities:**
- Manage oracle public keys and fault tolerance parameters
- Deploy VerifierConfig contracts for different parties
- Handle configuration lifecycle (create, activate, deactivate)
- Propagate config updates to all managed contracts

**Key Features:**
- Owned by administrator
- Implements `ManagedContractFactory` interface for contract deployment
- Implements `ConfirmedOwner` interface for ownership management
- Implements `Observable` interface for transparency
- Contains the master configuration state

**Authorization:**
- Only owner can set, activate, or deactivate configs
- Only owner can deploy new VerifierConfig contracts
- Observers can see configuration changes

### VerifierConfig

**Role:** Party-specific view of verifier state

**Responsibilities:**
- Hold oracle configurations for a specific party
- Provide read-only access to configuration state
- Enable verification without requiring admin permissions

**Key Features:**
- Owned by factory owner, observed by authorized user
- Contains copy of verifier state from factory
- Automatically updated when factory propagates changes
- Implements `ManagedContract` interface (managed by factory)

**Authorization:**
- Owner (factory admin) is signatory
- User is observer (can query but not modify)
- User can use this config for verification

**Lifecycle:**
- Created by `MasterVerifierConfigFactory.NewContract()`
- Updated by `MasterVerifierConfigFactory.UpdateContract()`
- Archived by `MasterVerifierConfigFactory.RemoveContract()`

### Verifier

**Role:** Stateless verification engine

**Responsibilities:**
- Perform cryptographic signature verification
- Validate f+1 signature threshold
- Return verified report data

**Key Features:**
- Stateless (no configuration stored)
- Requires VerifierConfig ContractId as parameter
- Can be owned by any party
- Can be shared or dedicated

**Authorization:**
- Controller-based: any party with the right ContractIds can verify
- Must be able to fetch the VerifierConfig (via observer relationship)

**Why Separate from Config?**
- Flexibility: Users can create their own Verifier instances
- Reusability: One Verifier can be used with multiple VerifierConfigs
- Simplicity: Verification logic is isolated from configuration management

## Report Verification

**Parse Signed Report**
```
Input: Hex-encoded signed report
Structure: abi.encode(
  bytes32[3] reportContext,
  bytes reportData,
  bytes32[] rs,
  bytes32[] ss,
  bytes32 rawVs
)
```

**Extract and Lookup Config Digest**
- Config digest is in `reportContext[0]` (32 bytes)
- Lookup in `VerifierConfig.verifierConfigStateView.verifierStates`
- Error if not found: `DigestNotSet`

**Validate Config is Active**
- Check `verifierState.isActive == True`
- Error if inactive: `DigestInactive`

**Check Signature Count**
- Expected: `verifierState.f + 1` signatures
- Actual: `length(rs)`
- Error if mismatch: `IncorrectSignatureCount`

**Verify Each Signature**
- Note: Canton does not support public key recovery meaning each signature must be checked against every stored key in Verifier:1.0.0.

**Result:**
- Success: Return `reportData` (verified payload)
- Failure: Error with specific reason

## Configuration Management

### Configuration Lifecycle

### Config Creation and Distribution

**Admin Creates Config**
```daml
-- Admin sets configuration on factory
factoryCid <- submit admin $ exerciseCmd factoryCid SetViewConfig with
  configDigest = "0x1234..." -- 32-byte digest
  signers = [pubKey1, pubKey2, pubKey3, pubKey4]
  f = 1 -- Fault tolerance
```

**Admin Deploys to Users**
```daml
-- Deploy VerifierConfig to specific users
factoryCid <- submit admin $ exerciseCmd (toInterfaceContractId @ManagedContractFactory factoryCid) NewContract with
  users = [user1, user2, user3]
```

**Users Verify Reports**
```daml
-- User can now verify reports
reportData <- submit user1 $ exerciseCmd verifierCid Verify with
  configCid = user1ConfigCid
  signedReportBytes = report
  sender = user1
```

### Config Update Propagation

When admin updates a config, changes must be propagated to all users:


**Important:** User applications must handle the new ContractIds after updates.