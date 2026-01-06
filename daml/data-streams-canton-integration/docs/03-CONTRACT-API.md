# Contract API Reference

Complete reference documentation for all Data Streams verification contracts on Canton.

## Verifier Contract

### Overview

The `Verifier` contract is a stateless verification engine that validates Chainlink Data Streams reports against configured oracle sets. It performs cryptographic signature verification to ensure report authenticity.

**Contract Version:** `Verifier:1.0.0`

### Template Definition

```daml
template Verifier
  with
    owner : Party
    observers : [Party]
  where
    signatory owner
    observer observers
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `owner` | `Party` | The party that owns this Verifier instance |
| `observers` | `[Party]` | List of parties that can observe this contract |

### Choices

#### Verify

Verifies a signed Data Streams report using oracle configuration from a VerifierConfig contract.

**Signature:**
```daml
nonconsuming choice Verify : BytesHex
  with
    configCid : ContractId VerifierConfig
    signedReportBytes : BytesHex
    sender : Party
  controller sender
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `configCid` | `ContractId VerifierConfig` | Reference to the VerifierConfig containing oracle keys and fault tolerance |
| `signedReportBytes` | `BytesHex` | Hex-encoded signed report (with or without `0x` prefix) |
| `sender` | `Party` | The party requesting verification (must be the controller) |

**Returns:** `BytesHex` - The verified report payload data

**Authorization:** `controller sender` - Any party can call this choice if they have the required ContractIds

**Behavior:**

1. **Fetch Configuration** - Retrieves the VerifierConfig to get oracle public keys and `f` value
2. **Parse Report** - Decodes the hex-encoded report structure:
   ```
   abi.encode(
     bytes32[3] reportContext,
     bytes reportData,
     bytes32[] rs,
     bytes32[] ss,
     bytes32 rawVs
   )
   ```
3. **Extract Config Digest** - Gets config digest from `reportContext[0]`
4. **Lookup Config** - Finds the verifier state for this digest
5. **Check Active Status** - Verifies the config is active
6. **Validate Signature Count** - Ensures exactly `f + 1` signatures are present
7. **Verify Signatures** - For each `(r, s)` signature:
   - Computes message hash: `keccak256(keccak256(reportData) || reportContext)`
   - Validates ECDSA signature against each oracle public key
   - Ensures no duplicate signers (each oracle signs at most once)

    NOTE: Daml does not currently support key recovery and so each public key is verified againt each signature.
8. **Threshold Check** - Confirms exactly `f + 1` unique valid signatures
9. **Return Data** - Returns the verified `reportData`

**Error Conditions:**

| Error | Condition | Description |
|-------|-----------|-------------|
| `DigestNotSet: <digest>` | Config digest not found | The config digest from the report doesn't exist in the VerifierConfig |
| `DigestInactive: <digest>` | Config is deactivated | The config exists but has been deactivated by admin |
| `IncorrectSignatureCount: <actual> signatures, expected <expected>` | Wrong number of signatures | Report has != `f+1` signatures |
| `MismatchedSignatures: rs length X != ss length Y` | Malformed signature data | The `rs` and `ss` arrays have different lengths |
| `BadVerification: signature did not match any oracle` | Invalid signature | A signature doesn't match any configured oracle public key |
| `BadVerification: insufficient unique signers` | Duplicate signers | An oracle signed more than once (duplicate detected) |

**Integration Example:**

```daml
-- Verify a Data Streams report
verifiedData <- submit myParty $ exerciseCmd verifierCid Verify with
  configCid = myConfigCid
  signedReportBytes = "0x1234..." -- Your hex-encoded report
  sender = myParty

-- Use the verified data
debug ("Verified report data: " <> verifiedData)
```

**Notes:**
- The `Verify` choice is non-consuming, so the Verifier contract persists after verification
- The hex-encoded report can include or omit the `0x` prefix
- Verification is deterministic - same inputs always produce same output

## VerifierConfig Contract

### Overview

The `VerifierConfig` contract holds oracle configuration and state for a specific party. It's created and managed by the `MasterVerifierConfigFactory` and provides read-only access to verifier state.

**Contract Version:** `VerifierConfig:1.0.0`

### Template Definition

```daml
template VerifierConfig
  with
    confirmedOwnerView : ConfirmedOwnerView
    observableView : ObservableView
    verifierConfigStateView : VerifierConfigStateView
  where
    signatory confirmedOwnerView.owner
    observer observableView.observers
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `confirmedOwnerView` | `ConfirmedOwnerView` | Ownership information (owner and pending owner) |
| `observableView` | `ObservableView` | List of observers who can query this contract |
| `verifierConfigStateView` | `VerifierConfigStateView` | Oracle configurations and state |

**Key Data Structures:**

**VerifierConfigStateView:**
```daml
data VerifierConfigStateView = VerifierConfigStateView
  with
    verifierStates : Map BytesHex VerifierConfigDigest
```

**VerifierConfigDigest:**
```daml
data VerifierConfigDigest = VerifierConfigDigest
  with
    latestConfigTime : Time          -- When config was last updated
    isActive : Bool                   -- Whether config is active
    f : Int                           -- Fault tolerance parameter
    oracles : Map BytesHex Int        -- Oracle public keys
```

### Choices

This contract primarily implements interface choices:

- **ConfirmedOwner interface** - Ownership management
- **Observable interface** - Observer management
- **ManagedContract interface** - Lifecycle management by factory
- **TypeAndVersion interface** - Contract versioning

Direct choices are managed by the `MasterVerifierConfigFactory`.

### Authorization

- **Signatory:** Factory owner (admin)
- **Observers:** Authorized users who can query and use the config
- **Note:** You must be an observer to fetch and use this contract

### Lifecycle

**Created by:**
```daml
-- Factory creates config for user
factoryCid <- submit admin $ exerciseCmd (toInterfaceContractId @ManagedContractFactory factoryCid) NewContract with
  users = [user]
```

**Updated by:**
```daml
-- Factory propagates config updates to users
factoryCid <- submit admin $ exerciseCmd (toInterfaceContractId @ManagedContractFactory factoryCid) UpdateContract with
  users = [user]
```

**Removed by:**
```daml
-- Factory archives user config
factoryCid <- submit admin $ exerciseCmd (toInterfaceContractId @ManagedContractFactory factoryCid) RemoveContract with
  users = [user]
```

**Important:** When updated, the ContractId changes. Your application must handle the new ContractId.

## MasterVerifierConfigFactory Contract

### Overview

The `MasterVerifierConfigFactory` contract is the admin contract for managing verifier configurations across multiple parties. It handles configuration lifecycle and deploys `VerifierConfig` contracts to users.

**Contract Version:** `MasterVerifierConfigFactory:1.0.0`

### Template Definition

```daml
template MasterVerifierConfigFactory
  with
    confirmedOwnerView : ConfirmedOwnerView
    observableView : ObservableView
    managedContractFactoryView : ManagedContractFactoryView
    verifierConfigStateView : VerifierConfigStateView
  where
    signatory confirmedOwnerView.owner
    observer observableView.observers
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `confirmedOwnerView` | `ConfirmedOwnerView` | Ownership information |
| `observableView` | `ObservableView` | Observers for transparency |
| `managedContractFactoryView` | `ManagedContractFactoryView` | Managed contracts map (user -> ContractId) |
| `verifierConfigStateView` | `VerifierConfigStateView` | Master configuration state |

### Choices

#### SetViewConfig

Creates or updates a verifier configuration.

**Signature:**
```daml
choice SetViewConfig : ContractId MasterVerifierConfigFactory
  with
    configDigest : BytesHex
    signers : [BytesHex]
    f : Int
  controller confirmedOwnerView.owner
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `configDigest` | `BytesHex` | 32-byte config digest (hex, with or without `0x`) |
| `signers` | `[BytesHex]` | List of 65-byte uncompressed ECDSA public keys (hex) |
| `f` | `Int` | Fault tolerance (minimum 1, must satisfy: `length(signers) > 3*f`) |

**Returns:** `ContractId MasterVerifierConfigFactory` - New factory ContractId

**Authorization:** `controller confirmedOwnerView.owner` - Only factory owner (admin)

**Validations:**

- Config digest must be exactly 32 bytes
- `1 <= f <= 10`
- Number of signers must be `> 3*f` (Byzantine Fault Tolerance requirement)
- Maximum 31 signers (OCR protocol limit: `maxNumOracles = 31`)
- All signers must be 65-byte uncompressed public keys (starting with `04`)
- No duplicate signers
- No zero addresses (all zeros)
- Config is set to **active by default**

**Behavior:**
1. Validates all input parameters
2. Creates oracle map from signer list
3. Checks for duplicates
4. Creates `VerifierConfigDigest` with `isActive = True`
5. Inserts or updates config in `verifierStates` map
6. Archives current factory and creates new one with updated state

**Example:**

```daml
-- Set configuration for a new oracle set
newFactoryCid <- submit admin $ exerciseCmd factoryCid SetViewConfig with
  configDigest = "0x0006f9b553e393ced311551efd30d1decedb63d76ad41737462e2cdbbdff1578"
  signers = 
    [ "0411111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111"
    , "0422222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222"
    , "0433333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333"
    , "0444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444"
    ]
  f = 1

-- Factory ContractId has changed - use newFactoryCid for future operations
```

**Error Conditions:**

| Error | Condition |
|-------|-----------|
| `InvalidConfigDigestLength` | Config digest is not 32 bytes |
| `ExcessSigners: <count> > 31` | More than 31 signers provided |
| `InsufficientSigners: <count> <= <3*f>` | Not enough signers for fault tolerance |
| `ZeroPublicKey` | One or more public keys are all zeros |
| `InvalidPublicKeyLength` | Public key is not 65 bytes |
| `FaultToleranceMustBePositive` | `f <= 0` |
| `NonUniqueSignatures` | Duplicate signers in the list |

#### ActivateViewConfig

Activates a previously deactivated configuration.

**Signature:**
```daml
choice ActivateViewConfig : ContractId MasterVerifierConfigFactory
  with
    configDigest : BytesHex
  controller confirmedOwnerView.owner
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `configDigest` | `BytesHex` | The config digest to activate |

**Returns:** `ContractId MasterVerifierConfigFactory` - New factory ContractId

**Authorization:** `controller confirmedOwnerView.owner` - Only factory owner (admin)

**Behavior:**
1. Looks up the config digest in `verifierStates`
2. Sets `isActive = True`
3. Archives current factory and creates new one with updated state

**Example:**

```daml
-- Activate a configuration
newFactoryCid <- submit admin $ exerciseCmd factoryCid ActivateViewConfig with
  configDigest = "0x0006f9b..."
```

**Error Conditions:**

| Error | Condition |
|-------|-----------|
| `DigestNotSet: <digest>` | Config digest doesn't exist |

**Notes:**
- Idempotent: Activating an already-active config succeeds
- After activation, the config can be used for verification

#### DeactivateViewConfig

Deactivates a configuration (prevents its use for verification).

**Signature:**
```daml
choice DeactivateViewConfig : ContractId MasterVerifierConfigFactory
  with
    configDigest : BytesHex
  controller confirmedOwnerView.owner
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `configDigest` | `BytesHex` | The config digest to deactivate |

**Returns:** `ContractId MasterVerifierConfigFactory` - New factory ContractId

**Authorization:** `controller confirmedOwnerView.owner` - Only factory owner (admin)

**Behavior:**
1. Looks up the config digest in `verifierStates`
2. Sets `isActive = False`
3. Archives current factory and creates new one with updated state

**Example:**

```daml
-- Deactivate a configuration
newFactoryCid <- submit admin $ exerciseCmd factoryCid DeactivateViewConfig with
  configDigest = "0x0006f9b..."
```

**Error Conditions:**

| Error | Condition |
|-------|-----------|
| `DigestNotSet: <digest>` | Config digest doesn't exist |

**Notes:**
- Idempotent: Deactivating an already-inactive config succeeds
- Verification attempts with this digest will fail with `DigestInactive` error

### Interface Choices

#### NewContract (via ManagedContractFactory)

Deploys new VerifierConfig contracts for specified users.

**Signature:**
```daml
-- Via ManagedContractFactory interface
exerciseCmd (toInterfaceContractId @ManagedContractFactory factoryCid) NewContract with
  users : [Party]
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `users` | `[Party]` | List of parties to deploy configs for |

**Returns:** `ContractId ManagedContractFactory` - New factory ContractId (as interface)

**Behavior:**
- For each user:
  1. Creates `VerifierConfig` with factory owner as signatory
  2. Adds user as observer
  3. Copies current `verifierConfigStateView` from factory
  4. Stores ContractId in `managedContractFactoryView.contracts` map

**Example:**

```daml
-- Deploy configs to multiple users
newFactoryCid <- submit admin $ exerciseCmd (toInterfaceContractId @ManagedContractFactory factoryCid) NewContract with
  users = [user1, user2, user3]

-- Each user now has their own VerifierConfig
```

**Notes:**
- Users become observers of their VerifierConfig
- Each user gets a separate contract for privacy
- Factory tracks all deployed configs in its `contracts` map

#### UpdateContract (via ManagedContractFactory)

Propagates factory config updates to user contracts.

**Signature:**
```daml
-- Via ManagedContractFactory interface
exerciseCmd (toInterfaceContractId @ManagedContractFactory factoryCid) UpdateContract with
  users : [Party]
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `users` | `[Party]` | List of parties whose configs should be updated |

**Returns:** `ContractId ManagedContractFactory` - New factory ContractId (as interface)

**Behavior:**
- For each user:
  1. Archives old VerifierConfig
  2. Creates new VerifierConfig with updated `verifierConfigStateView`
  3. Updates ContractId in factory's `contracts` map

**Example:**

```daml
-- Update factory config first
factoryCid1 <- submit admin $ exerciseCmd factoryCid SetViewConfig with
  configDigest = newDigest
  signers = newSigners
  f = newF

-- Propagate to users
factoryCid2 <- submit admin $ exerciseCmd (toInterfaceContractId @ManagedContractFactory factoryCid1) UpdateContract with
  users = [user1, user2, user3]

-- All user contracts now have new config
```

**Important:** User VerifierConfig ContractIds change after update. Users must obtain new ContractIds.

#### RemoveContract (via ManagedContractFactory)

Removes and archives user VerifierConfig contracts.

**Signature:**
```daml
-- Via ManagedContractFactory interface
exerciseCmd (toInterfaceContractId @ManagedContractFactory factoryCid) RemoveContract with
  users : [Party]
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `users` | `[Party]` | List of parties whose configs should be removed |

**Returns:** `ContractId ManagedContractFactory` - New factory ContractId (as interface)

**Behavior:**
- For each user:
  1. Archives user's VerifierConfig
  2. Removes user from factory's `contracts` map

**Example:**

```daml
-- Remove user configs
newFactoryCid <- submit admin $ exerciseCmd (toInterfaceContractId @ManagedContractFactory factoryCid) RemoveContract with
  users = [user1]

-- user1's VerifierConfig is now archived
```

## Data Structures

### ConfirmedOwnerView

```daml
data ConfirmedOwnerView = ConfirmedOwnerView
  with
    owner : Party                -- Current owner
    pendingOwner : Optional Party -- Pending owner (during transfer)
```

Used for ownership management via the `ConfirmedOwner` interface.

### ObservableView

```daml
data ObservableView = ObservableView
  with
    observers : [Party]  -- List of observer parties
```

Used for observer management via the `Observable` interface.

### ManagedContractFactoryView

```daml
data ManagedContractFactoryView = ManagedContractFactoryView
  with
    contracts : Map Party (ContractId ManagedContract)
```

Maps users to their VerifierConfig ContractIds.

### VerifierConfigStateView

```daml
data VerifierConfigStateView = VerifierConfigStateView
  with
    verifierStates : Map BytesHex VerifierConfigDigest
```

Maps config digests to their configurations.

### VerifierConfigDigest

```daml
data VerifierConfigDigest = VerifierConfigDigest
  with
    latestConfigTime : Time           -- Last update timestamp
    isActive : Bool                    -- Active status
    f : Int                            -- Fault tolerance
    oracles : Map BytesHex Int         -- Oracle public key -> index
```

Configuration details for a specific config digest.

## Interface Implementations

### ConfirmedOwner Interface

Implemented by: `MasterVerifierConfigFactory`, `VerifierConfig`

**Purpose:** Ownership management with two-step transfer

**Choices:**
- `TransferOwnership` - Initiates ownership transfer
- `AcceptOwnership` - New owner accepts transfer

### Observable Interface

Implemented by: `MasterVerifierConfigFactory`, `VerifierConfig`, `Verifier`

**Purpose:** Observer management for transparency

**Choices:**
- `AddObserver` - Add a new observer
- `RemoveObserver` - Remove an observer

### ManagedContractFactory Interface

Implemented by: `MasterVerifierConfigFactory`

**Purpose:** Lifecycle management of child contracts

**Choices:**
- `NewContract` - Deploy new managed contracts
- `UpdateContract` - Update managed contracts
- `RemoveContract` - Remove managed contracts

### ManagedContract Interface

Implemented by: `VerifierConfig`

**Purpose:** Marks contracts as manageable by factories

### TypeAndVersion Interface

Implemented by: All contracts

**Purpose:** Contract versioning for upgrades

**Choice:**
- `GetTypeAndVersion` - Returns contract type and version string

## Quick Reference

### Common Operations

**Deploy Config to User:**
```daml
factoryCid <- submit admin $ exerciseCmd (toInterfaceContractId @ManagedContractFactory factoryCid) NewContract with
  users = [user]
```

**Verify Report:**
```daml
reportData <- submit user $ exerciseCmd verifierCid Verify with
  configCid = userConfigCid
  signedReportBytes = report
  sender = user
```

**Update Config:**
```daml
-- 1. Update factory
factoryCid1 <- submit admin $ exerciseCmd factoryCid SetViewConfig with ...
-- 2. Propagate to users
factoryCid2 <- submit admin $ exerciseCmd (toInterfaceContractId @ManagedContractFactory factoryCid1) UpdateContract with
  users = [user1, user2]
```

**Activate/Deactivate Config:**
```daml
-- Activate
factoryCid <- submit admin $ exerciseCmd factoryCid ActivateViewConfig with
  configDigest = digest

-- Deactivate
factoryCid <- submit admin $ exerciseCmd factoryCid DeactivateViewConfig with
  configDigest = digest
```
