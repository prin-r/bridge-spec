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

A structure that encapsulates the address and the amount of voting power on BandChain of a single validator.

| Field Name  | Type      | Description                           |
| ----------- | --------- | ------------------------------------- |
| `validator` | `address` | validator's address                   |
| `power`     | `uint256` | validator's voting power on BandChain |

#### request_packet

A structure that encapsulates the information about the request.

| Field Name         | Type     | Description                                                                                                                                                     |
| ------------------ | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `client_id`        | `string` | a string that refer to the requester, for example "from_scan", ...                                                                                              |
| `oracle_script_id` | `u64`    | an integer that refer to a specific oracle script on BandChain                                                                                                  |
| `params`           | `bytes`  | an obi encode of the request's parameters, for example "000000034254430000000a" (obi encode of ["BTC", 10])                                                     |
| `ask_count`        | `u64`    | the minimum number of validators necessary for the request to proceed to the execution phase. Therefore the minCount must be less than or equal to the askCount |
| `min_count`        | `u64`    | the number of validators that are requested to respond to this request                                                                                          |

#### response_packet

A structure that encapsulates the information about the response.

| Field Name       | Type     | Description                                                                                         |
| ---------------- | -------- | --------------------------------------------------------------------------------------------------- |
| `client_id`      | `string` | a string that refer to the requester, for example "from_scan", ...                                  |
| `request_id`     | `u64`    | an integer that refer to a specific request on BandChain                                            |
| `ans_count`      | `u64`    | a number of answers that was answered by validators                                                 |
| `request_time`   | `u64`    | unix time at which the request was created on BandChain                                             |
| `resolve_time`   | `u64`    | unix time at which the request got a number of reports/answers greater than or equal to `min_count` |
| `resolve_status` | `u32`    | status of the request (0=Open, 1=Success, 2=Failure, 3=Expired)                                     |
| `result`         | `bytes`  | an obi encode of the request's result, for example "0000aaaa" (obi encode of [ 43690 ])             |

#### iavl_merkle_path

A structure of merkle proof that shows how the data leaf is part of the `oracle module`**_[g]_** tree. The proof’s content is the list of “iavl_merkle_path” from the leaf to the root of the tree.

| Field Name         | Type                     | Description                                                    |
| ------------------ | ------------------------ | -------------------------------------------------------------- |
| `is_data_on_right` | `bool`                   | whether the data is on the right subtree of this internal node |
| `subtree_height`   | `u8`                     | the height of this subtree                                     |
| `subtree_size`     | `u64`                    | the size of this subtree                                       |
| `subtree_version`  | `u64`                    | the latest block height that this subtree has been updated     |
| `sibling_hash`     | `bytes`, fixed size = 32 | hash of the other child subtree                                |

#### multi_store_proof

A structure that encapsulates sibling module hashes of the `app_hash`**_[A]_** which are `params module`**_[h]_**, `main,mint modules`**_[ρ3]_**, `acc,distr,evidence,gov modules`**_[ρ5]_**, `slashing,staking,supply,upgrade modules`**_[ρ10]_**.

| Field Name                               | Type                     | Description                                                        |
| ---------------------------------------- | ------------------------ | ------------------------------------------------------------------ |
| `acc_to_gov_stores_merkle_hash`          | `bytes`, fixed size = 32 | root hash of acc,distr,evidence,gov modules (**_[ρ5]_**)           |
| `main_and_mint_stores_merkle_hash`       | `bytes`, fixed size = 32 | root hash of main and mint modules (**_[ρ3]_**)                    |
| `oracle_iavl_state_hash`                 | `bytes`, fixed size = 32 | root hash of oracle module (**_[g]_**)                             |
| `params_stores_merkle_hash`              | `bytes`, fixed size = 32 | root hash of params module (**_[h]_**)                             |
| `slashing_to_upgrade_stores_merkle_hash` | `bytes`, fixed size = 32 | root hash of slashing,staking,supply,upgrade modules (**_[ρ10]_**) |

#### block_header_merkle_parts

A structure that encapsulates ...

| Field Name                               | Type                     | Description                                               |
| ---------------------------------------- | ------------------------ | --------------------------------------------------------- |
| `version_and_chain_id_hash`              | `bytes`, fixed size = 32 | root hash of version and chain id components (**_[1α]_**) |
| `time_hash`                              | `bytes`, fixed size = 32 | hash of time component (**_[3]_**)                        |
| `last_block_id_and_other`                | `bytes`, fixed size = 32 | root hash of last_block_id and                            |
| `next_validator_hash_and_consensus_hash` | `bytes`, fixed size = 32 | root hash of version and chain id components (**_[1ε]_**) |
| `last_results_hash`                      | `bytes`, fixed size = 32 | hash of last results component (**_[B]_**)                |
| `evidence_and_proposer_hash`             | `bytes`, fixed size = 32 | hash of evidence and proposer components (**_[2Δ]_**)     |

#### tm_signature

A structure that encapsulates Tendermint's precommit data and validator's signature for performing signer recovery for ECDSA secp256k1 signature. Tendermint's precommit data compose of block hash and some additional information prepended and appended to the block
hash. The prepended part (prefix) and the appended part (suffix) are different for each signer
(including signature size, machine clock, validator index, etc).

| Field Name           | Type                     | Description                                                      |
| -------------------- | ------------------------ | ---------------------------------------------------------------- |
| `r`                  | `bytes`, fixed size = 32 | a part of signature                                              |
| `s`                  | `bytes`, fixed size = 32 | a part of signature                                              |
| `v`                  | `u8`                     | a value that helps reduce the calculation of public key recovery |
| `signed_data_prefix` | `bytes`                  | The prepended part of Tendermint's precommit data                |
| `signed_data_suffix` | `bytes`                  | The appended part of Tendermint's precommit data                 |

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
