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
- [Bridge's storages](#bridge's-storages)
  - [total_validator_power](#total_validator_power)
  - [validator_powers](#validator_powers)
  - [oracle_state](#oracle_state)
  - [requests_cache](#requests_cache)
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
    ...         ...                   .
                                       .
                                        \
                                         \
                                _______[H(2)]______
                              /                    \
                           [H(1)]                 [C(1)]
                         /        \             /        \
                     [value]     [C(0)]       ...         ...
  ```

## Bridge's storages

#### total_validator_power

xxxxxxxxxxxxx

#### validator_powers

xxxxxxxxxxxxx

#### oracle_state

xxxxxxxxxxxxx

#### requests_cache

xxxxxxxxxxxxx

## Bridge's functions

#### get_total_validator_power

xxxxxxxxxxxxx
