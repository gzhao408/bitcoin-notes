# Lifecycle of a Transaction

## Table of Contents

- [Lifecycle of a Transaction](#Lifecycle-of-a-Transaction)
  - [Transaction Creation](#Transaction-Creation)
    - [Transaction Creation through Bitcoin Core Wallet](#Transaction-Creation-through-Bitcoin-Core-Wallet)
    - [Transaction Children](#Transaction-Children)
  - [Validation and Submission to Mempool](#Validation-and-Submission-to-Mempool)
    - [Mempool Validation](#Mempool-Validation)
      - [Context-Free Non-Script Checks](#Context-Free-Non-Script-Checks)
      - [Contextual Non-Script Checks](#Contextual-Non-Script-Checks)
      - [Signature and Script Checks](#Signature-and-Script-Checks)
    - [Submission to Mempool](#Submission-to-Mempool)
  - [P2P Transaction Relay](#P2P-Transaction-Relay)
  - [Inclusion in a Block](#Inclusion-in-a-Block)
    - [Transaction Announcement and Broadcast](#Transaction-Announcement-and-Broadcast)
    - [Transaction Request and Download](#Transaction-Request-and-Download)
    - [Orphans](#Orphans)
    - [Mining](#Mining)
    - [Block Relay](#Block-Relay)
    - [Block Validation](#Block-Validation)
    - [State Changes and Persistence to Disk](#State-Changes-and-Persistence-to-Disk)
    - [Wallet Updates](#Wallet-Updates)
  - [Other?](#Other)

## Transaction Creation

The process of creating a transaction does not need to be done on a node. For example, users can
generate their transactions on another wallet and/or entirely offline, and then submit raw
transactions to their Bitcoin Core node via the
[`sendrawtransaction`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/rpc/rawtransaction.cpp#L825)
RPC.

### Transaction Creation through Bitcoin Core Wallet

The Bitcoin Core wallet also allows users to create transactions with varying levels of
customization. The steps are roughly as follows:

1. A payee [provides an invoice or
   destination](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/wallet/wallet.cpp#L2103)
to pay to, committing to the spending conditions. This helps the wallet determine the
[outputs](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/outputtype.h#L23-L26),
which comprise the payments.

2. The wallet is calculates feerate estimates, output types, and other options based on the user's input,
   pre-configured defaults and preferences, historical blocks and current mempool contents (queried from
the node).

3.  The transaction is
["funded"](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/wallet/spend.cpp#L373)
using the set of
[UTXOs](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/wallet/spend.cpp#L67)
available in the wallet; these comprise the inputs. A change output [may or may not be
added](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/wallet/spend.cpp#L767-L778)
to the transaction.

4. Some combination of signatures and/or scripts is added to each input to satisfy their
   corresponding outputs' spending conditions. This process may involve various steps based on how many
signatures are needed and where the key(s) are stored.

### Transaction Children

A child transaction (one spending the outputs of this transaction) can be created as soon as the
transaction has a txid - this can happen before it's even signed, which the Lightning Network takes
advantage of in order to open payment channels.

On the other hand, among other limitations, children of coinbase transactions would need to wait
[100
blocks](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/wallet/wallet.cpp#L2881)
before becoming consensus-valid. Transactions outputs can also specify a timelock before which they
cannot be spent. The Bitcoin Core wallet also applies [various
filters](https://github.com/bitcoin/bitcoin/blob/6312b8370c5d3d9fb12eab992e3a32176d68f006/src/wallet/spend.cpp#L67)
to the set of existing UTXOs it is aware of in order to limit what it considers available funds to
spend from.

## Validation and Submission to Mempool

Regardless of where the transaction originated from, before being broadcast to the network, the
transaction must be accepted into the node's [mempool](). The mempool is designed to be a cache of
unconfirmed transactions to help [select transactions for inclusion in a
block](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/miner.cpp#L308)
based on fees and manual miner prioritization. For non-mining nodes, keeping a mempool helps
increase block validation performance and aid transaction relay. In addition to enforcing consensus
rules Bitcoin Core nodes apply a set of rules known as mempool _policy_ (also called "standardness"
rules).

### Mempool Validation

Beyond applying consensus rules correctly, the primary consideration in mempool validation is DoS
prevention. Malicious nodes can create fake transactions very cheaply (both monetarily and
computationally); there are no Proof of Work requirements on transactions.  As such, [mempool
validation](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L426)
is designed to fail fast and protect the node from expending any unnecessary resources.

We might categorize transaction validation checks in a few different ways:

- Consensus vs Policy: These can also be thought of as mandatory vs non-mandatory checks. They are
  not mutually exclusive, but we make all possible efforts to
  compartamentalize consensus rules to avoid making mempool logic consensus-critical.

- [Script](https://en.bitcoin.it/wiki/Script) vs Non-script: We make this distinction because script
  checking is the most computationally intensive part of transaction validation.

- Contextual vs Context-Free: The context refers to the current state of the block chain, or
  [ChainState](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.h#L566)
(thinking about the Bitcoin network as a distributed state machine). Contextual checks might require
the current block height or knowledge of the current UTXO set, while context-free checks only need
the transaction itself.

#### Context-Free Non-Script Checks

Mempoool validation in Bitcoin Core starts off with non-script checks (sometimes called
["PreChecks"](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L541),
the name of the function in which these checks run).

As a defensive strategy, the node starts with context-free and/or easily computed checks.
Here are some examples:

- None of the outputs are trying to send a value [less than 0 or greater than 21 million
  BTC](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/consensus/tx_check.cpp#L25-L27).

- The transaction [isn't a
  coinbase](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L568)
- there can't be any coinbase transactions outside of blocks.

- The transaction isn't [more than 400,000 weight
  units](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/policy/policy.cpp#L88).
It's possible for a larger transation to be consensus-valid, but it would occupy too much space in
the mempool. If we allowed these transactions, an attacker could try to cripple our mempool by
sending very large transactions that are later conflicted out by blocks.

#### Contextual Non-Script Checks

Perhaps the most obvious non-script consensus check here is to [make sure the inputs are
available](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L641-L662),
either in the current chainstate or as an output of an in-mempool transaction. Each input in the
transaction must refer to a UTXO created by previous transaction by their
txid and index in the output vector.
Rather than looking through the entire blockchain (hundreds of gigabytes stored on disk), Bitcoin
Core nodes keep a [layered
cache](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.h#L517-L541)
of the available
["coins"](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/coins.h#L30)
(a few gigabytes, much of which can be kept in memory). To make this process more efficient, coins
fetched from disk during mempool validation are [kept in
memory](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1116-L1124)
if the transaction is accepted to the mempool.  This prevents a peer from thrashing the coins cache
(also bottlenecking block validation performance) by creating invalid transactions spending random
coins - remember that malicious nodes would be able to do this without providing a valid signature.

Timelocks are also checked here - the node grabs the Median Time Past
([BIP113](https://github.com/bitcoin/bips/blob/master/bip-0113.mediawiki)) and/or block height at
the current chainstate to check transaction
[`nLockTime`](https://doxygen.bitcoincore.org/class_c_transaction.html#a54d5948c11f499b28276eab6bbfdf0c5)
and input
[`nSequence`](https://doxygen.bitcoincore.org/class_c_tx_in.html#a635deeaf3ca4e8b3e1a97054607211b9).

What happens if a new block arrives on the wire while we're in the middle of validating this
transaction? It would be unsafe for the wallet to submit transactions to the mempool while
transactions from the latest block are being removed. Luckily, the chainstate and mempool are
guarded by mutexes; they are effectively
[frozen](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1022)
for the duration of this validation session. The node won't process any new blocks until it has
finished.

#### Signature and Script Checks

Transaction [script
checks](https://doxygen.bitcoincore.org/validation_8cpp.html#a6a96a3e1e6818904fdd5f51553b7ea60) are
actually context-free; the
[`scriptSig`](https://doxygen.bitcoincore.org/class_c_tx_in.html#aba540fd902366210a6ad6cd9a18fe763)
and [`witness`](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki#specification) for
each input is paired with the
[`scriptPubKey`](https://doxygen.bitcoincore.org/class_c_tx_out.html#a25bf3f2f4befb22a6a0be45784fe57e2)
in the [corresponding
UTXO](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1469)
and passed into the script interpreter. The [script
interpreter](https://doxygen.bitcoincore.org/interpreter_8h.html) simply evaluates the series of
opcodes and data based on the arguments passed to it. One such argument is a set of [script
verification
flags](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/script/interpreter.h#L42-L143)
indicating which rules to apply during script verification.

For example, the `OP_CHECKSEQUENCEVERIFY` opcode
[repurposed](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki) `OP_NOP3`. The script
verification flag `SCRIPT_VERIFY_CHECKSEQUENCEVERIFY` instructs the script interpreter [to
interpret](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/script/interpreter.cpp#L587)
the opcode `0xb2` as the instruction to
[check](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/script/interpreter.cpp#L1760)
that the input's `nSequence` is greater than the stack value or as a no-op. At the BIP112 activation
height, [nodes
pass](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1695-L1697)
`SCRIPT_VERIFY_CHECKSEQUENCEVERIFY=1` into the script interpreter.

Another part of script validation is signature verification (indicated in a script by opcodes such
as `OP_CHECKSIGVERIFY` which is the most expensive part of transaction validation.

Transactions might have multiple signatures on various parts of the transaction. For example,
multiple parties might have contributed to funding the transaction, each signing some combination of
the inputs and/or outputs that they care about. Even in the most basic, single signature
transaction, we need to serialize and hash parts of the transaction (based on the [sighash
flag(s)](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/script/interpreter.h#L25-L35)
specified for each signature). To save the node from repetitive work, at the very start of
script checks, subparts of the transaction are [serialized, hashed, and
stored](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/script/interpreter.cpp#L1423)
in a
[`PrecomputedTransactionData`](https://doxygen.bitcoincore.org/struct_precomputed_transaction_data.html)
struct for use in signature verification.

[`MemPoolAccept`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L426)
performs two sets of script checks:
[`PolicyScriptChecks`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L917)
and
[`ConsensusScriptChecks`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L943).
The former [runs the script
interpreter](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L926)
using [consensus and policy
flags](https://doxygen.bitcoincore.org/policy_8h.html#ab28027bf27efcdd6535a13175a89ca5a) and caches
the signature result (if successful) in the [signature
cache](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/script/sigcache.cpp#L90).
The latter runs the script interpreter using [consensus flags
only](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L965)
and [caches the full validation
result](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1490),
identified by the wtxid and script verification flags, in the script cache. If a new consensus rule
is activated between now and the block in which this transaction is included, the cached result is
no longer valid, but this is easily detected based on the script verification flags. The cached
signature verification result from
[`PolicyScriptChecks`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L917)
can be used immediately in
[`ConsensusScriptChecks`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L943);
we almost never need to verify the same signature more than once!

### Submission to Mempool

The mempool primarily serves as a size-limited cache of the best candidates for inclusion in the
next block.

FIXME: Finalize()

A transaction being added to the mempool is an event that clients of
[`ValidationInterface`](https://doxygen.bitcoincore.org/class_c_validation_interface.html) are
[notified
about](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1046).
If the transaction originated from the node's wallet, the [wallet
notes](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/wallet/wallet.cpp#L1187)
that it has been added to mempool.

## P2P Transaction Relay

A node participating in transaction relay broadcasts all of the transactions in its mempool.
The primary goal in transaction relay is to propagate transactions to every node in the network in a
timely manner, but privacy and network bandwidth are also important consideration in Bitcoin.  Since
the average block interval is about 10 minutes, we'll consider a few second-delay acceptable if it
helps obfuscate transaction origins and avoids clogging up the network with redundant transaction
messages.

Technically, the Bitcoin P2P protocol specifies that transactions are communicated using a [`TX`
message](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/protocol.h#L121).
Most nodes relay transactions using a three-part dialogue:

1. The sender sends an [`INV`
   (Inventory)](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L4571)
announcing the new transaction by its txid or wtxid.

2. Nodes [manage](https://doxygen.bitcoincore.org/class_tx_request_tracker.html) various
   transaction announcements from peers to decide which transactions they are interested in and
which peer(s) to request them from. To request a transaction, a node sends a
[`GETDATA`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/protocol.h#L100)
message to a peer with a list of transactions identified by txid or wtxid.

3. The sender [responds to
   `GETDATA`s](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L1876-L1906)
by sending the full transaction in a `TX` message.

[Erlay](https://bitcoinops.org/en/topics/erlay/) would also introduce an additional method of
transaction relay.

## Inclusion in a Block

If we think of the Bitcoin network as a distributed state machine, the state is the current set of
available coins (UTXO set), while the blockchain is a ledger (or journal) of atomic state changes
batched into blocks. We can think of a transaction is an atomic state change redistributing existing
coins (or minting new coins, as per the rules agreed upon by the network) to new `scriptPubKey`s.
In the Bitcoin network, a transaction included in a valid block is a proposed state change
(non-miners ask miners to propose it on their behalf by offering a fee on their transaction).
Since v0.8, Bitcoin Core nodes have used a UTXO set to represent state and validate transactions.
To fully validate new blocks, nodes only need to consult their UTXO set and knowledge of the current
consensus rules.

### Transaction Announcement and Broadcast

With respect to privacy in transaction relay, we specifically want to hide the origin (IP address)
of a transaction. For each peer, nodes send a batch of transaction announcements at [random
intervals](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L4584-L4588)
(poisson timer with 2-second average for outbound peers and 5-second average for inbound peers).

Upon broadcasting a transaction for the first time, Bitcoin Core nodes store them in an
[_unbroadcast_](https://doxygen.bitcoincore.org/class_c_tx_mem_pool.html#a3df5ff43adfe0f407daa6fdef8965ba8)
set and rebroadcast them periodically until they see an `GETDATA` for it.

### Transaction Request and Download

Since a node typically has several peers relaying transactions, it will likely receive multiple
announcements for the same transaction, but only requests it from one peer at a time to avoid
redundant network traffic. At the same time, it's possible for a request to fail. A peer might evict
the transaction from their mempool, take too long to respond, terminate the connection, etc.
Malicious nodes may also try to take advantage of flaws in the transaction request logic to censor
or stall a transaction from being propagated to miners. In these cases, nodes must remember which
other peers announced the same transaction in order to re-request it.

Rather than responding to transaction `INV`s with `GETDATA`s immediately, nodes [store all the
announcements
received](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L2859),
batching ones corresponding to the same transaction, selecting the best candidate to request the
transaction from based on connection type and how many requests are already in flight to a specific
peer. Nodes [prefer to
download](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L1053-L1066)
from manual connections, outbound connections, and peers using WTXID relay. If the first
announcement(s) to arrive are from non-preferred peers, the node waits for other peers to announce
the same transaction for a short period of time (several seconds) before sending the request.

### Orphans

Relevant code: [src/txorphanage.h](https://doxygen.bitcoincore.org/txorphanage_8h.html)

FIXME: orphans



A signature on a transaction indicates that the owner of the private key has agreed to spend the
coins, but the transaction should be untrusted until there is network consensus that the coins have
been sent - it is trivially cheap for a private key owner to sign multiple transactions sending the
same coins. In Bitcoin, the consensus protocol requires transactions to be included in a block
containing a valid Proof of Work and accepted by the network as part of the most-work chain - it is
prohibitively expensive for a private key owner to reverse a transaction by rewriting (i.e.
recomputing Proofs of Work for) these blocks.

### Mining

Miners can start working on a block as soon as they have the previous block hash. The process of
creating a new block might look something like this:

1. The miner calls the Bitcoin Core RPC
   [`getblocktemplate`](https://developer.bitcoin.org/reference/rpc/getblocktemplate.html) to
generate a block
[_template_](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/miner.h#L28-L34)
containing a consensus-valid set of transactions and header, just mising the nonce.

2. The miner dispatches the block template to other hardware (perhaps a dedicated rack of ASICs, a
   cloud instance, or other nodes operated by mining pool members) dedicated to finding the nonce.

3. If a solution is found, the miner calls
   [`submitblock`](https://developer.bitcoin.org/reference/rpc/submitblock.html) to submit the block
to their node and broadcast it.

Bitcoin Core's [`BlockAssembler`](https://doxygen.bitcoincore.org/class_block_assembler.html)
(invoked by `getblocktemplate`) uses the mempool, sorted by ancestor score (ancestor feerate
including manual
[prioritization](https://developer.bitcoin.org/reference/rpc/prioritisetransaction.html) by the
miner), to determine which transactions to include in the next block.  This means a "package"
consisting of a 5 sat/vB parent and 100 sat/vB child may get picked over a 20 sat/vB transaction,
but not over a singular 100 sat/vB transaction with no dependencies.

### Block Relay

Once a new block is found, propagation speed is
[crucial](https://podcast.chaincode.com/2020/03/12/matt-corallo-6.html) to the decentralization of
the network. One part of this is block relay (measured by the latency between two peers), and the
other is block validation performance.

Blocks can contain megabytes worth of transactions. Nodes that keep a mempool have typically seen
[the vast majority of them already](https://www.youtube.com/watch?v=EHIuuKCm53o).

The methods for block relay are listed below, along with a description of how the transaction is
communicated with or without sending the full transaction data over P2P.

Peers negotiate which block relay communication method to use when initially establishing the
connection. Since v0.10, Bitcoin Core nodes [sync
headers-first](https://github.com/bitcoin/bitcoin/pull/4468), optimistically accepting headers (80
bytes each) to build their block chain (technically, tree, since forks, stale blocks, and invalid
blocks are possible).

As such, block validation and download overlap slightly since a few checks are done in between
downloading block headers and the rest of the block.

- High Bandwidth Compact Blocks: the sender creates a
  [BIP152](https://github.com/bitcoin/bips/blob/master/bip-0152.mediawiki) compact block comprising
only the block header, prefilled coinbase transaction, and shortids for all other transactions. If
the receiver does not recognize a transaction shortid, it [may
request](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L3434)
it through `GETBLOCKTXN`.

- Headers First: to prevent huge spikes in network traffic, senders flood [block headers
  only](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L4526)
(which is just 80 bytes). The recipient can choose to request the [full
block](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L2093)
or [compact
block](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L2107).

- `INV`/`GETDATA` Dialogue: blocks can also be announced [through `INV`
  messages](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L1813).


### Block Validation

Since consensus rules depend on block height and time (both of which can decrease in a reorg), the
consensus rules are recalculated for each block prior to validation. Regardless of whether or not
transactions have already been previously validated and accepted to the mempool, nodes check
block-wide (e.g. [total sigop
cost](https://github.com/bitcoin/bitcoin/blob/9df1906091f84d9a3a2e953a0424a88e0931ea33/src/validation.cpp#L1935),
[duplicate
transactions](https://github.com/bitcoin/bitcoin/blob/9df1906091f84d9a3a2e953a0424a88e0931ea33/src/validation.cpp#L1778-L1866),
[timestamps](https://github.com/bitcoin/bitcoin/blob/9df1906091f84d9a3a2e953a0424a88e0931ea33/src/validation.cpp#L3172-L3179),
[block subsidy
amount](https://github.com/bitcoin/bitcoin/blob/9df1906091f84d9a3a2e953a0424a88e0931ea33/src/validation.cpp#L1965-L1969))
and transaction-wide (e.g. [witness
commitments](https://github.com/bitcoin/bitcoin/blob/9df1906091f84d9a3a2e953a0424a88e0931ea33/src/validation.cpp#L3229-L3255),
availability of inputs, and [input
scripts](https://github.com/bitcoin/bitcoin/blob/9df1906091f84d9a3a2e953a0424a88e0931ea33/src/validation.cpp#L1946))
consensus rules for each block.

Script checking is parallelized in block validation to speed it up. Block transactions are checked
in order (and coins set updated which allows for dependencies within the block), and their individual
input script checks [added to a work
queue](https://github.com/bitcoin/bitcoin/blob/9df1906091f84d9a3a2e953a0424a88e0931ea33/src/validation.cpp#L1887)
that runs on a different set of independent threads while block validation continues. While failures
should be rare (creating a valid proof of work for an invalid block is quite expensive), any
consensus failure on a transaction invalidates the entire block, so no state changes are saved
until these threads successfully complete.

If the node already validated a transaction before it was included in a block, no consensus rules
have changed, and the script cache has not evicted this transaction's entry, it doesn't need to run
script checks again - it just [uses the script
cache](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1419-L1430)!

### State Changes and Persistence to Disk

Since historical block data is not needed regularly and quite large, it is stored on disk and
fetched if needed.  If configured by the user, the node keeps an
[index](https://doxygen.bitcoincore.org/class_tx_index.html) of transactions to quickly look up
their locations on disk.  Additionally, the node should be prepared for a reorg - the block may
become stale if a new most-work chain is found, and the changes to the UTXO set will need to be
undone. As such, every block has a set of corresponding [undo
data](https://doxygen.bitcoincore.org/class_c_block_undo.html).

Blocks and undo data are persisted to disk in the `blocks/` directory. Each `blkNNNNN.dat` file stores
raw block data, and its corresponding undo data is stored in a `revNNNNN.dat` file
in the same directory. If the node operator has configured a maximum memory allocation for this
node, this process will also prune blocks by oldest-first to stay within limits.

While the UTXO set is comparatively smaller than the block chain, it doesn't always fit in memory.
A node's [view of available coins]() is implemented in layers of maps from outpoints to
scriptPubKeys. Every validation session (for both [block
validation](https://github.com/bitcoin/bitcoin/blob/6312b8370c5d3d9fb12eab992e3a32176d68f006/src/validation.cpp#L2380)
on top of the current tip and unconfirmed [transaction
validation](https://github.com/bitcoin/bitcoin/blob/6312b8370c5d3d9fb12eab992e3a32176d68f006/src/validation.cpp#L427)
on top of current chainstate and mempool) creates a temporary view of the state and flushes
it to the appropriate view.

### Wallet Updates

The Bitcoin Core wallet tracks incoming and outgoing transactions and subscribes to major events by
[implementing](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/wallet/wallet.cpp#L1187-L1265)
the node's `ValidationInterface`. The most relevant information for the wallet is which coins are
available and how likely they are to remain spendable - this is quantified as
["depth"]()https://github.com/bitcoin/bitcoin/blob/55a156fca08713b020aafef91f40df8ce4bc3cae/src/wallet/spend.h#L25
by the number of confirmations on the transaction (or on a conflicting transaction, represented
using a negative number).  A UTXO made available by a transaction 100 blocks deep in the most-work
chain is considered pretty safe. On the other hand, if a newly confirmed transaction conflicts with
one that sent coins to the wallet, those coins now have a "depth" of -1 and [will not be considered
a probable source of
funds](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/wallet/spend.cpp#L94).

## Other?
