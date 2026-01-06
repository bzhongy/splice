# Getting Started: Verify Your First Data Streams Report

This guide will help you verify your first Chainlink Data Streams report on Canton.

## Overview

Chainlink Data Streams provides high-frequency, cryptographically signed oracle reports. This integration allows you to verify these reports on the Canton blockchain using Daml smart contracts, ensuring that the data you consume is authentic and has been signed by a threshold of trusted oracle nodes.

For more background on Data Streams, see the [official Chainlink Data Streams documentation](https://docs.chain.link/data-streams).


## Integration

### Import the Verifier Contract

In your Daml module, import the Verifier contract:

```daml
import Verifier
import DA.Crypto.Text (BytesHex)
```

### Get Your VerifierConfig Contract ID

Your VerifierConfig contract is managed by CLL. You need the ContractId to verify reports.

```daml
-- Your VerifierConfig ContractId (as provided)
-- This contains the oracle public keys and fault tolerance (f) for verification
let configCid : ContractId VerifierConfig = ...
```

**Note:** You must be an observer of the VerifierConfig contract to use it for verification, which again is provided by CLL.

### Step 3: Create or Reference a Verifier Instance

The Verifier contract is stateless - you can create your own or share one:

```daml
-- Option A: Create your own Verifier
verifierCid <- submit myParty $ createCmd Verifier with
  owner = myParty
  observers = []

-- Option B: Use a shared Verifier (if one exists)
-- let verifierCid = ... (get from somewhere)
```

### Step 4: Verify a Report

Call the `Verify` choice with your signed report:

```daml
-- Your hex-encoded signed report from Chainlink Data Streams
let signedReport : BytesHex = "0x..." -- Can include 0x prefix or not

-- Verify the report
reportData <- submit myParty $ exerciseCmd verifierCid Verify with
  configCid = configCid
  signedReportBytes = signedReport
  sender = myParty
```

**Success!** If verification succeeds, `reportData` contains the verified report payload.

### Step 5: Use the Verified Data

The `reportData` is the hex-encoded report payload. You can now parse and use it in your application. Schema types are available within the [SDK](https://github.com/smartcontractkit/data-streams-sdk/tree/main/go/report)

## What Happens During Verification?

When you call `Verify`, the contract performs these checks:

1. **Report Parsing** - Decodes the hex-encoded report structure
2. **Config Digest Extraction** - Reads the config digest from the report context
3. **Config Validation** - Ensures the config exists and is active
4. **Signature Count Check** - Verifies exactly `f+1` signatures are present
5. **ECDSA Verification** - Validates each signature against configured oracle public keys
6. **Uniqueness Check** - Ensures no oracle has signed twice

**Byzantine Fault Tolerance:** The `f+1` requirement means the system can tolerate up to `f` malicious or faulty oracle nodes. With `f+1` signatures, at least one signature is guaranteed to be from an honest node.

## Tips for Production

1. **Handle Config Updates** - VerifierConfig ContractIds change when configs are updated. These changes are usually infrequent.

2. **Error Handling** - Always handle verification errors gracefully. 

3. **Report Integrity** - Check report timestamps to ensure data isn't stale along with the feed id to ennsure it's the expected report.

4. **Observer Management** - Ensure you're added as an observer by CLL