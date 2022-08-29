---
id: checkpoint-sync-concept
title: "Checkpoint sync: Conceptual overview"
sidebar_label: "Checkpoint sync"
---
import CheckpointSyncPresent from '@site/static/img/checkpoint-sync-present.png';
import CheckpointSyncAbsent from '@site/static/img/checkpoint-sync-absent.png';
import NetworkPng from '@site/static/img/network.png';
import BlockchainSimplified from '@site/static/img/blockchain-simplified.png';
import ClientStackPng from '@site/static/img/client-stack.png';
import EpochsBlocks from '@site/static/img/epochs-blocks.png';
import Finality from '@site/static/img/finality.png';

import {HeaderBadgesWidget} from '@site/src/components/HeaderBadgesWidget.js';

# Checkpoint sync: Conceptual overview

<HeaderBadgesWidget commaDelimitedContributors="Mick,Potuz,Kasey" />

:::info Looking for the how-to?

This is a conceptual overview. See [How to sync from a checkpoint](../prysm-usage/checkpoint-sync.md) to learn how to configure checkpoint sync. 

:::

**Checkpoint sync** is a feature that significantly speeds up the initial sync between your beacon node and the Beacon Chain. With checkpoint sync configured, your beacon node will begin syncing from a recently finalized checkpoint instead of syncing from genesis. This can make installations, validator migrations, recoveries, and testnet deployments *way* faster.

This conceptual overview will help you understand **how checkpoint sync works** and **the value that it provides to the Ethereum network**. Let's start by speed-running through some foundational concepts.

## Foundations

### Ethereum, nodes, and networks

Ethereum is a decentralized **network** of **nodes** that communicate via peer-to-peer connections. These connections are formed by computers running Ethereum's specialized client software (like Prysm). An Ethereum **node** is a running instance of Ethereum's client software. This software is responsible for running the Ethereum blockchain. There are two primary types of nodes in Ethereum: **execution nodes** and **beacon nodes**. Colloquially, a "node" refers to an execution node and beacon node working together:

<img src={ClientStackPng} /> 

Nodes can be configured to run on one of Ethereum's many networks. Production apps and real ETH live on Ethereum Mainnet, while test networks allow protocol and application developers to test new features using testnet ETH. See [Nodes and networks](nodes-networks.md) for a more detailed refresher on these topics.


### Blockchain data structure

Ethereum's "world state" is stored in a blockchain data structure. In the most rudimentary sense, a blockchain is a [linked list](https://en.wikipedia.org/wiki/Linked_list) on steroids. Like a linked list, Ethereum's blockchain is a sequence of items that starts with a "tail" and ends at its "head". Ethereum's tail is called its **genesis block** - the first block that was ever created. Its most recently added block is referred to as the "head of the chain":

<img src={BlockchainSimplified} />

Ethereum's nodes work together to process one batch of transactions at a time. These batches of transactions are Ethereum's blocks. As users and applications submit transactions to the network, new blocks are created to contain them. These blocks are tentatively appended to the head of the chain before becoming permanently enshrined in Ethereum Mainnet.

### Blocks, epochs, and slots

It can be helpful to think of Ethereum as a "world computer" with a processor and hard drive. Its processor is the [Ethereum Virtual Machine](https://ethereum.org/en/developers/docs/evm/) (EVM) and its hard drive is the blockchain underlying Ethereum Mainnet. Both of these components live on every Ethereum node.

Every 12 seconds, Ethereum's processor dequeues a collection of transactions and attempts to organize them into a block. These 12-second periods are called **slots**. `1 slot = 12 seconds` is a rule that's hardcoded within the [Beacon Chain protocol specification](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#time-parameters-1). 

Every slot is an opportunity for one new block of transactions to be proposed and tentatively accepted by the Ethereum network. Every 32 slots (or 384 seconds), Ethereum has an opporunity to upgrade a batch of slots from "proposed" to "finalized". Ethereum's blocks are finalized *at most* once every 32 slots. These 32-slot timespans are called **epochs**:

<img src={EpochsBlocks} />

 The above diagram illustrates [Epoch 1](https://ethscan.org/epoch/1) on Ethereum Mainnet. This epoch, like all other epochs, contains 32 slots. Every slot contains one block. Some blocks have been approved by the network. The block in this epoch's fourth slot, or the zero-based [slot 35 of the chain](https://ethscan.org/block/35), included [154 transactions](https://etherchain.org/block/0x8d3f027beef5cbd4f8b29fc831aba67a5d74768edca529f5596f07fd207865e1#pills-txs). One of these transactions was [a transfer of 3.7 ETH](https://etherchain.org/tx/0x9f421378c2cd87fcad6185cf2690881857b077a28e46d89a49240900a7a9836e).


### Justification, finality, and checkpoints

**Justification** is the process of marking a block as *tentatively canonical*. **Finalization** is the process of upgrading a block from *tentatively canonical* to *canonical*. When a block is finalized, its transactions are finalized, and the probability of transaction reversal is near-zero because it would require social consensus and a hard fork.

Whenever a new epoch begins, a new set of blocks can be justified, and an older set of justified blocks can be finalized. The Ethereum network uses **checkpoints** to periodically perform these promotions. A checkpoint consists of a **block** and an **epoch number**. In most cases, a checkpoint will be the first block of its epoch <a class="footnote" href='#footnote-1'>[12]</a>.

A block is **justified** after:

 1. Its epoch has passed, and either
 1. More than 2/3 of attesting validators agree that the block is a justified checkpoint, or
 2. A **future checkpoint** becomes justified, and the block is an ancestor of that newly justified checkpoint.

A block is **finalized** after either:

 1. The block is a **justified checkpoint** and 2/3 of attesting validators identify another justified checkpoint in a future epoch, or
 2. A **future epoch's justified checkpoint** becomes finalized, and the block is an ancestor of that newly finalized checkpoint.

Epochs are used to identify checkpoints, and checkpoints are used to implement the notions of justification and finality. Justification and finalization occur *at most* one epoch at a time. Ethereum's current head is the most recently justified checkpoint:

<img src={Finality} />

The above epochs can be described as follows:

1. **Epoch 1**: All blocks are finalized.
2. **Epoch 2**: The first block is finalized, because the following epoch's checkpoint was justified. The rest of the blocks are justified. This epoch's first block is the most recently finalized checkpoint.
3. **Epoch 3**: All blocks are proposed, only the first has been justified. This epoch's first block is also the most recently justified checkpoint, and thus, is the head of the chain.
4. **Epoch 4**: Blocks are being proposed, none are justified. Its first block could become a justified checkpoint, but only after the epoch ends.

Let's imagine that Bob wants to send Alice some ETH. In the happy scenario, Bob's transaction would flow through the following (oversimplified) transaction lifecycle:

 1. **Transaction signed**: Bob signs a transaction that moves ETH from his wallet to Alice's wallet using the private key associated with his wallet.
 2. **Transaction submitted**: Bob submits this transaction to the Ethereum network. Theoretically, all nodes receive it.
 3. **Proposer selected**: The Ethereum network randomly selects a validator node to fill the current epoch's current slot with a new block. This validator node will be the only node in the world that's allowed to propose a block into this slot.  
 4. **Block created**: This randomly selected block proposer dequeues a batch of transactions from its internal transaction queue, verifies the legitimacy of each dequeued transaction, and builds a new block that contains Bob's transaction.
 5. **Block proposed**: The block proposer broadcasts this proposed new block to peer nodes.
 6. **Attesters selected**: The Ethereum network randomly selects a small committee of other validator nodes to attest to the legitimacy of the proposed block and the transactions it contains.
 7. **Block justified**: After a supermajority of attesters identify a future justified checkpoint, this block is marked as justified along with all of its ancestor blocks.
 8. **Block finalized**: After a supermajority of attesters identify another future justified checkpoint, the previous checkpoint is marked as finalized along with all of its ancestor blocks.


## Checkpoint sync

### Syncing without checkpoint sync

Beacon nodes maintain a local copy of the Ethereum's [Beacon Chain](https://ethereum.org/en/upgrades/beacon-chain/). When you tell Prysm's beacon node to start running for the first time, Prysm will "replay" the history of the Beacon Chain starting with the very first block (the [genesis block](https://beaconscan.com/slots?epoch=0)), fetching the oldest blocks from peers until the entire chain has been downloaded:

<img src={CheckpointSyncAbsent} /> 

This process can take a very long time - on the magnitude of days to weeks. 

### Syncing with checkpoint sync

Checkpoint sync speeds things up by telling your beacon node to sync from a **recently finalized checkpoint**, allowing it to skip over the majority of the Beacon Chain's history:

<img src={CheckpointSyncPresent} /> 

The process of retrieving historical blocks before the starting block is called "backfilling", and will be supported in a future Prysm release. **Backfilling isn't required to run a validator** - it's only required if you want to run an archive node, serve peer node requests for historical blocks, or query chain history through your local beacon node's Beacon API.


## Frequently asked questions

**Where can I learn more about checkpoint sync and related concepts?** <br/>

 - [Gasper](https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/gasper/) describes the concept of finality in the context of Gasper, the consensus mechanism securing proof-of-stake Ethereum
 - [What happens after finality in Ethereum PoS](https://hackmd.io/@prysmaticlabs/finality) provides an in-depth explanation of finality
 - [Proof of Stake](https://ethereum.org/pt/developers/docs/consensus-mechanisms/pos/) is a great place to learn about Ethereum Proof of Stake, finality, and checkpoints
 - [Checkpoint Sync Safety](https://www.symphonious.net/2022/05/21/checkpoint-sync-safety/) by Adrian Sutton
 - [How to: Checkpoint Sync](https://notes.ethereum.org/@launchpad/checkpoint-sync) by members of the Ethereum Foundation
 - [WS sync in practice](https://notes.ethereum.org/@djrtwo/ws-sync-in-practice) by Danny Ryan


----

Footnotes:

<strong id='footnote-1'>1</strong>. A checkpoint block may end up being in the middle of the previous epoch. Note that the proposer selected to fill the first slot of an epoch may not produce a valid block. If this happens, attesters will identify the candidate checkpoint by working backwards through the previous epoch until they find a valid block. For example: slot 20 of the previous epoch may be used as a checkpoint if blocks from slots 21...32 are missing. <br />


<RequestUpdateWidget />