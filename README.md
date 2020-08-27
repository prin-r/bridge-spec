# None EVM Bridge specification

**Notice**: This document is a work-in-progress for researchers and implementers.

## Table of contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

**Table of Contents**

- [Introduction](#introduction)
- [Utility Functions](#utility-functions)
  - sha256
  - secp256k1 public key recovery
- [Dependency](#dependency)
  - OBI
- [Lite Client Verification Overview](#lite-client-verification-overview)
- [Structs](#structs)
  - [validator_with_power](#validator_with_power)
  - [request_packet](#request_packet)
  - [response_packet](#response_packet)
  - [iavl_merkle_path](#iavl_merkle_path)
  - [multi_store_proof](#multi_store_proof)
  - [block_header_merkle_parts](#block_header_merkle_parts)
  - [tm_signature](#tm_signature)
- [Bridge's storages](#bridge's-storages)
  - [total_validator_power](#total_validator_power)
  - [validator_powers](#validator_powers)
  - [oracle_states](#oracle_states)
  - [requests_caches](#requests_caches)
- [Bridge's functions](#bridge's-functions)
  - [get_total_validator_power](#get_total_validator_power)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

At Band Protocol, we provide a way for other blockchains to access off-chain information through our decentralized oracle. As part of that offering, we also provide a specification of lite client verification for anyone who requested data from our oracle to verify the validity of the result they received. We call the instance of lite client that is existed on other blockchains **Bridge**. The implementation of **Bridge** can be a smart contract (additional logic published by user) or a module (build in logic of a blockchain).

## Utility Functions

To implement the **Bridge** we only need two utility functions.

- 1. [sha256](https://en.wikipedia.org/wiki/SHA-2)
  - see python example implementation [here](utils/sha256.py)
- 2. [secp256k1 public key recovery](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm)
  - see python example implementation [here](utils/secp256k1.py)

## Dependency

**Bridge** implementation only have one spacial depencency which is **OBI**.

**OBI** or Oracle Binary Encoding is the standard way to serialized and deserialize binary data in the BandChain ecosystem.

- see full spec [here](https://docs.bandchain.org/developer/technical-specifications/obi.html#specification)
- see example implementation [here](https://github.com/bandprotocol/bandchain/blob/master/obi/pyobi/pyobi/pyobi.py)

## Lite Client Verification Overview

Once the other blockchain receives the oracle result, they proceed to verify that the result actually comes from BandChain. They do this by submitting a verification request to the **Bridge**. The aim of this process is to ensure that the data received is actually part of BandChain’s state and is signed by a sufficient number of BandChain’s block validators.

This process can be divided into two unrelated sub-processes.

- 1. **relay_oracle_state**: Verify that an `oracle module`**_[g]_** root hash module really exist on BandChain at a specific block and then save that root hash into **Bridge**'s state. This process requires the signatures of several validators signed on the block hash in which everyone who signs must have a total voting power greater than or equal to two-thirds of the entire voting power. The block hash is made up of multiple values that come from the BandChain state, where `oracle module`**_[g]_** root hash is one of them.

  ```text
                                 __ [BlockHash] __
                       _________|                 |___________
                      |                                       |
                   [3α]                                     [3ß]
            ______|    |_______                     _______|    |_______
           |                   |                   |                    |
       _ [2α] _            _ [2ß] _             _ [2Γ] _               [2Δ]
      |        |          |        |           |        |              |  |
    [1α]      [1ß]      [1Γ]      [1Δ]       [1ε]      [1ζ]          [C]  [D]
    |  |      |  |      |  |      |  |       |  |      |  |
  [0]  [1]  [2]  [3]  [4]  [5]  [6]  [7]   [8]  [9]  [A]  [B]
                                                      |
                                                      |
                                                      |
                                  ________________[AppHash]_______________
                                 /                                        \
                       _______[ρ9]______                          ________[ρ10]________
                      /                  \                       /                     \
                 __[ρ5]__             __[ρ6]__              __[ρ7]__               __[ρ8]__
                /         \          /         \           /         \            /         \
              [ρ1]       [ρ2]     [ρ3]        [ρ4]       [i]        [j]          [k]        [l]
             /   \      /   \    /    \      /    \
           [a]   [b]  [c]   [d] [e]   [f]  [g]    [h]

  # Leafs of BlockHash tree
  [0] - version               [1] - chain_id            [2] - height        [3] - time
  [4] - last_block_id         [5] - last_commit_hash    [6] - data_hash     [7] - validators_hash
  [8] - next_validators_hash  [9] - consensus_hash      [A] - app_hash      [B] - last_results_hash
  [C] - evidence_hash         [D] - proposer_address

  # Leafs of AppHash tree
  [a] - acc      [b] - distr   [c] - evidence  [d] - gov
  [e] - main     [f] - mint    [g] - oracle    [h] - params
  [i] - slashing [j] - staking [k] - supply    [l] - upgrade
  ```

- 2. **verify_oracle_data**: Verify a specific value that store under `oracle module`**_[g]_** is really existed by hashing the corresponding node's from bottom to top.

  - H(n) is an `oracle module`**_[g]_** root hash from the previous diagram.
  - C(i) is a corresponding node to H(i) where **i ∈ {0,1,2,...,n}** .

  ```text
                            _______________[H(n)]_______________
                          /                                      \
              _______[H(n-1)]______                             [C(n)]
            /                      \                          /        \
        [C(n-1)]                    \                       ...        ...
       /        \                    .
     ...        ...                   .
                                       .
                                        \
                                         \
                                _______[H(2)]______
                              /                    \
                           [H(1)]                 [C(1)]
                         /        \             /        \
                     [value]     [C(0)]       ...        ...
  ```

## Structs

#### validator_with_power

| Name        | Type      |
| ----------- | --------- |
| `validator` | `address` |
| `power`     | `uint256` |

```solidity
// An example of creating validator_with_power in Solidity.
contract Bridge {
    struct validator_with_power {
        address validator;
        uint256 power;
    }
}
```

#### request_packet

| Name               | Type     |
| ------------------ | -------- |
| `client_id`        | `string` |
| `oracle_script_id` | `u64`    |
| `params`           | `bytes`  |
| `ask_count`        | `u64`    |
| `min_count`        | `u64`    |

#### response_packet

| Name             | Type     |
| ---------------- | -------- |
| `client_id`      | `string` |
| `request_id`     | `u64`    |
| `ans_count`      | `u64`    |
| `request_time`   | `u64`    |
| `resolve_time`   | `u64`    |
| `resolve_status` | `u32`    |
| `result`         | `bytes`  |

#### iavl_merkle_path

| Name               | Type                     |
| ------------------ | ------------------------ |
| `is_data_on_right` | `bool`                   |
| `subtree_height`   | `u8`                     |
| `subtree_size`     | `u64`                    |
| `subtree_version`  | `u64`                    |
| `sibling_hash`     | `bytes`, fixed size = 32 |

#### multi_store_proof

| Name                                     | Type                     |
| ---------------------------------------- | ------------------------ |
| `acc_to_gov_stores_merkle_hash`          | `bytes`, fixed size = 32 |
| `main_and_mint_stores_merkle_hash`       | `bytes`, fixed size = 32 |
| `oracle_iavl_state_hash`                 | `bytes`, fixed size = 32 |
| `params_stores_merkle_hash`              | `bytes`, fixed size = 32 |
| `slashing_to_upgrade_stores_merkle_hash` | `bytes`, fixed size = 32 |

#### block_header_merkle_parts

| Name                                     | Type                     |
| ---------------------------------------- | ------------------------ |
| `version_and_chain_id_hash`              | `bytes`, fixed size = 32 |
| `time_hash`                              | `bytes`, fixed size = 32 |
| `last_block_id_and_other`                | `bytes`, fixed size = 32 |
| `next_validator_hash_and_consensus_hash` | `bytes`, fixed size = 32 |
| `last_results_hash`                      | `bytes`, fixed size = 32 |
| `evidence_and_proposer_hash`             | `bytes`, fixed size = 32 |

#### tm_signature

| Name                 | Type                     |
| -------------------- | ------------------------ |
| `r`                  | `bytes`, fixed size = 32 |
| `s`                  | `bytes`, fixed size = 32 |
| `v`                  | `u8`                     |
| `signed_data_prefix` | `bytes`                  |
| `signed_data_suffix` | `bytes`                  |

## Bridge's storages

#### total_validator_power

A storage variable that has the ability to hold a positive integer.

```solidity
// An example of creating total_validator_power in Solidity.
contract Bridge {
    uint256 public total_validator_power;
}
```

#### validator_powers

A storage mapping that has the ability to map an address to an integer.
For blockchains without the address type, something equivalent such as string, bytes or integer can be used instead.

```solidity
// An example of creating validator_powers in Solidity.
contract Bridge {
    mapping(address => uint256) public validator_powers;
}
```

#### oracle_states

A storage mapping that has the ability to map a positive integer (block height of BandChain) to a bytes32 (`oracle module`**_[g]_** root hash).
For blockchains without the bytes32 type, something equivalent such as string, bytes or integer can be used instead.

```solidity
// An example of creating validator_powers in Solidity.
contract Bridge {
    // A mapping from BandChain's block height to BandChain's oracle module root hash.
    mapping(uint256 => bytes32) public oracle_states;
}
```

#### requests_caches

A storage mapping that has the ability to map a bytes32 (hash of request packet) to a struct response packet.
For blockchains without the bytes32 type, something equivalent such as string, bytes or integer can be used instead.

```solidity
// An example of creating requests_caches in Solidity.
contract Bridge {
    // A mapping from hash of request packet to a struct response packet.
    mapping(bytes32 => response_packet) public requests_caches;
}
```

## Bridge's functions

#### get_total_validator_power

xxxxxxxxxxxxx
