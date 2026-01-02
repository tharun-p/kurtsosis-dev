# Ethereum Staking Deposit POC with Kurtosis

A proof-of-concept project for running a local Ethereum network using Kurtosis and testing validator deposits.

---

## Prerequisites

- [Kurtosis CLI](https://docs.kurtosis.com/install/) installed
- [Foundry (cast)](https://book.getfoundry.sh/getting-started/installation) installed
- [eth2-val-tools](https://github.com/protolambda/eth2-val-tools) installed
- Docker running

### API Reference

- [Ethereum Beacon APIs](https://ethereum.github.io/beacon-APIs/) - Official specification for interacting with the Consensus Layer

---

## Quick Start

### 1. Start the Kurtosis Ethereum Network

```bash
kurtosis run github.com/ethpandaops/ethereum-package --args-file ./network_params.yaml
```

This spins up a local Ethereum network with:
- **Execution Layer**: Geth
- **Consensus Layer**: Lighthouse
- **Validators**: 128 pre-configured validators

### 2. Network Configuration

The network is configured via `network_params.yaml`:

```yaml
participants:
  - el_type: geth
    el_image: ethereum/client-go:latest
    cl_type: lighthouse
    cl_image: sigp/lighthouse:latest
    count: 1
    validator_count: 128

network_params:
  network_id: "3151908"
  deposit_contract_address: "0x4242424242424242424242424242424242424242"
  seconds_per_slot: 12
```

---

## Service Endpoints (from startup_log)

After startup, the following services are available:

| Service | Port | URL |
|---------|------|-----|
| EL RPC (Geth) | 8545 | `http://127.0.0.1:64198` |
| EL WebSocket | 8546 | `ws://127.0.0.1:64199` |
| EL Metrics | 9001 | `http://127.0.0.1:64196` |
| CL HTTP (Lighthouse) | 4000 | `http://127.0.0.1:64204` |
| CL Metrics | 5054 | `http://127.0.0.1:64203` |
| Validator Client Metrics | 8080 | `http://127.0.0.1:64205` |

> **Note**: Ports are dynamically assigned. Check your `startup_log` for actual values.

---

## Making a Validator Deposit

### Step 1: Generate Deposit Data

Use `eth2-val-tools` to generate a new validator mnemonic and deposit data:

```bash
# Generate a new mnemonic
eth2-val-tools mnemonic
```

Then generate deposit data for your validator:

```bash
eth2-val-tools deposit-data \
  --fork-version 0x10000038 \
  --source-min=64 \
  --source-max=65 \
  --as-json-list \
  --validators-mnemonic "your-24-word-mnemonic-here" \
  --withdrawal-credentials-type=0x02 \
  --withdrawal-address=0xafF0CA253b97e54440965855cec0A8a2E2399896
```
> **Tip**: You can retrieve the `fork_version` value by querying your beacon client at `{{baseUrl}}/eth/v1/beacon/genesis`. The response will include the `genesis_fork_version` field.


**Output Example:**
```json
[{
  "account": "m/12381/3600/64/0/0",
  "deposit_data_root": "f0b6b9c086e80830337dffcef63c3237ab6ce46fabc668e3d83fa5e95cafd476",
  "pubkey": "97ba76e13476722ab6f79bab5e797ad25d778eab6ca720202b99dfb694b96c65a58b28f1bf4fba6a618f55d68b886e1f",
  "signature": "a26248eeb38b314f328a7f9fc6c8bb99f29e92a5a2e2d0839fc9293cf54df2447d5e1a9291586ad19d29914123a9a91a1847c617df44ae0fb7228ff418e65e92aecc212d0d8886d25a26497cc8606d29c1d97b2a0a6e8d702acdd1273baed17c",
  "value": 32000000000,
  "version": 1,
  "withdrawal_credentials": "020000000000000000000000aff0ca253b97e54440965855cec0a8a2e2399896"
}]
```

### Step 2: Check Account Balance

```bash
cast balance --ether --rpc-url http://0.0.0.0:64198 0xafF0CA253b97e54440965855cec0A8a2E2399896
```
> **Note:** The `cast` commands above use a pre-funded validator address and private key (`0xafF0CA253b97e54440965855cec0A8a2E2399896`), which you will find listed in your `startup_log`. These addresses and keys are for local test usage only and will be **visible in your log output.


### Step 3: Send Deposit Transaction

Use `cast send` to submit the 32 ETH deposit to the deposit contract:

```bash
cast send --rpc-url http://0.0.0.0:64198 \
  --private-key 04b9f63ecf84210c5366c66d68fa1f5da1fa4f634fad6dfc86178e4d79ff9e59 \
  --value 32ether \
  0x4242424242424242424242424242424242424242 \
  "deposit(bytes,bytes,bytes,bytes32)" \
  <PUBKEY> \
  <WITHDRAWAL_CREDENTIALS> \
  <SIGNATURE> \
  <DEPOSIT_DATA_ROOT>
```

**Full Example:**
```bash
cast send --rpc-url http://0.0.0.0:64198 \
  --private-key 04b9f63ecf84210c5366c66d68fa1f5da1fa4f634fad6dfc86178e4d79ff9e59 \
  --value 32ether \
  0x4242424242424242424242424242424242424242 \
  "deposit(bytes,bytes,bytes,bytes32)" \
  97ba76e13476722ab6f79bab5e797ad25d778eab6ca720202b99dfb694b96c65a58b28f1bf4fba6a618f55d68b886e1f \
  020000000000000000000000aff0ca253b97e54440965855cec0a8a2e2399896 \
  a26248eeb38b314f328a7f9fc6c8bb99f29e92a5a2e2d0839fc9293cf54df2447d5e1a9291586ad19d29914123a9a91a1847c617df44ae0fb7228ff418e65e92aecc212d0d8886d25a26497cc8606d29c1d97b2a0a6e8d702acdd1273baed17c \
  f0b6b9c086e80830337dffcef63c3237ab6ce46fabc668e3d83fa5e95cafd476
```

### Step 4: Verify Deposit in Pending Queue

Query the beacon chain to check if the deposit is in the pending queue:

```bash
curl http://127.0.0.1:4000/eth/v1/beacon/states/head/validators | grep "pending"
```

**Expected Response (when deposit is pending):**
```json
{
  "version": "fulu",
  "execution_optimistic": false,
  "finalized": false,
  "data": [
    {
      "pubkey": "0x97ba76e13476722ab6f79bab5e797ad25d778eab6ca720202b99dfb694b96c65a58b28f1bf4fba6a618f55d68b886e1f",
      "withdrawal_credentials": "0x020000000000000000000000aff0ca253b97e54440965855cec0a8a2e2399896",
      "amount": "32000000000",
      "signature": "0xa26248eeb38b314f328a7f9fc6c8bb99f29e92a5a2e2d0839fc9293cf54df2447d5e1a9291586ad19d29914123a9a91a1847c617df44ae0fb7228ff418e65e92aecc212d0d8886d25a26497cc8606d29c1d97b2a0a6e8d702acdd1273baed17c",
      "slot": "447"
    }
  ]
}
```

---

## Pre-funded Accounts

The network comes with pre-funded accounts for testing. Here are a few:

| Address | Private Key |
|---------|-------------|
| `0x8943545177806ED17B9F23F0a21ee5948eCaa776` | `bcdf20249abf0ed6d944c0288fad489e33f66b3960d9e6229c1cd214ed3bbe31` |
| `0xafF0CA253b97e54440965855cec0A8a2E2399896` | `04b9f63ecf84210c5366c66d68fa1f5da1fa4f634fad6dfc86178e4d79ff9e59` |
| `0xE25583099BA105D9ec0A67f5Ae86D90e50036425` | `39725efee3fb28614de3bacaffe4cc4bd8c436257e2c8bb887c4b5c4be45e76d` |

> See `startup_log` for the complete list of 21 pre-funded accounts.

---

## Important Addresses

| Contract | Address |
|----------|---------|
| Deposit Contract | `0x4242424242424242424242424242424242424242` |
| Default Withdrawal Address | `0x8943545177806ED17B9F23F0a21ee5948eCaa776` |

---

## Network Parameters

| Parameter | Value |
|-----------|-------|
| Network ID | `3151908` |
| Seconds per Slot | `12` |
| Fork Version | `0x10000038` |
| Genesis Validators Root | `0xd1ec305b97bf6336571c2348e4a8bf173684b0cdb7e55f7e6554d51f8478b5a3` |
| Enclave Name | `billowing-caldera` |

---

## File Structure

```
.
├── readme.md              # This file
├── network_params.yaml    # Kurtosis network configuration
├── cast-utils             # Cast commands for deposits
├── eth-val-tools-utils    # Validator tools commands
├── startup_log            # Kurtosis startup output & network info
└── validator_keys/        # Generated validator keystores
    ├── keys/
    ├── secrets/
    ├── teku-keys/
    ├── teku-secrets/
    ├── nimbus-keys/
    ├── lodestar-secrets/
    ├── prysm/
    └── pubkeys.json
```

---

## Useful Commands

```bash
# List running Kurtosis enclaves
kurtosis enclave ls

# Get enclave details
kurtosis enclave inspect billowing-caldera

# Stop and clean up
kurtosis enclave stop billowing-caldera
kurtosis clean -a

# View service logs
kurtosis service logs billowing-caldera el-1-geth-lighthouse
kurtosis service logs billowing-caldera cl-1-lighthouse-geth
```

---

## References

- [Ethereum Staking Launchpad](https://launchpad.ethereum.org/)
- [Kurtosis Ethereum Package](https://github.com/ethpandaops/ethereum-package)
- [eth2-val-tools](https://github.com/protolambda/eth2-val-tools)
- [Foundry Book](https://book.getfoundry.sh/)
