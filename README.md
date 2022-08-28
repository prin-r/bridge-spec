# Bridge contract

**This document is working in progress**

## Table of contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

**Table of Contents**

- [Introduction](#introduction)
- [Utility Functions](#utility-functions)
  - sha256
  - ecrecover
- [Dependencies](#dependencies)
  - OBI
  - Protobuf
- [Lite Client Verification Overview](#lite-client-verification-overview)
  - [Getting the proof from Band](#Getting_the_proof_from_Band)
  - [Proof structure](#Proof_structure)
- [Bridge contract](#bridge-contract)
  - [Structs](#structs)
    - [validator_with_power](#validator_with_power)
    - [request_packet](#request_packet)
    - [response_packet](#response_packet)
    - [iavl_merkle_path](#iavl_merkle_path)
    - [multi_store_proof](#multi_store_proof)
    - [block_header_merkle_parts](#block_header_merkle_parts)
    - [tm_signature](#tm_signature)
    - [block_detail](#block_detail)
    - [common_encoded_vote_part](#common_encoded_vote_part)
  - [Storages](#Storages)
    - [total_validator_power](#total_validator_power)
    - [validator_powers](#validator_powers)
    - [oracle_states](#oracle_states)
    - [encoded_chain_id](#encoded_chain_id)
  - [Functions](#bridge's-functions)
    - [get_total_validator_power](#get_total_validator_power)
    - [get_number_of_validators](#get_number_of_validators)
    - [get_validators](#get_validators)
    - [get_all_validators](get_all_validators)
    - [get_oracle_state](#get_oracle_state)
    - [get_validator_power](#get_validator_power)
    - [merkle_leaf_hash](#merkle_leaf_hash)
    - [merkle_inner_hash](#merkle_inner_hash)
    - [encode_varint_unsigned](#encode_varint_unsigned)
    - [encode_varint_signed](#encode_varint_signed)
    - [get_block_header](#get_block_header)
    - [check_parts_and_encoded_common_parts](#check_parts_and_encoded_common_parts)
    - [check_time_and_recover_signer](#check_time_and_recover_signer)
    - [get_parent_hash](#get_parent_hash)
    - [relay_block](#relay_block)
    - [verify_oracle_data](#verify_oracle_data)
    - [verify_requests_count](#verify_requests_count)
    - [relay_and_verify](#relay_and_verify)
    - [relay_and_multi_verify](#relay_and_multi_verify)
    - [relay_and_verify_count](#relay_and_verify_count)
    - [verify_proof](#verify_proof)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

Band Protocol provide a way for other blockchains to access off-chain information through our decentralized oracle. As part of that offering, we also provide a specification of lite client verification for anyone who requested data from our oracle to verify the validity of the result they received. We call the instance of lite client that is existed on other blockchains **Bridge**. The implementation of **Bridge** can be a smart contract (additional logic published by user) or a module (build in logic of a blockchain).

## Utility Functions

To implement the **Bridge** we only need two utility functions.

- 1. [sha256](https://en.wikipedia.org/wiki/SHA-2)
  - see python example implementation [here](../../example_utils_functions/sha256.py)
- 2. [ecrecover](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm)
  - see python example implementation [here](../../example_utils_functions/secp256k1.py)

## Dependencies

- **OBI** or Oracle Binary Encoding is the standard way to serialized and deserialize binary data in the BandChain ecosystem.
  - see OBI wiki page [here](https://github.com/bandprotocol/bandchain/wiki/Oracle-Binary-Encoding-(OBI))
  - see example implementations [here](https://github.com/bandprotocol/bandchain/tree/master/obi)

- **ProtobufLib** or Oracle Binary Encoding is the standard way to serialized and deserialize binary data in the BandChain ecosystem.
  - we adopted the lib from [here](https://github.com/lazyledger/protobuf3-solidity-lib/blob/master/contracts/ProtobufLib.sol)
  - see the .sol file [here](./library/ProtobufLib.sol)

## Lite Client Verification Overview

When the client needs external data, they will come to the oracle. In our case, they will come to the Bandchain and make a request transaction. After that, the oracle mechanic behind will gather the result for the request, which will be stored persistently in the chain's state. 
Once the client saw the oracle result, they proceed to verify that the result actually comes from BandChain. 
They do this by gathering a Bandchain's block with enough `signatures` and relaying that block to the `Bridge`. If block relaying was successful, they could verify their result using a Merkle proof-like scheme. This process aims to ensure that the data received is part of BandChain's state and is signed by a sufficient number of BandChain's block validators.

The signatures were created by validators signing the message on [Tendermint](https://github.com/tendermint/tendermint/tree/v0.34.x) called [CanonicalVote](https://github.com/tendermint/tendermint/blob/v0.34.x/spec/core/data_structures.md#canonicalvote). The [CanonicalVote](https://github.com/tendermint/tendermint/blob/v0.34.x/spec/core/data_structures.md#canonicalvote) is composed of six fields. Some of those fields have their sub-fields containing the block hash. The diagram below help explain the [CanonicalVote](https://github.com/tendermint/tendermint/blob/v0.34.x/spec/core/data_structures.md#canonicalvote) message by expanding its struct, which includes the structs of the internal fields as well. 

```go
type SignedMsgType int32

type CanonicalPartSetHeader struct {
    Total   uint32  `protobuf:"varint,1,opt,name=total`
    Hash    []byte  `protobuf:"bytes,2,opt,name=hash`
}

type CanonicalBlockID struct {
    // This is the block hash
    Hash            []byte                  `protobuf:"bytes,1,opt,name=hash`
    PartSetHeader   CanonicalPartSetHeader  `protobuf:"bytes,2,opt,name=part_set_header`
}

type CanonicalVote struct {
    Type        SignedMsgType       `protobuf:"varint,1,opt,name=type`
    Height      int64               `protobuf:"fixed64,2,opt,name=height`
    Round       int64               `protobuf:"fixed64,3,opt,name=round`
    BlockID     *CanonicalBlockID   `protobuf:"bytes,4,opt,name=block_id`
    Timestamp   time.Time           `protobuf:"bytes,5,opt,name=timestamp`
    ChainID     string              `protobuf:"bytes,6,opt,name=chain_id`
}
```

As the validators must sign on the [CanonicalVote](https://github.com/tendermint/tendermint/blob/v0.34.x/spec/core/data_structures.md#canonicalvote), most fields and sub-fields are the same for all validators except the `Timestamp`. Each validator can come up with its `Timestamp`, which can be different from others.
Most of the [CanonicalVote](https://github.com/tendermint/tendermint/blob/v0.34.x/spec/core/data_structures.md#canonicalvote)'s parts can be included directly in the lite-client-proof except the `block hash` or `CanonicalVote.BlockID.Hash`, which need to be calculated on the client chain.
In the diagram below, the `block hash` will be calculated by hashing the raw data from the bottom of the tree upwards to the root.

```text
                                          __ [BlockHash] __
                                _________|                 |___________
                               |                                       |
                             [3α]                                    [3ß]
                     ________|  |_______                     ________|  |________
                    |                   |                   |                    |
                _ [2α] _            _ [2ß] _             _ [2Γ] _               [2Δ]
               |        |          |        |           |        |              |  |
             [1α]      [1ß]      [1Γ]      [1Δ]       [1ε]      [1ζ]          [C]  [D]
             |  |      |  |      |  |      |  |       |  |      |  |
           [0]  [1]  [2]  [3]  [4]  [5]  [6]  [7]   [8]  [9]  [A]  [B]
                                                               |
                                                               |
                                                               |
                                           ________________[AppHash]_________________
                                          /                                          \
                      _________________[I14]_________________                        [G]
                     /                                        \
          _______[I12]______                          _______[I13]________
         /                  \                        /                    \
    __[I8]__             __[I9]__                __[I10]__              __[I11]__
   /         \          /         \            /          \            /         \
 [I0]       [I1]     [I2]        [I3]        [I4]        [I5]        [I6]       [I7]
/   \      /   \    /    \      /    \      /    \      /    \      /    \     /    \
[0]   [1]  [2]   [3] [4]   [5]  [6]    [7]  [8]    [9]  [A]    [B]  [C]    [D]  [E]   [F]

# Leafs of BlockHash tree
[0] - version               [1] - chain_id            [2] - height        [3] - time
[4] - last_block_id         [5] - last_commit_hash    [6] - data_hash     [7] - validators_hash
[8] - next_validators_hash  [9] - consensus_hash      [A] - app_hash      [B] - last_results_hash
[C] - evidence_hash         [D] - proposer_address

# Leafs of AppHash tree
[0] - auth     [1] - authz    [2] - bank    [3] - capability [4] - crisis  [5] - dist
[6] - evidence [7] - feegrant [8] - gov     [9] - ibccore    [A] - mint    [B] - oracle
[C] - params   [D] - slashing [E] - staking [F] - transfer   [G] - upgrade
```


As described above, the entire process can be divided into two parts.
1. **relay_block**: Verify that an `oracle module`<strong><em>[B]</em></strong> root hash module really exist on BandChain at a specific block and then save that root hash into **Bridge**'s state. This process requires the signatures of several validators signed on the block hash in which everyone who signs must have a total voting power greater than or equal to two-thirds of the entire voting power. The block hash is made up of multiple values that come from the BandChain state, where `oracle module`<strong><em>[B]</em></strong> root hash is one of them.
2. **verify_oracle_data**: Verify a specific value that store under `oracle module`<strong><em>[B]</em></strong> is really existed by hashing the corresponding node's from bottom to top.

  - **n** is the height of IAVL merkle tree
  - **H(n)** is an `oracle module`<strong><em>[B]</em></strong> root hash from the previous diagram.
    - **H(0)** is basically derived from **value**
  - **C(i)** is a corresponding node to H(i) where **i ∈ {0,1,2,...,n-1}** .

  ```text
                            _______________[H(n)]_______________
                          /                                      \
              _______[H(n-1)]______                             [C(n-1)]
            /                      \                          /        \
        [C(n-2)]                    \                       ...        ...
       /        \                    .
     ...        ...                   .
                                       .
                                        \
                                         \
                                _______[H(2)]______
                              /                    \
                           [H(1)]                 [C(1)]
                         /        \             /        \
                      [H(0)]     [C(0)]       ...        ...
                      [value]
  ```

### Getting_the_proof_from_Band

Usually the client can use the Tendermint RPC call to ask for the information they need to construct the proof.
However, we have implemented an endpoint to make this process easier. Our proof endpoint on the mainnet is `https://laozi4.bandchain.org/api/oracle/proof` + `/A_SPECIFIC_REQUEST_ID`.

Please see this example [proof of the request number 11124603](https://laozi4.bandchain.org/api/oracle/proof/11124603).

### Proof_structure

Example proof struct

**_Please note that this example cut out some of the `merkle_paths` and `signatures` to make it easier to read._**

```json
{
    "block_height":"9023063",
    "oracle_data_proof":{
        "result":{
            "oracle_script_id":"37",
            "calldata":"AAAAIN8cxc1vvrIbADsPV8GCSs5LhNVrln97PJVTl25QjoqyAAAAAGK0IkkAAAAUAGMEZobkbcbxWRi2GuKxIUWFNKU=",
            "ask_count":"16",
            "min_count":"1",
            "request_id":"11124603",
            "ans_count":"13",
            "request_time":"1661414867",
            "resolve_time":"1661414873",
            "resolve_status":1,
            "result":"AAAAUMLDALuONqBE8bh9vcXMZxtS2z2aFEyCTwRxz8rawrix4E9ApDNCcFv5o3NMyb666tyIrNYkN1XiKu/1SBKUbI783wBkUzYdtqMAJ4bhKpUKAAAAQLx5jr83yZAYwkq0AJBypNfuMRWa/iaE9oW82KLlZ4IinQlOmVtWnh/ZR1O2RNtbBbe5FZfkHDVWa1gjPYLEcEw="
        },
        "version":"9023058",
        "merkle_paths":[
            {
                "is_data_on_right":true,
                "subtree_height":1,
                "subtree_size":"2",
                "subtree_version":"9023058",
                "sibling_hash":"BC49843DE942EFC722E33DF11A0F2D5F535C46E88BC0A73619ACE776DE30FF81"
            },
            {
                "is_data_on_right":true,
                "subtree_height":3,
                "subtree_size":"5",
                "subtree_version":"9023059",
                "sibling_hash":"40E1F726F0A1C5CB8589A22A665E9CA9C5D09468A746ED78D20D2E7DB0946831"
            }
        ]
    },
    "block_relay_proof":{
        "multi_store_proof":{
            "auth_to_fee_grant_stores_Merkle_hash":"12BC0C7C74ABB52A53B80DCD2006077842ABDD6370A765C37A1AA607B05EB08F",
            "gov_to_ibc_core_stores_merkle_hash":"901C6E9F2615AECEACCA2CB2E5AEB9FD6BAA411637E04C79D8AC3FC6BE852A52",
            "mint_store_merkle_hash":"96AC1DE3D6B0DB2E725C2797396917B0EB4C5EE924BA115F985F026BF56BD33E",
            "oracle_iavl_State_hash":"CED5C04B645F2FC4FE09390D4CB10D945F3D8D1EFCB7A5D0C03CE4A62ECB344C",
            "params_to_transfer_stores_merkle_hash":"02A2938AB10B9470FD0F6D6FE9FBDA3E9158CA305C9C862C786D63BB3E56C46A",
            "upgrade_store_merkle_hash":"C9C8849ED125CC7681329C4D27B83B1FC8ACF7A865C9D1D1DF575CCA56F48DBE"
        },
        "block_header_merkle_parts":{
            "version_and_chain_id_hash":"327BB8E7C548CDA20A3F9330131E658F4BCB5BE238F5EE94D79FD5019E3F7559",
            "height":"9023063",
            "time_second":"1661414887",
            "time_nano_second":925334537,
            "last_block_id_and_other":"5CC36D89F15B2CE914E82BC8A6FAFCE35E9A2DD9F403A0119465AE27923A01C1",
            "next_validator_hash_and_consensus_hash":"0CCD14362C66E03A8E99E6CB25F523DF17DD5375521E92CD96686EAB2A2E5AE0",
            "last_results_hash":"2AD670892C7CEEDFFA685848433C20487F736029D95F4B2EA8F87FFA35031AF0",
            "evidence_and_proposer_hash":"F168695977991FEBB011BF7B88B3482C1E8352F3DBE40DE3FECE87E678EA7D98"
        },
        "common_encoded_vote_part":{
            "signed_data_prefix":"08021157AE89000000000022480A20",
            "signed_data_suffix":"1224080112208A906920FEE879E49590274DAE39CA3C7027D5AB645AF6946E316F9C51F55C0E"
        },
        "signatures":[
            {
                "r":"CE3A3511742253465E4CF8E66435D4EF3024E066C085E2FE259D12FD51B3885F",
                "s":"31C943EEC7517CF5431A1C8BB792F0D5185B69DA7F8CDDF4339636472833B0EE",
                "v":28,
                "encoded_timestamp":"08EADB9C980610B5F8A9F802"
            },
            {
                "r":"62E1DE88CBB581B5798D682F4CCD9A8E1C81C7C323D5E5DD32445633931ABD6A",
                "s":"76B6013A16AAE1FB4A4272E081ED64E81C9A462EC6D675D3E3790781BD7205F0",
                "v":27,
                "encoded_timestamp":"08EADB9C98061097C398F402"
            },
            {
                "r":"9B17F1A2338CE7079878BA2D58BFC2378C54A18AB8EBC3880A347FD5546EF9D1",
                "s":"0F2915E06D3CA4267DE2F102642DD9529CA9679D3A64B2288DAEBEB2ED9AEE91",
                "v":27,
                "encoded_timestamp":"08EADB9C980610F59AC9F702"
            }
        ]
    }
}
```

## Bridge contract

### Structs

#### validator_with_power

A structure that encapsulates the public key and the amount of voting power on BandChain of a single validator.

| Field Name  | Type                | Description                                                                                                                           |
| ----------- | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `validator` | `bytes`, fixed size | validator's public key or something unique that is derived from public key such as hash of public key, compression form of public key |
| `power`     | `u64`               | validator's voting power on BandChain                                                                                                 |

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

A structure of merkle proof that shows how the data leaf is part of the `oracle module`<strong><em>[B]</em></strong> tree. The proof’s content is the list of “iavl_merkle_path” from the leaf to the root of the tree.

| Field Name         | Type                     | Description                                                    |
| ------------------ | ------------------------ | -------------------------------------------------------------- |
| `is_data_on_right` | `bool`                   | whether the data is on the right subtree of this internal node |
| `subtree_height`   | `u8`                     | the height of this subtree                                     |
| `subtree_size`     | `u64`                    | the size of this subtree                                       |
| `subtree_version`  | `u64`                    | the latest block height that this subtree has been updated     |
| `sibling_hash`     | `bytes`, fixed size = 32 | hash of the other child subtree                                |

#### multi_store_proof

A structure that encapsulates sibling module hashes of the `app_hash`<strong><em>[A]</em></strong> which are `params module`<strong><em>[h]</em></strong>, `main,mint modules`<strong><em>[ρ3]</em></strong>, `acc,distr,evidence,gov modules`<strong><em>[ρ5]</em></strong>, `slashing,staking,supply,upgrade modules`<strong><em>[ρ10]</em></strong>.

| Field Name                               | Type                     | Description                                                                            |
| ---------------------------------------- | ------------------------ | -------------------------------------------------------------------------------------- |
| `acc_to_gov_stores_merkle_hash`          | `bytes`, fixed size = 32 | root hash of acc,distr,evidence,gov modules (<strong><em>[ρ5]</em></strong>)           |
| `main_and_mint_stores_merkle_hash`       | `bytes`, fixed size = 32 | root hash of main and mint modules (<strong><em>[ρ3]</em></strong>)                    |
| `oracle_iavl_state_hash`                 | `bytes`, fixed size = 32 | root hash of oracle module (<strong><em>[B]</em></strong>)                             |
| `params_stores_merkle_hash`              | `bytes`, fixed size = 32 | root hash of params module (<strong><em>[h]</em></strong>)                             |
| `slashing_to_upgrade_stores_merkle_hash` | `bytes`, fixed size = 32 | root hash of slashing,staking,supply,upgrade modules (<strong><em>[ρ10]</em></strong>) |

#### block_header_merkle_parts

A structure that encapsulates the components of a block header that correspond to height<strong><em>[2]</em></strong> and app hash <strong><em>[A]</em></strong>.

| Field Name                               | Type                     | Description                                                                                               |
| ---------------------------------------- | ------------------------ | --------------------------------------------------------------------------------------------------------- |
| `version_and_chain_id_hash`              | `bytes`, fixed size = 32 | root hash of version and chain id components (<strong><em>[1α]</em></strong>)                             |
| `time_hash`                              | `bytes`, fixed size = 32 | hash of time component (<strong><em>[3]</em></strong>)                                                    |
| `last_block_id_and_other`                | `bytes`, fixed size = 32 | root hash of last block id, last commit hash, data hash, validators hash (<strong><em>[2ß]</em></strong>) |
| `next_validator_hash_and_consensus_hash` | `bytes`, fixed size = 32 | root hash of version and chain id components (<strong><em>[1ε]</em></strong>)                             |
| `last_results_hash`                      | `bytes`, fixed size = 32 | hash of last results component (<strong><em>[B]</em></strong>)                                            |
| `evidence_and_proposer_hash`             | `bytes`, fixed size = 32 | hash of evidence and proposer components (<strong><em>[2Δ]</em></strong>)                                 |

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

<strong>Example for creating the total_validator_power storage</strong>

Solidity

```solidity
contract Bridge {
    uint256 public total_validator_power;
}
```

Score

```python3
class Bridge(IconScoreBase):
  def __init__(self, db: IconScoreDatabase) -> None:
    self.total_validator_power = VarDB("total_validator_power", db, value_type=int)
```

#### validator_powers

A storage mapping that has the ability to map an address to an integer.
For blockchains without the address type, something equivalent such as string, bytes or integer can be used instead.

<strong>Example for creating the validator_powers storage</strong>

Solidity

```solidity
contract Bridge {
    mapping(address => uint256) public validator_powers;
}
```

Score

```python3
class Bridge(IconScoreBase):
  def __init__(self, db: IconScoreDatabase) -> None:
    self.validator_powers = DictDB("validator_powers", db, value_type=int)
```

#### oracle_states

A storage mapping that has the ability to map a positive integer (block height of BandChain) to a bytes32 (`oracle module`<strong><em>[B]</em></strong> root hash).
For blockchains without the bytes32 type, something equivalent such as string, bytes or integer can be used instead.

<strong>Example for creating the oracle_states storage</strong>

Solidity

```solidity
contract Bridge {
    mapping(uint256 => bytes32) public oracle_states;
}
```

Score

```python3
class Bridge(IconScoreBase):
  def __init__(self, db: IconScoreDatabase) -> None:
    self.oracle_state = DictDB("oracle_state", db, value_type=bytes)
```

#### requests_caches

A storage mapping that has the ability to map a bytes32 (hash of request packet) to a struct response packet.
For blockchains without the bytes32 type, something equivalent such as string, bytes or integer can be used instead. And for blockchains who do not have money can use gold instead

<strong>Example for creating the requests_caches storage</strong>

Solidity

```solidity
contract Bridge {
  mapping(bytes32 => response_packet) public requests_caches;
}
```

Score

```python3
class Bridge(IconScoreBase):
  def __init__(self, db: IconScoreDatabase) -> None:
    # We store an encoded of response_packet because in Score we can only store primitive types
    self.requests_cache = DictDB("requests_cache", db, value_type=bytes)
```

## Bridge's functions

#### get_total_validator_power

Get the total voting power of active validators currently on duty. This function should read value from the storage `total_validator_power` and then return the value.

params

```
no parameters
```

return values

| Type  | Field Name         | Description                                                    |
| ----- | ------------------ | -------------------------------------------------------------- |
| `u64` | total voting power | value of `total_validator_power` that is read from the storage |

#### get_oracle_state

Get the iAVL Merkle tree hash of `oracle module`<strong><em>[B]</em></strong> from given block height of the BandChain. This function should read value from the storage `oracle_states` and then return the value.

params

| Type  | Field Name   | Description                                                                                                                                      |
| ----- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `u64` | block height | The height of block in BandChain that the `oracle module`<strong><em>[B]</em></strong> hash was relayed on the chain where this `Bridge` resides |

return values

| Type    | Field Name        | Description                                              |
| ------- | ----------------- | -------------------------------------------------------- |
| `bytes` | oracle state hash | Hash of the `oracle module`<strong><em>[B]</em></strong> |

#### get_validator_power

Get voting power of a validator on BandChain from the storage `validator_power`. This function receive the public key of the validator (can be hash of the public key) and then return the voting power.

params

| Type    | Field Name | Description                                                                                                                                                                                                                                                                     |
| ------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bytes` | validator  | The public key of a validator or something unique that is derived from public such as hash of public key (in Ethereum block chain, we use type `address`). This can be any type that can be used to represent public key. For example `bytes`, `address`, `uint`, `string`, ... |

return values

| Type  | Field Name   | Description                 |
| ----- | ------------ | --------------------------- |
| `u64` | voting power | Voting power of a validator |

#### update_validator_powers

Update validator powers by owner. This function receive array of struct `validator_with_power` and then update storages `validator_power` and `total_voting_power` without returning any value. The encoded format of the array of struct `validator_with_power` can be used instead in case the platform does not support the use of the complex type parameters.

params

| Type            | Field Name            | Description                               |
| --------------- | --------------------- | ----------------------------------------- |
| `[{bytes,u64}]` | validators with power | An array of struct `validator_with_power` |

return values

```
no return value
```

<strong>Example implementation</strong>

```python3
# Update validator powers by owner.
# We use an encoded of [{bytes,u64}] in this case because right now Score does not support complex type parameters
# @param validators_bytes OBI encoded of the changed set of BandChain validators.
@external
def update_validator_powers(validators_bytes: bytes):
    # This function can only be call by owner
    if msg.sender != owner:
        revert("NOT_AUTHORIZED")

    obi = PyObi("""[{pubkey:bytes, power:u64}]""")
    total_validator_power = total_validator_power.get()
    for validator_with_power in obi.decode(validators_bytes):
        pubkey = validator_with_power["pubkey"]
        power = validator_with_power["power"]
        if len(pubkey) != 64:
            revert(f"PUBKEY_SHOULD_BE_64_BYTES_BUT_GOT_{len(pubkey)}_BYTES")

        total_validator_power -= validator_powers[pubkey]
        total_validator_power += power

        validator_powers[pubkey] = power

    total_validator_power.set(total_validator_power)
```

#### merkle_leaf_hash

This function receive any bytes as an `input` and then does these following step.

1. prepend the `input` with a zero byte.
2. return [sha256](#utility-functions) of the result from `1.`.

params

| Type    | Field Name | Description |
| ------- | ---------- | ----------- |
| `bytes` | `input`    | Any bytes   |

return values

| Type                     | Field Name | Description                                |
| ------------------------ | ---------- | ------------------------------------------ |
| `bytes`, fixed size = 32 | result     | [sha256](#utility-functions)(0x00 + input) |

<strong>Example implementation</strong>

```python3
def merkle_leaf_hash(input: bytes) -> bytes:
    return sha256(bytes([0]) + input)
```

#### merkle_inner_hash

This function takes two parameters `left` and `right`, both of them are bytes, and then does these the following steps.

1. append `left` with `right` and then prepend it with a byte 01.
2. return sha256 of the result from `1.`.

params

| Type    | Field Name | Description |
| ------- | ---------- | ----------- |
| `bytes` | `input`    | Any bytes   |

return values

| Type                     | Field Name | Description                                       |
| ------------------------ | ---------- | ------------------------------------------------- |
| `bytes`, fixed size = 32 | result     | [sha256](#utility-functions)(0x01 + left + right) |

<strong>Example implementation</strong>

```python3
def merkle_inner_hash(left: bytes, right: bytes) -> bytes:
    return sha256(bytes([1]) + left + right)
```

#### encode_varint_unsigned

This function receive an integer as an `input` and then return an [encode varint unsigned of the `input`](#https://developers.google.com/protocol-buffers/docs/encoding).

params

| Type        | Field Name | Description          |
| ----------- | ---------- | -------------------- |
| any integer | `input`    | Any unsigned integer |

return values

| Type    | Field Name | Description                           |
| ------- | ---------- | ------------------------------------- |
| `bytes` | result     | Encode varint unsigned of the `input` |

<strong>Example implementation</strong>

```python3
def encode_varint_unsigned(input: int) -> bytes:
    result = b""
    while input > 0:
        result += bytes([128 | (input & 127)])
        input >>= 7
    return result[: len(result) - 1] + bytes([result[len(result) - 1] & 127])
```

#### encode_varint_signed

This function receive an integer as an `input` and then return an [`encode varint signed of the input`](#https://developers.google.com/protocol-buffers/docs/encoding). We can say it basically return [encode_varint_unsigned(input ⨯ 2)](#encode_varint_unsigned)

params

| Type        | Field Name | Description        |
| ----------- | ---------- | ------------------ |
| any integer | `input`    | Any signed integer |

return values

| Type    | Field Name | Description                         |
| ------- | ---------- | ----------------------------------- |
| `bytes` | result     | Encode varint signed of the `input` |

<strong>Example implementation</strong>

```python3
def encode_varint_signed(input: int) -> bytes:
    return encode_varint_unsigned(input * 2)
```

#### get_block_header

This function receive 3 parameters which are struct `block_header_merkle_parts`, `app_hash`**_[A]_** and `block_height`**_[2]_**. It will calculate the `BlockHash` according to [`merkle tree`](https://en.wikipedia.org/wiki/Merkle_tree) hashing scheme and then return the `BlockHash`.

```text
                               __ [BlockHash] __
                     _________|                 |__________
                    |                                      |
                  [3α]                                    [3ß]
          ________|  |_______                     ________|  |________
         |                   |                   |                    |
     _ [2α] _            _ [2ß] _             _ [2Γ] _               [2Δ]
    |        |          |        |           |        |              |  |
  [1α]      [1ß]      [1Γ]      [1Δ]       [1ε]      [1ζ]          [C]  [D]
  |  |      |  |      |  |      |  |       |  |      |  |
[0]  [1]  [2]  [3]  [4]  [5]  [6]  [7]   [8]  [9]  [A]  [B]

# Leafs of BlockHash tree
[0] - version               [1] - chain_id            [2] - height        [3] - time
[4] - last_block_id         [5] - last_commit_hash    [6] - data_hash     [7] - validators_hash
[8] - next_validators_hash  [9] - consensus_hash      [A] - app_hash      [B] - last_results_hash
[C] - evidence_hash         [D] - proposer_address
```

params

| Type                                    | Field Name                  | Description                          |
| --------------------------------------- | --------------------------- | ------------------------------------ |
| `{bytes,bytes,bytes,bytes,bytes,bytes}` | `block_header_merkle_parts` | A struct `block_header_merkle_parts` |
| `bytes`                                 | `input`                     | Any signed integer                   |
| `u64`                                   | `input`                     | Any signed integer                   |

return values

| Type                     | Field Name  | Description                                   |
| ------------------------ | ----------- | --------------------------------------------- |
| `bytes`, fixed size = 32 | `BlockHash` | The block hash of BancChain at `block_height` |

1. **_[2]_** = [merkle_leaf_hash](#merkle_leaf_hash)([encode_varint_unsigned](#encode_varint_unsigned)(`block_height`))

2. **_[1ß]_** = [merkle_inner_hash](#merkle_inner_hash)(**_[2]_**, `block_header_merkle_parts.time_hash`**_[3]_**)

3. **_[2α]_** = [merkle_inner_hash](#merkle_inner_hash)(`block_header_merkle_parts.version_and_chain_id_hash`**_[1α]_**, **_[1ß]_**)

4. **_[3α]_** = [merkle_inner_hash](#merkle_inner_hash)(**_[2α]_**, `block_header_merkle_parts.last_block_id_and_other`**_[2ß]_**)

5. **_[A]_** = [merkle_leaf_hash](#merkle_leaf_hash)(bytes([32]) + app_hash) // Prepend with a byte of 32 or 0x20

6. **_[1ζ]_** = [merkle_inner_hash](#merkle_inner_hash)(**_[A]_**, `block_header_merkle_parts.last_results_hash`**_[B]_**)

7. **_[2Γ]_** = [merkle_inner_hash](#merkle_inner_hash)(`block_header_merkle_parts.next_validator_hash_and_consensus_hash`**_[1ε]_**, **_[1ζ]_**)

8. **_[3ß]_** = [merkle_inner_hash](#merkle_inner_hash)(**_[2Γ]_**, `block_header_merkle_parts.evidence_and_proposer_hash`**_[2Δ]_**)

9. `BlockHash` = [merkle_inner_hash](#merkle_inner_hash)(**_[3α]_**,**_[3ß]_**)

10. return `BlockHash`

<strong>Example implementation</strong>

```python3
# Notice that we use bytes instead of struct block_header_merkle_parts because currently Score does not support sturct parameters. So the bytes block_header_merkle_parts in this case is just a concatenation of every fields from struct block_header_merkle_parts.
def get_block_header(block_header_merkle_parts: bytes, app_hash: bytes, block_height: int) -> bytes:
    return merkle_inner_hash(  # [BlockHeader]
        merkle_inner_hash(  # [3α]
            merkle_inner_hash(  # [2α]
                block_header_merkle_parts[0:32],  # [1α]
                merkle_inner_hash(  # [1ß]
                    merkle_leaf_hash(encode_varint_unsigned(block_height)),  # [2]
                    block_header_merkle_parts[32:64],  # [3]
                ),
            ),
            block_header_merkle_parts[64:96],  # [2ß]
        ),
        merkle_inner_hash(  # [3ß]
            merkle_inner_hash(  # [2Γ]
                block_header_merkle_parts[96:128],  # [1ε]
                merkle_inner_hash(  # [1ζ]
                    merkle_leaf_hash(bytes([32]) + app_hash), block_header_merkle_parts[128:160]  # [A], [B]
                ),
            ),
            block_header_merkle_parts[160:192],  # [2Δ]
        ),
    )
```

#### get_app_hash

This function receive a struct `multi_store_proof` as an input and then return the `AppHash` by the calculation according to [`merkle tree`](https://en.wikipedia.org/wiki/Merkle_tree) hashing scheme.

```text
                                               ________________[AppHash]_________________
                                              /                                          \
                          _________________[I14]_________________                        [G]
                         /                                        \
              _______[I12]______                          _______[I13]________
             /                  \                        /                    \
        __[I8]__             __[I9]__                __[I10]__              __[I11]__
       /         \          /         \            /          \            /         \
     [I0]       [I1]     [I2]        [I3]        [I4]        [I5]        [I6]       [I7]
    /   \      /   \    /    \      /    \      /    \      /    \      /    \     /    \
  [0]   [1]  [2]   [3] [4]   [5]  [6]    [7]  [8]    [9]  [A]    [B]  [C]    [D]  [E]   [F]

  # Leafs of BlockHash tree
  [0] - version               [1] - chain_id            [2] - height        [3] - time
  [4] - last_block_id         [5] - last_commit_hash    [6] - data_hash     [7] - validators_hash
  [8] - next_validators_hash  [9] - consensus_hash      [A] - app_hash      [B] - last_results_hash
  [C] - evidence_hash         [D] - proposer_address
```

params

| Type                              | Field Name        | Description                  |
| --------------------------------- | ----------------- | ---------------------------- |
| `{bytes,bytes,bytes,bytes,bytes}` | multi_store_proof | A struct `multi_store_proof` |

return values

| Type                   | Field Name | Description   |
| ---------------------- | ---------- | ------------- |
| `bytes`, fix size = 32 | result     | The `AppHash` |

1. **_[g]_** = [merkle_leaf_hash](#merkle_leaf_hash)(0x066f7261636c6520 + [sha256](#utility-functions)([sha256](#utility-functions)(`multi_store_proof.oracle_iavl_state_hash`))) // calculate double sha256 of multi_store_proof.oracle_iavl_state_hash and then prepend with oracle prefix (uint8(6) + "oracle" + uint8(32)) and then calculate merkle_leaf_hash

2. **_[ρ4]_** = [merkle_inner_hash](#merkle_inner_hash)(**_[g]_**, `multi_store_proof.params_stores_merkle_hash`**_[h]_**)

3. **_[ρ6]_** = [merkle_inner_hash](#merkle_inner_hash)(`multi_store_proof.main_and_mint_stores_merkle_hash`**_[ρ3]_**, **_[ρ4]_**)

4. **_[ρ9]_** = [merkle_inner_hash](#merkle_inner_hash)(`multi_store_proof.acc_to_gov_stores_merkle_hash`**_[ρ5]_**, **_[ρ6]_**)

5. `AppHash` = [merkle_inner_hash](#merkle_inner_hash)(**_[ρ9]_**, `multi_store_proof.slashing_to_upgrade_stores_merkle_hash`**_[ρ10]_**)

6. return `AppHash`

```python3
def get_app_hash(multi_store_proof: bytes) -> bytes:
    acc_to_gov_stores_merkle_hash = multi_store_proof[0:32]  # [ρ5]
    main_and_mint_stores_merkle_hash = multi_store_proof[32:64]  # [ρ3]
    oracle_iavl_state_hash = multi_store_proof[64:96]  # [g]
    params_stores_merkle_hash = multi_store_proof[96:128]  # [h]
    slashing_to_upgrade_stores_merkle_hash = multi_store_proof[128:160]  # [ρ10]
    return merkle_inner_hash(  # [AppHash]
        merkle_inner_hash(  # [ρ9]
            acc_to_gov_stores_merkle_hash,  # [ρ5]
            merkle_inner_hash(  # [ρ6]
                main_and_mint_stores_merkle_hash,  # [ρ3]
                merkle_inner_hash(
                    merkle_leaf_hash(  # [ρ4]
                        # oracle prefix (uint8(6) + "oracle" + uint8(32))
                        bytes.fromhex("066f7261636c6520")
                        + sha256(sha256(oracle_iavl_state_hash))
                    ),  # [6]
                    params_stores_merkle_hash,  # [ρ7]
                ),
            ),
        ),
        slashing_to_upgrade_stores_merkle_hash,  # [ρ10]
    )
```

#### recover_signer

This fucntion receive a struct `tm_signature` as an input and return a validator's public key by using [ecrecover](#utility-functions).

params

| Type                           | Field Name   | Description                 |
| ------------------------------ | ------------ | --------------------------- |
| `{bytes,bytes,u8,bytes,bytes}` | tm_signature | A struct `tm_signature`     |
| `bytes`                        | block_hash   | The block hash of BandChain |

return values

| Type    | Field Name | Description                   |
| ------- | ---------- | ----------------------------- |
| `bytes` | public_key | The pulbic key of a validator |

<strong>Example implementation</strong>

```python3
def recover_signer(
    r: bytes,
    s: bytes,
    v: int,
    signed_data_prefix: bytes,
    signed_data_suffix: bytes,
    block_hash: bytes
) -> bytes:
    return ecrecover(sha256(signed_data_prefix + block_hash + signed_data_suffix), r, s, v)
```

#### get_parent_hash

This fucntion receive a struct `iavl_merkle_path` and data_subtree_hash bytes as inputs and return the parent hash of them.

params

| Type                      | Field Name        | Description                     |
| ------------------------- | ----------------- | ------------------------------- |
| `{bool,u8,u64,u64,bytes}` | iavl_merkle_path  | A struct `iavl_merkle_path`     |
| `bytes`                   | data_subtree_hash | The hash of a node in IAVL tree |

return values

| Type                      | Field Name        | Description                     |
| ------------------------- | ----------------- | ------------------------------- |
| `{bool,u8,u64,u64,bytes}` | iavl_merkle_path  | A struct `iavl_merkle_path`     |
| `bytes`                   | data_subtree_hash | The hash of a node in IAVL tree |

<strong>Example implementation</strong>

```python3
def get_parent_hash(
    is_data_on_right: bool,
    subtree_height: int,
    subtree_size: int,
    subtree_version: int,
    sibling_hash: bytes,
    data_subtree_hash: bytes
) -> bytes:
    left_subtree = sibling_hash if is_data_on_right else data_subtree_hash
    right_subtree = data_subtree_hash if is_data_on_right else sibling_hash
    return sha256(
        bytes([(subtree_height << 1) & 255]) +
        encode_varint_signed(subtree_size) +
        encode_varint_signed(subtree_version) +
        bytes([32]) +
        left_subtree +
        bytes([32]) +
        right_subtree
    )
```

#### relay_block

This function relays a new `oracle module` **_[B]_** hash to the `Bridge`.

params

| Type                                    | Field Name                | Description                          |
| --------------------------------------- | ------------------------- | ------------------------------------ |
| `u64`                                   | block_height              | Block height of BancChain            |
| `{bytes,bytes,bytes,bytes,bytes}`       | multi_store_proof         | A struct `multi_store_proof`         |
| `{bytes,bytes,bytes,bytes,bytes,bytes}` | block_header_merkle_parts | A struct `block_header_merkle_parts` |
| `[{bytes,bytes,u8,bytes,bytes}]`        | signatures                | An array of struct `signatures`      |

return values

```
no return value
```

1. Calculate `AppHash` by calling [get_app_hash](#get_app_hash)(`multi_store_proof`).

2. Calculate `BlockHash` by calling [get_block_header](#get_block_header)(`block_header_merkle_parts`, `app_hash`, `block_height`).

3. Loop recover every signature from `signatures`. In each round we will get the public key of the validator who have signed on the hash of `signed_data_prefix`, `block_hash` and `signed_data_suffix`.

   - Check that there is no repeated public key.
   - Read validator's voting power from storage [voting_powers](#voting_powers) by the public key that just recovered. If the public key is not found then the voting power should be 0.
   - Accumulate the voting power

4. Check that the accumulate of voting power should be greater than or equal 2/3 of the [total_validator_power](#total_validator_power).

5. Save the `oracle mudule` **_[B]_** hash to the storage [oracle_state](#oracle_state).

<strong>Example implementation</strong>

```python3
def relay_block(
    block_height: int,
    multi_store_proof: bytes,
    block_header_merkle_parts: bytes,
    signatures: bytes,
) -> None:
    obi = PyObi(
      """
      [
          {
              r: bytes,
              s: bytes,
              v: u8,
              signed_data_prefix: bytes,
              signed_data_suffix: bytes
          }
      ]
      """
    )
    app_hash = get_app_hash(multi_store_proof)
    block_hash = get_block_header(block_header_merkle_parts, app_hash, block_height)
    recover_signers = [recover_signer(sig["r"], sig["s"], sig["v"], sig["signed_data_prefix"], sig["signed_data_suffix"], block_hash) for sig in obi.decode(signatures)]
    sum_voting_power = 0
    signers_checking = set()
    for signer in recover_signers:
        if signer in signers_checking:
            revert(f"REPEATED_PUBKEY_FOUND: {signer.hex()}")

        signers_checking.add(signer)
        sum_voting_power += validator_powers[signer]

    if sum_voting_power * 3 <= total_validator_power.get() * 2:
        revert("INSUFFICIENT_VALIDATOR_SIGNATURES")

    oracle_state[block_height] = multi_store_proof[64:96]
```

#### verify_oracle_data

This function verifies that the given data is a valid data on BandChain as of the given block height.

params

| Type                                                              | Field Name                        | Description                                                                                                       |
| ----------------------------------------------------------------- | --------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `u64`                                                             | block_height                      | Block height of BancChain                                                                                         |
| `{{string,u64,bytes,u64,u64},{string,u64,u64,u64,u64,u32,bytes}}` | request_packet_and_respond_packet | A struct or a tuple of `request_packet` and `response_packet`                                                     |
| `u64`                                                             | version                           | Lastest block height that the data node was updated                                                               |
| `[{bool,u8,u64,u64,bytes}]`                                       | iavl_merkle_paths                 | An array of `iavl_merkle_path` which is the merkle proof that shows how the data leave is part of the oracle iAVL |

return values

| Type                                                              | Field Name                        | Description                                                   |
| ----------------------------------------------------------------- | --------------------------------- | ------------------------------------------------------------- |
| `{{string,u64,bytes,u64,u64},{string,u64,u64,u64,u64,u32,bytes}}` | request_packet_and_respond_packet | A struct or a tuple of `request_packet` and `response_packet` |

```text
n is the height of IAVL merkle tree.

H(0) is sha256(bytes([0]) + bytes([2]) + encode_varint_signed(version) + bytes([9]) + b"\xff" + request_id.to_bytes(8,'big') + bytes([32]) + sha256(obi_encode(request_packet_and_respond_packet)))

H(n) is an oracle module root hash from the previous diagram.

C(i) is a corresponding node to H(i) where i ∈ {0,1,2,...,n-1}.

H(i+1) is get_parent_hash(C(i), H(i)).

                      _______________[H(n)]_______________
                    /                                      \
        _______[H(n-1)]______                             [C(n-1)]
      /                      \                          /        \
  [C(n-2)]                    \                       ...        ...
  /        \                    .
...        ...                   .
                                  .
                                  \
                                    \
                          _______[H(2)]______
                        /                    \
                      [H(1)]                 [C(1)]
                    /        \             /        \
                 [H(0)]     [C(0)]       ...        ...
                 [value]
```

1. Read the `oracle module` **_[B]_** hash from the given `block_height` to check if it is available or not.

   - If the `oracle module` **_[B]_** hash is not available for the given `block_height` then `revert`
   - Else contiune

2. Calculate `H(0)`

3. Calculate the hash of parent node of H(i) and C(i) hash by hashing from bottom to the top according to [`merkle tree`](https://en.wikipedia.org/wiki/Merkle_tree) hashing scheme.

   - For i ∈ {0,1,2,...,n-1},`H(i+1)` = [get_parent_hash](#get_parent_hash)(C(i), H(i))

4. Check that `H(n)` must equal to `oracle module` **_[B]_** hash that just read from the storage in step 1.

   - If `H(n)` != `oracle module` **_[B]_** hash then `revert`
   - Else continue

5. return `request_packet_and_respond_packet`

<strong>Example implementation</strong>

```python3
def verify_oracle_data(
    block_height: int,
    request_packet_and_respond_packet: bytes,
    version: int,
    iavl_merkle_paths: bytes
) -> dict:
    oracle_state_root = oracle_state[block_height]
    if oracle_state_root == None:
        revert("NO_ORACLE_ROOT_STATE_DATA")

    packet = PyObi(
        """
        {
            req: {
                client_id: string,
                oracle_script_id: u64,
                calldata: bytes,
                ask_count: u64,
                min_count: u64
            },
            res: {
                client_id: string,
                request_id: u64,
                ans_count: u64,
                request_time: u64,
                resolve_time: u64,
                resolve_status: u32,
                result: bytes
            }
        }
        """
    ).decode(request_packet_and_respond_packet)

    # H(0)
    current_merkle_hash = sha256(
        # Height of tree (only leaf node) is 0 (signed-varint encode)
        bytes([0])
        + bytes([2])  # Size of subtree is 1 (signed-varint encode)
        + encode_varint_signed(version)
        +
        # Size of data key (1-byte constant 0x01 + 8-byte request ID)
        bytes([9])
        + b"\xff"  # Constant 0xff prefix data request info storage key
        + packet["res"]["request_id"].to_bytes(8, "big")
        + bytes([32])  # Size of data hash
        + sha256(request_packet_and_respond_packet)
    )

    len_merkle_paths = PyObi(
        """
        [
            {
                is_data_on_right: bool,
                subtree_height: u8,
                subtree_size: u64,
                subtree_version: u64,
                sibling_hash: bytes
            }
        ]
        """
    ).decode(iavl_merkle_paths)

    # Goes step-by-step computing hash of parent nodes until reaching root node.
    for path in len_merkle_paths:
        # H(i+1) = get_parent_hash(C(i), H(i))
        current_merkle_hash = iavl_merkle_path.get_parent_hash(
            path["is_data_on_right"],
            path["subtree_height"],
            path["subtree_size"],
            path["subtree_version"],
            path["sibling_hash"],
            current_merkle_hash,
        )

    # Verifies that the computed Merkle root matches what currently exists.
    # Compare H(n) and [B]
    if current_merkle_hash != oracle_state_root:
        revert("INVALID_ORACLE_DATA_PROOF")

    return packet
```

#### relay_and_verify

This function performs oracle state relay and oracle data verification in one go. The caller submits the obi encoded proof and receives back the decoded data, ready to be validated and used.

params

| Type    | Field Name | Description                                                        |
| ------- | ---------- | ------------------------------------------------------------------ |
| `bytes` | proof      | The obi encoded data for oracle state relay and data verification. |

return values

| Type                                                              | Field Name                        | Description                                                   |
| ----------------------------------------------------------------- | --------------------------------- | ------------------------------------------------------------- |
| `{{string,u64,bytes,u64,u64},{string,u64,u64,u64,u64,u32,bytes}}` | request_packet_and_respond_packet | A struct or a tuple of `request_packet` and `response_packet` |

1. Decode the `proof` using obi into 7 elements which are `block_height`, `multi_store_proof`, `block_header_merkle_parts`, `signatures`, `request_packet_and_respond_packet`, `version` and `iavl_merkle_paths`.

2. Relay the `oracle module` **_[B]_** to the state by call the function [relay_block](#relay_block) with `block_height`, `multi_store_proof`, `block_header_merkle_parts` and `signatures` as parameters.

3. Return the result from calling function [verify_oracle_data](#verify_oracle_data) with `block_height`,`request_packet_and_respond_packet`, `version` and `iavl_merkle_paths` as parameters.

<strong>Example implementation</strong>

```python3
def relay_and_verify(proof: bytes) -> dict:
    proof_dict = PyObi(
        """
        {
            block_height: u64,
            multi_store_proof: bytes,
            block_header_merkle_parts: bytes,
            signatures: bytes,
            request_packet_and_respond_packet: bytes,
            version: u64,
            iavl_merkle_paths: bytes
        }
        """
    ).decode(proof)

    relay_block(
        proof_dict["block_height"],
        proof_dict["multi_store_proof"],
        proof_dict["block_header_merkle_parts"],
        proof_dict["signatures"],
    )

    return verify_oracle_data(
        proof_dict["block_height"],
        proof_dict["request_packet_and_respond_packet"],
        proof_dict["version"],
        proof_dict["iavl_merkle_paths"],
    )
```

#### get_latest_response

This function recevie the representation of a [request_packet](#request_packet) struct which can struct, tuple, dict, bytes, etc and returns the latest corresponse [response_packet](#response_packet) that was relayed.

params

| Type                         | Field Name     | Description                                                                                                |
| ---------------------------- | -------------- | ---------------------------------------------------------------------------------------------------------- |
| `{string,u64,bytes,u64,u64}` | request_packet | The representation of a [request_packet](#request_packet) struct which can struct, tuple, dict, bytes, etc |

return values

| Type                                 | Field Name      | Description                                                                                                  |
| ------------------------------------ | --------------- | ------------------------------------------------------------------------------------------------------------ |
| `{string,u64,u64,u64,u64,u32,bytes}` | response_packet | the representation of a [response_packet](#response_packet) struct which can struct, tuple, dict, bytes, etc |

1. Read the latest corresponse [response_packet](#response_packet) to the [request_packet](#request_packet) from the storage [requests_cache](#requests_cache).

   - If there is no [response_packet](#response_packet) then return empty.
   - Else return the representation of [response_packet](#response_packet) such as struct, dict, tuple, bytes, etc.

<strong>Example implementation</strong>

```python3
def get_latest_response(self, encoded_request: bytes) -> dict:
    res = self.requests_cache[encoded_request]
    if not res:
        return None
    return PyObi(
        """
        {
            client_id: string,
            request_id: u64,
            ans_count: u64,
            request_time: u64,
            resolve_time: u64,
            resolve_status: u32,
            result: bytes
        }
        """
    ).decode(res)
```

#### relay

This function performs oracle state relay and oracle data verification in one go. After that, the results will be recorded to the state by using the OBI encoded of RequestPacket as key.

params

| Type    | Field Name | Description                                                        |
| ------- | ---------- | ------------------------------------------------------------------ |
| `bytes` | proof      | The obi encoded data for oracle state relay and data verification. |

return values

```
no return value
```

1. Call [relay_and_verify](#relay_and_verify) with `proof` as an input to relay the `oracle module` **_[B]_** hash first and then verify that the [request_packet](#request_packet) and the [response_packet](#response_packet) are really exist on BandChain.

   - If the calling [relay_and_verify](#relay_and_verify) is fail then the transaction will be `revert` otherwise continue

2. Get the current latest [response_packet](#response_packet) by calling [get_latest_response](#get_latest_response)(obi_encode(request_packet)).

3. Check that the `resolve_time` of the new [response_packet](#response_packet) is newer than the current latest [response_packet](#response_packet).

   - If the `resolve_time` of the new [response_packet](#response_packet) is greater than the `resolve_time` of the current [response_packet](#response_packet) then continue.
   - Else `revert`

4. Check that the `resolve_status` of the new [response_packet](#response_packet) of is 1 (0=Open, 1=Success, 2=Failure, 3=Expired).

   - If `resolve_status` == 1 then continue
   - Else `revert`

5. Save new [response_packet](#response_packet) into the storage [requests_cache](#requests_cache) by using obi_encode([response_packet](#response_packet)) as a key.

<strong>Example implementation</strong>

```python3
def relay(proof: bytes) -> None:
    packet = relay_and_verify(proof)
    req = packet["req"]
    res = packet["res"]

    req_key = PyObi(
        """
            {
                client_id: string,
                oracle_script_id: u64,
                calldata: bytes,
                ask_count: u64,
                min_count: u64
            }
        """
    ).encode(packet["req"])

    prev_res = get_latest_response(req_key)
    prev_resolve_time = (
        (
            prev_res.get("resolve_time", 0)
            if isinstance(prev_res.get("resolve_time", 0), int)
            else 0
        )
        if isinstance(prev_res, dict)
        else 0
    )

    if prev_resolve_time >= res["resolve_time"]:
        revert("REQUEST_HAS_OUTDATED")

    if res["resolve_status"] != 1:
        revert("FAIL_REQUEST_IS_NOT_SUCCESSFULLY_RESOLVED")

    requests_cache[req_key] = PyObi(
        """
        {
            client_id: string,
            request_id: u64,
            ans_count: u64,
            request_time: u64,
            resolve_time: u64,
            resolve_status: u32,
            result: bytes
        }
        """
    ).encode(res)
```
