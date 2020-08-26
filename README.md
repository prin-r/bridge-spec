# None EVM Bridge specification

**Notice**: This document is a work-in-progress for researchers and implementers.

## Table of contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

**Table of Contents**

- [Introduction](#introduction)
  - [Lite Client Verification Process](#lite-client-verification-process)
- [Utility Functions](#utility-functions)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

At Band Protocol, we provide a way for other blockchains to access off-chain information through our decentralized oracle. As part of that offering, we also provide a specification of lite client verification for anyone who requested data from our oracle to verify the validity of the result they received. We call the instance of lite client that is existed on other blockchains `Bridge`. The implementation of `Bridge` can be a smart contract (additional logic published by user) or a module (build in logic of a blockchain).

### Lite Client Verification Process

Once the other blockchain receives the oracle result, they proceed to verify that the result actually comes from BandChain. They do this by submitting a verification request to the `Bridge`. The aim of this process is to ensure that the data received is actually part of BandChain’s state and is signed by a sufficient number of BandChain’s block validators.

This process can be divided into two unrelated sub-processes.

- 1. `relay_oracle_state`: Verify that a root hash of the oracle module really exist on BandChain at a specific block and then save that root hash into `Bridge`'s state. This process requires the signatures of several validators signed on the block hash in which everyone who signs must have a total voting power greater than or equal to two-thirds of the entire voting power. The block hash is made up of multiple values that come from the BandChain state, where oracle module root hash is one of them.

  ```text
                                 __ [BlockHash] __
                       _________|                 |___________
                      |                                       |
                   [3α]                                     [3ß]
             ____/      \______                      _____/      \______
            |                  |                    |                   |
         [2α]                [2ß]                 [2Γ]                [2Δ]
        /    \              /    \               /    \              /    \
    [1α]      [1ß]      [1Γ]      [1Δ]       [1ε]      [1ζ]        [C]    [D]
    /  \      /  \      /  \      /  \       /  \      /  \
  [0]  [1]  [2]  [3]  [4]  [5]  [6]  [7]   [8]  [9]  [A]  [B]
                                                      |
                                                      |
                                                      |
                                  ________________[AppHash]_______________
                                 /                                        \
                       _______[ɩ9]______                          ________[ɩ10]________
                      /                  \                       /                     \
                 __[ɩ5]__             __[ɩ6]__              __[ɩ7]__               __[ɩ8]__
                /         \          /         \           /         \            /         \
              [ɩ1]       [ɩ2]     [ɩ3]        [ɩ4]       [i]        [j]          [k]        [l]
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

- 2. `verify_oracle_data`:

## Introduction
