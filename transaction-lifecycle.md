# Lifecycle of a Transaction

## Table of Contents

- [Lifecycle of a Transaction](#Lifecycle-of-a-Transaction)
  - [Transaction Creation through Bitcoin Core Wallet](#Transaction-Creation-through-Bitcoin-Core-Wallet)
    - [Wallet Parameters](#Wallet-Parameters)
    - [Transaction Payload](#Transaction-Payload)
    - [Coin Selection](#Coin-Selection)
    - [Signing](#Signing)
  - [Validation and Submission to Mempool](#Validation-and-Submission-to-Mempool)
    - [Mempool Validation](#Mempool-Validation)
      - [Context-Free Non-Script Checks](#Context-Free-Non-Script-Checks)
      - [Contextual Non-Script Checks](#Contextual-Non-Script-Checks)
      - [Signature and Script Checks](#Signature-and-Script-Checks)
    - [Submission to Mempool](#Submission-to-Mempool)
  - [P2P Transaction Relay](#P2P-Transaction-Relay)
    - [Transaction Announcement and Broadcast](#Transaction-Announcement-and-Broadcast)
    - [Transaction Request and Download](#Transaction-Request-and-Download)
    - [Orphans](#Orphans)
    - [Future Updates in Transaction Relay](#Future-Updates-in-Transaction-Relay)
  - [Inclusion in a Block](#Inclusion-in-a-Block)
    - [Mining](#Mining)
    - [Block Relay](#Block-Relay)
    - [Block Validation](#Block-Validation)
    - [State Changes and Persistence to Disk](#State-Changes-and-Persistence-to-Disk)
    - [Wallet Updates](#Wallet-Updates)
    - [Transaction Children](#Transaction-Children)

## Transaction Creation through Bitcoin Core Wallet

The process of creating a transaction does not need to be done on a node. Users can generate their
transactions on another wallet and/or entirely offline, and then submit raw transactions to their
Bitcoin Core node via one of its user interfaces (i.e. [`sendrawtransaction`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/rpc/rawtransaction.cpp#L825) through CLI or RPC).

The Bitcoin Core wallet also allows users to create transactions with varying levels of
customization. The steps are roughly as follows:

### Wallet Parameters

The wallet is configured (either automatically using defaults or through user input) with options
such as change output types, etc. It grabs feerate estimates from the node based on the user's
preferences, historical blocks and current mempool contents.  FIXME

### Transaction Payload

Relevant code:

Every [transaction](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/primitives/transaction.h#L259) has metadata, [inputs](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/primitives/transaction.h#L270), and [outputs](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/primitives/transaction.h#L271). The first part of the transaction to be
determined is typically the [outputs](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/outputtype.h#L23-L26); a payee [provides an invoice](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/wallet/wallet.cpp#L2103) to pay to. If the payee has a
special redeem script for these coins (e.g. multisig or timelock), they generate the script ahead of
time and embed the script hash into the destination.

The user specifies what payments are to be made in this transaction. These comprise the "recipient"
or "payment" outputs.  FIXME

### Coin Selection

Relevant code: [src/wallet/coinselection.h](https://doxygen.bitcoincore.org/coinselection_8h.html), [`CWallet::CreateTransactionInternal()`](https://doxygen.bitcoincore.org/class_c_wallet.html#a039a52a83b58db62d9c15d8371b9456e)

The [transaction is "funded"](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/wallet/spend.cpp#L373) using a set of [UTXOs available](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/wallet/spend.cpp#L67) in the wallet; these comprise the inputs.
A change output [may or may not be added](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/wallet/spend.cpp#L767-L778) to the transaction.  FIXME

### Signing

[Signatures](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/script/sign.cpp#L472) and [scripts are added](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/script/standard.cpp#L351). This process may involve various steps based on how many
[signatures are needed](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/script/sign.cpp#L199) and where the keys are stored.  FIXME

## Validation and Submission to Mempool

Regardless of where the transaction originated from, before being broadcast to the network, the
transaction must be [accepted into the node's mempool](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/node/transaction.cpp#L66). The mempool is designed to be a [cache of
unconfirmed transactions](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/txmempool.h#L81) that helps [miners select the best transactions](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/miner.cpp#L308) to include in the next
block. For non-mining nodes, keeping a mempool helps increase [block validation performance](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1438) and [aid
transaction relay](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L3149). In addition to enforcing [consensus rules](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L943), Bitcoin Core nodes apply a set of rules
known as [mempool _policy_](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L917) (also sometimes called "standardness" rules).

### Mempool Validation

Relevant code: [`MemPoolAccept`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L426), [policy.h](https://doxygen.bitcoincore.org/policy_8h.html)

Beyond applying consensus rules correctly, the primary consideration in mempool validation is DoS
prevention. Malicious nodes can create fake transactions very cheaply (both monetarily and
computationally): fees are not paid until [transactions are included in a block](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/miner.cpp#L234), and there are no
Proof of Work requirements on transactions. As such, [mempool validation logic](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1020-L1049) is designed to fail
fast and protect the node from expending any unnecessary resources.

We might categorize transaction validation checks in a few different ways:

- Consensus vs Policy: These can also be thought of as mandatory vs non-mandatory checks. They are
  not mutually exclusive, but we make all possible efforts to
  compartamentalize consensus rules to avoid making mempool logic consensus-critical.

- Script vs Non-script: We make this distinction because [script checking](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L932) is the most computationally
  intensive part of transaction validation.

- Contextual vs Context-Free: The context refers to the [current state of the block chain](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L439) (thinking
  about the Bitcoin network as a distributed state machine). Contextual checks might require the
[current block height](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L965) or knowledge of the [current UTXO set](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L414), while context-free checks only need the
transaction itself.

#### Context-Free Non-Script Checks

Relevant code: [`CheckTransaction()`](https://doxygen.bitcoincore.org/tx__check_8cpp.html#ac015dec2b8cac7f01b1f53b78a80d854)

Mempoool validation in Bitcoin Core starts off with non-script checks (sometimes called [`PreChecks`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L541),
the name of the function in which these checks run).

As a defensive strategy, the node starts with context-free and/or easily computed checks.
Here are some examples:

- None of the outputs are trying to send a [value less than 0](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/consensus/tx_check.cpp#L25) or [greater than 21 million BTC](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/consensus/tx_check.cpp#L27).

- The transaction [isn't a coinbase](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L563) - there can't be any coinbase transactions outside of blocks.

- The transaction isn't [more than 400,000 weight units](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/policy/policy.cpp#L88). It's possible for a larger transation to be
  consensus-valid, but it would occupy too much space in the mempool. If we allowed these
transactions, an attacker could try to cripple our mempool by sending very large transactions that
are later conflicted out by blocks.

#### Contextual Non-Script Checks

Relevant code: [`MemPoolAccept::PreChecks()`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L541)

Perhaps the most obvious non-script consensus check here is to [make sure the inputs are available](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L414). Each
input in the transaction must refer to a UTXO created by previous transaction by the [txid](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L410) and [index](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L411)
in the [output vector](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L397). Rather than looking through the entire blockchain (hundreds of gigabytes
stored on disk), Bitcoin Core nodes keep a [layered cache of the available coins](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/coins.h#L213) (a few gigabytes,
much of which can be kept in memory) to make this process more efficient Coins fetched from disk
during mempool validation are kept in memory if and only if the [transaction is accepted to the
mempool](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L3150). This prevents a peer from thrashing the coins cache (also bottlenecking block validation
performance) by creating invalid transactions spending random coins - remember that malicious nodes
would be able to do this without providing a valid signature.

Timelocks are also checked here - the node grabs the Median Time Past ([BIP113](https://github.com/bitcoin/bips/blob/master/bip-0113.mediawiki)) and/or block height
at the current chainstate to check the transaction [`nLockTime`](https://doxygen.bitcoincore.org/class_c_transaction.html#a54d5948c11f499b28276eab6bbfdf0c5) and input(s) [`nSequence`](https://doxygen.bitcoincore.org/class_c_tx_in.html#a635deeaf3ca4e8b3e1a97054607211b9). The node
[freezes chainstate](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1022) for the duration of this validation session to ensure those values don't change.
Even if a new block (which could extend or invalidate the chain tip) arrives on the wire, the
node won't process it until it has finished validating this transaction.

#### Signature and Script Checks

Relevant code: [src/script/interpreter.h](https://doxygen.bitcoincore.org/interpreter_8h.html), [`CheckInputScripts()`](https://doxygen.bitcoincore.org/validation_8cpp.html#a6a96a3e1e6818904fdd5f51553b7ea60), [`PrecomputedTransactionData`](https://doxygen.bitcoincore.org/struct_precomputed_transaction_data.html),
[`g_scriptExecutionCache`](https://doxygen.bitcoincore.org/validation_8cpp.html#ab821bded1c098da86bb13a4ac9d1208b), [`CScriptCheck`](https://doxygen.bitcoincore.org/class_c_script_check.html), [`STANDARD_SCRIPT_VERIFY_FLAGS`](https://doxygen.bitcoincore.org/policy_8h.html#ab28027bf27efcdd6535a13175a89ca5a)

Transaction [script checks](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1356) are actually context-free; the unlocking script ([`scriptSig`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1357),
`redeemScript`, and [`witness`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1358)) in each input is paired with the locking script ([`scriptPubKey`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/script/interpreter.cpp#L1979)) in
the [corresponding UTXO](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1469) and passed into the script interpreter. The [script interpreter](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/script/interpreter.cpp#L431) simply
[evaluates the series of opcodes](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/script/interpreter.cpp#L508) and data based only on the arguments passed to it. One such argument
is a [set of script verification flags](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L922) indicating which rules to apply during script verification.

For example, the [`OP_CHECKSEQUENCEVERIFY`](https://doxygen.bitcoincore.org/script_8h.html#a63e349a6089a54da9fe09a3d858648bdaff2d931584100152c5a1af8c875c3cb6) opcode [repurposed](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki) `OP_NOP3`. The script verification flag
[`SCRIPT_VERIFY_CHECKSEQUENCEVERIFY`](https://doxygen.bitcoincore.org/interpreter_8h.html#a05589fbab0657f08285ebdfe93f5ec9eaddf7dd85ffe0653f478d6888bcbe4021) instructs the script interpreter [to interpret](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/script/interpreter.cpp#L587) the opcode [`0xb2`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/script/script.h#L191)
as ["check that the input sequence is greater than the stack value"](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/script/interpreter.cpp#L1800) or as a no-op. At the [BIP113](https://github.com/bitcoin/bips/blob/master/bip-0113.mediawiki)
activation height, [nodes pass](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1695) `SCRIPT_VERIFY_CHECKSEQUENCEVERIFY=1` into the script interpreter.

Another part of script validation is [signature verification](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/script/interpreter.cpp#L416) (indicated in a script by opcodes such
as [`OP_CHECKSIGVERIFY`](https://doxygen.bitcoincore.org/script_8h.html#a63e349a6089a54da9fe09a3d858648bdaad934d053cc56be9612b2faa9f396eeb)), which is the [most expensive](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/secp256k1/src/secp256k1.c#L424) part of transaction validation.

Transactions might have multiple [signatures](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/primitives/transaction.h#L69) on various parts of the transaction. For example,
multiple parties might have contributed to funding the transaction, each signing on some combination
of the outputs that they care about. Even in the most basic, single signature transaction, we need
to [serialize and hash](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/script/interpreter.cpp#L1686) parts of the transaction according to the [sighash flag(s)](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/script/interpreter.h#L25-L35) used. To save the
node from repetitive work, at the [very start of script checks](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1442), subparts of the transaction are
[serialized, hashed, and stored](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/script/interpreter.cpp#L1423) in a [`PrecomputedTransactionData`](https://doxygen.bitcoincore.org/struct_precomputed_transaction_data.html) struct for use in signature
verification.

[`MemPoolAccept`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L426) performs two sets of script checks: [`PolicyScriptChecks`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L917) and
[`ConsensusScriptChecks`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L943). The former [runs the script interpreter](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L926) using [consensus and policy flags](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L922) and
[caches the result](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/script/sigcache.cpp#L114) in the [signature cache](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/script/sigcache.cpp#L90). The latter runs the script interpreter using [consensus
flags only](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L965) and [caches the full validation result](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1490), identified by the [wtxid and script verification
flags](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1426), in the script cache. If a new consensus rule is activated between now and the block in which
this transaction is included, the cached result is no longer valid, but this is easily detected
based on the script verification flags. The cached signature verification - which does not have
specific script verification flags - result from [`PolicyScriptChecks`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L917)  is used immediately in
[`ConsensusScriptChecks`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L943); we almost never need to verify the same signature twice!

### Submission to Mempool

Relevant code: [`MemPoolAccept::Finalize()`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L974), [`CWallet::transactionAddedToMempool`](https://doxygen.bitcoincore.org/group__map_wallet.html#gad8ca1601aff7d2ba2f7c39fa0ff8573e)


[The mempool](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/txmempool.h#L566) primarily serves as a [size-limited cache](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1013) of the best candidates for inclusion in the
next block. Thus, transactions are evicted based on their [descendant feerate](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/txmempool.cpp#L1060).

A transaction being added to the mempool is an event that implementers of [`ValidationInterface`](https://doxygen.bitcoincore.org/class_c_validation_interface.html) are
[notified about](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1046). If the transaction originated from the node's wallet, the [wallet notes](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/wallet/wallet.cpp#L1187) that it has
been added to mempool.

## P2P Transaction Relay

Relevant code: [`PeerManagerImpl::RelayTransaction()`](https://doxygen.bitcoincore.org/class_peer_manager.html#a46d809bb4a782f3bb03dd6ccc89ab6cf)

A node participating in transaction relay broadcasts [all of the transactions in its mempool](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L4632).
The primary goal in transaction relay is to propagate transactions to every node in the network in a
timely manner, but _privacy_ and network bandwidth are also important consideration in Bitcoin. Since the
average block interval is about 10 minutes, we'll consider a [few second-delay](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L4584-L4588) acceptable if it helps
obfuscate transaction origins and avoids clogging up the network with redundant transaction messages.

Technically, the Bitcoin P2P protocol specifies that transactions are communicated using a [`TX`
message](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/protocol.h#L121).  Most nodes relay transactions using a three-part dialogue:

1. The sender [sends an `INV` (Inventory) message](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L4571) announcing the new transaction by its [witness hash](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L4607).
   Non-segwit nodes or those implementing an older protocol version may communicate using txids.

2. The receiver [sends a `GETDATA`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L4808) to request transactions they [are interested in](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/txrequest.cpp#L596) (i.e. don't already
   have), also identified by their hash.

3. The sender [sends the full transaction](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L1879) in a `TX` message.

### Transaction Announcement and Broadcast

Relevant code: [`CTxMemPool::m_unbroadcast_txids`](https://doxygen.bitcoincore.org/class_c_tx_mem_pool.html#a3df5ff43adfe0f407daa6fdef8965ba8), [`PeerManagerImpl::ReattemptInitialBroadcast()`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L1099)

With respect to privacy in transaction relay, we specifically want to hide the origin (IP address)
of a transaction. For each peer, nodes send a batch of transaction announcements at [random intervals](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L4584-L4588)
(poisson timer with 2-second average for outbound peers and 5-second average for inbound peers).

After broadcasting a transaction for the first time, Bitcoin Core [nodes store them](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/node/transaction.cpp#L101) in an
_unbroadcast_ set until they see an `INV` for it, indicating that the transaction has been accepted
to a peer's mempool.

### Transaction Request and Download

Relevant code: [PR#19988](https://github.com/bitcoin/bitcoin/pull/19988), [txrequest.h](https://doxygen.bitcoincore.org/txrequest_8h.html)

Since a node typically has at least 8 peers relaying transactions (there are [8 full relay outbound
connections](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net.h#L62) even with inbounds disabled), it will likely [receive multiple announcements](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L2859), but only
requests the transaction from one peer at a time to avoid redundant network traffic. At the same
time, it's possible for a request to fail. A peer might [evict the transaction from their mempool](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/txmempool.cpp#L256),
take too long to respond, terminate the connection, etc. Malicious nodes may also try to take
advantage of flaws in the transaction request logic to censor or stall a transaction from being
propagated to miners. In these cases, nodes must remember which other peers announced the same
transaction in order to re-request it.

Rather than responding to transaction `INV`s with `GETDATA`s immediately, nodes [store all the
announcements received](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L2859), batching ones corresponding to the same transaction, selecting the [best
candidate](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/txrequest.cpp#L587) to request the transaction from based on connection type and how many requests are already
in flight. Nodes [prefer to download](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L1060-L1066) from manual connections, outbound connections, and peers using
WTXID relay. If the first announcement(s) to arrive are from non-preferred peers, the node waits for
other peers to announce the same transaction for a short period of time (several seconds) before
sending the request.

### Orphans

Relevant code: [src/txorphanage.h](https://doxygen.bitcoincore.org/txorphanage_8h.html)

FIXME: orphans

### Future Updates in Transaction Relay

- [Erlay](https://github.com/bitcoin/bitcoin/pull/21515): this would introduce an additional method of transaction announcements that reduces `INV`
  and `GETDATA` traffic (much of which is redundant, since each node may hear about a transaction
from dozens of peers). Rather than sending `INV`s for each transaction, peers periodically use set
reconciliation to determine which transactions to send to each other.

- [Package Relay](https://github.com/bitcoin/bitcoin/pull/16401): Package validation (rather than just individual transaction validation) may produce
  a more accurate evaluation of transaction validity. With package relay, in addition to (or perhaps
as a substite for) relaying transactions individually, nodes may instruct peers to validate specific
transactions together as packages.

## Inclusion in a Block

A [signature on a transaction](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/wallet/spend.cpp#L827) indicates that the owner of the private key has agreed to spend the
coins, but the transaction should be untrusted until there is network consensus that the coins have
been sent - it is trivially cheap for a private key owner to sign multiple transactions sending the
same coins. In Bitcoin, our consensus protocol requires the transaction to be included in a block
containing a [valid Proof of Work](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/pow.cpp#L74) and [accepted by the network](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L2382) as part of the most-work chain - it is
prohibitively expensive for a private owner to reverse a transaction by rewriting (and recomputing
Proof of Work for) these blocks.

### Mining

Relevant code: [src/miner.h](https://doxygen.bitcoincore.org/miner_8h.html), [`BlockAssembler::CreateNewBlock()`](https://doxygen.bitcoincore.org/class_block_assembler.html#acd45f01ed2556e805c82907def19efcb)

Miners can start working on a block as soon as they [have the previous block hash](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L2846). The process of
creating a new block might look something like this:

1. The miner calls the Bitcoin Core RPC [`getblocktemplate`](https://doxygen.bitcoincore.org/mining_8cpp.html#a55115d559f5dbfb0d7c56692cd2b69e4) to generate a block _template_ containing
   a consensus-valid set of transactions and header, just mising the nonce.

2. The miner dispatches the block template to other hardware (perhaps a rack of ASICs, a cloud
   instance, or other nodes operated by mining pool members) dedicated to finding the nonce.

3. If a solution is found, the miner calls [`submitblock`](https://doxygen.bitcoincore.org/mining_8cpp.html#a5195db6e4464b7653c398ff7a362b21e) to broadcast the new block.

Bitcoin Core's mining code (used by [`getblocktemplate`](https://doxygen.bitcoincore.org/mining_8cpp.html#a55115d559f5dbfb0d7c56692cd2b69e4)) uses the mempool, [sorted by ancestor score](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/miner.cpp#L320)
(ancestor feerate + manual prioritization by the miner), to determine which transactions to [include
in the next block](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/miner.cpp#L226). This means a "package" consisting of a 5 sat/vB parent and 100 sat/vB child may
get picked over a 20 sat/vB transaction, but not over a singular 100 sat/vB transaction with no
dependencies.

### Block Relay

Relevant code: [BIP152](https://github.com/bitcoin/bips/blob/master/bip-0152.mediawiki)

Blocks can contain megabytes worth of transactions, and nodes that keep a mempool have seen
at least 98% of them already (FIXME source: [gmax's block relay talk](https://www.youtube.com/watch?v=EHIuuKCm53o)).

The methods for block relay are listed below, along with a description of how the transaction is
communicated with or without sending the full transaction data over P2P. Note that peers negotiate
which block relay communication method to use _before_ blocks are communicated.

- High Bandwidth Compact Blocks: the sender [creates a BIP152 compact block](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L1481) comprising only the block
  header, prefilled coinbase transaction, and shortids for all other transactions. If the
receiver does not recognize a transaction shortid, it [may request](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L3434) it through `GETBLOCKTXN`.

- Headers First: to prevent huge spikes in network traffic, senders flood [block headers only](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L4526) (which
  is just 80 bytes). The recipient can choose to request the [full block](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L2093) or [compact block](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L2107).

- `INV`/`GETDATA` Dialogue: the peers sacrifice latency for lower bandwidth usage by announcing blocks [through
  `INV` messages](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/net_processing.cpp#L1813). The recipient can request the full or compact block.

### Block Validation

Relevant code: [`-par`](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/init.cpp#L395), [`g_scriptExecutionCache`](https://doxygen.bitcoincore.org/validation_8cpp.html#ab821bded1c098da86bb13a4ac9d1208b), [`CChainState::ActivateBestChain()`](https://doxygen.bitcoincore.org/class_c_chain_state.html#af771ed84553b1fb7269cfdfb54f66c48)

FIXME: non-script checks

If the node [already validated this transaction before](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1490) it was included in a block, no consensus rules
have changed, and the script cache has not evicted this transaction's entry, it doesn't need to [run
script checks again](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L1428-L1430)!

### State Changes and Persistence to Disk

Relevant code: [src/undo.h](https://doxygen.bitcoincore.org/undo_8h.html), [`CTxUndo`](https://doxygen.bitcoincore.org/class_c_tx_undo.html), [`SaveBlockToDisk()`](https://doxygen.bitcoincore.org/blockstorage_8cpp.html#ac821ccc511a43e5404cf9955156fde45)

To fully validate new blocks, nodes only need to [store some representation of the current state](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/coins.cpp#L222) and
knowledge of the current consensus rules. Archival nodes [store the entire block chain](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/node/blockstorage.cpp#L177) and can thus
query any arbitrary block or transaction from Bitcoin history. Pruned nodes only [store a portion of
the most recent blocks](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L2207). Additionally, the node should be prepared for a [reorg](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/validation.cpp#L2529) - the block may become
stale if a new most-work chain is found, and the changes to the UTXO set will need to be undone.

FIXME: coins cache flushing

Historical [blocks are persisted](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/flatfile.cpp#L81) to disk in the [`blocks/` directory](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/node/blockstorage.cpp#L210). Each `blkNNNNN.dat` file stores
raw block data, and its corresponding undo data is [stored in a `revNNNNN.dat` file](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/node/blockstorage.cpp#L108) in the same
directory. If the node operator has configured a maximum memory allocation for this node, this
process will also prune blocks by oldest-first to stay within limits.

### Wallet Updates

Relevant code: [src/wallet/spend.h](https://doxygen.bitcoincore.org/spend_8h.html), [`CWallet::blockConnected`](https://doxygen.bitcoincore.org/group__map_wallet.html#ga0cf4d454590b1513b967523c77f495ad)

The Bitcoin Core wallet tracks incoming and outgoing transactions and subscribes to [major events](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/wallet/wallet.cpp#L1187-L1265) by
implementing the node's [`ValidationInterface`](https://doxygen.bitcoincore.org/class_c_validation_interface.html). The most relevant information for the wallet is which
coins are available and how likely they are to remain spendable - this is quantified by the number
of confirmations on the transaction (or on a [_conflicting_](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/wallet/transaction.h#L342) transaction, [represented as negative
numbers](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/wallet/wallet.cpp#L1128)). A UTXO made available by a transaction 100 blocks deep in the most-work chain is
considered pretty safe. On the other hand, if a newly confirmed transaction conflicts with one that
sent coins to the wallet, those coins now have a "depth" of -1 and [will not be considered a probable
source of funds](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/wallet/spend.cpp#L94).

### Transaction Children

A transaction has a "child" when its outputs are used as the inputs to fund another transaction
(which may or may not be valid). Technically, a child can be created as soon as the transaction has
a txid (i.e. after the transaction has been signed), including some limitations:

* The Bitcoin Core wallet [may not consider the coins reliable enough to spend](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/wallet/spend.cpp#L102). For example, the coin
  selection algorithm never selects unconfirmed UTXOs that originate from foreign sources.
* Transactions spending coinbase outputs are not considered consensus valid [until 100 blocks later](https://github.com/bitcoin/bitcoin/blob/1a369f006fd0bec373b95001ed84b480e852f191/src/wallet/wallet.cpp#L2881).