# None EVM Bridge specification

**Notice**: This document is a work-in-progress for researchers and implementers.

## Table of contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

**Table of Contents**

- [Introduction](#introduction)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

At Band Protocol, we provide a way for dApps to access off-chain information through our decentralized oracle. As part of that offering, we also provide a lite client for anyone who requested data from our oracle to verify the validity of the result they received. An instance of this client exists on each of the blockchains to which Band has integrated with.
In this article, we examine the verification process performed by our lite client on EVM-compatible blockchains with Solidity. When someone submits a verification request to our lite client, they must also send in the encoded result they got from our oracle. That result is not just the data they requested, but also contains information on the request itself as well as the associated response.
The lite client’s purpose is then to use that information to show that the data the user requested exists on BandChain, thus verifying the oracle result’s validity. Before looking more into how the client does that, let’s take a step back and examine the oracle data request flow that precedes it.
