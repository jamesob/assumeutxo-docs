
## Abstract

An implementation and deployment plan for `assumeutxo` is proposed, which uses serialized UTXO sets to substantially reduce the amount of time needed to bootstrap a usable Bitcoin node with acceptable changes in security.

#### Design goals

1. Provide a realistic avenue for non-hobbyist users to run a fully validating node,
1. avoid imposing significant added storage burden, and
1. make no significant concessions in security.

#### Plan

This feature could be deployed in two, possibly three, phases:

1. UTXO snapshots (~3.2GB) can be created and loaded via RPC in lieu of the normal IBD process.
    - They will be obtained through means outside of bitcoind.
    - A hardcoded `assumeutxo` hash will fix which snapshots are considered valid.
    - Asynchronous validation of the snapshot will be performed in the background after it is loaded.
This phase includes most, if not all, of the changes needed to existing validation code. (see [the PR](https://github.com/bitcoin/bitcoin/pull/15606))
1. Snapshots will be generated, stored, and transmitted by bitcoind.
    - To mitigate DoS concerns and storage consumption, nodes will store subsets of FEC-split chunks spanning three snapshots (one current, two historical) at an expected ~1.2GB storage burden.
    - Nodes bootstrapping will, if assumeutxo is enabled, obtain these chunks from several distinct peers, reassemble a snapshot, and load it.
    - The hardcoded assumeutxo value will change from a content hash to a Merkle root committing to the set of chunks particular to a certain snapshot.
    - We may consider adding a rolling UTXO set hash for local node storage to make accessing expected UTXO set hashes faster, and may augment the assumeutxo commitment with its value.
1. (Far down the road) a consensus commitment to the UTXO set hash at a given height may be considered. The snapshot and background validation process may be reused as we migrate to a more compact representation of the UTXO set, e.g. [utreexo](https://www.youtube.com/watch?v=edRun-6ubCc), [UHS](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2018-May/015967.html), or [accumulators](https://eprint.iacr.org/2018/1188).

If parts of that were incomprehensible, keep reading for details.

Right now you can help by reviewing this proposal and the draft code.

#### Resources

- Github issue: https://github.com/bitcoin/bitcoin/issues/15605
- Phase 1 draft implementation: https://github.com/bitcoin/bitcoin/pull/15606

#### Already familiar?

If you've been following this discussion and already understand the basics of
`assumeutxo`, you can probably skip down to the [*Security* section](#security).

#### Acknowledgements

I'd like to thank the following people for helping with this proposal, though they should not be held accountable for any dumb mistakes I may have made in the preparation of this document and related code:

Suhas Daftuar, Pieter Wuille, Greg Maxwell, Matt Corallo, Alex Morcos, Dave Harding, AJ Towns, Sjors Provoost, Marco Falke, Russ Yanofsky, and Jim Posen.


## Basics

### What's a UTXO snapshot?

It's a serialized version of the unspent transaction output (UTXO) set, as seen from a certain height in the chain. The serialized UTXOs are packaged with some metadata: e.g.
  - total number of coins contained in the snapshot,
  - the block header for the latest block encapsulated in a snapshot (its "base"),

and a few other things. You can see the full data structure (in its current form) in the [assumeutxo pull request](https://github.com/jamesob/bitcoin/blob/utxo-dumpload-compressed/src/validation.h#L827-L881).

### What's `assumeutxo`?

It's a piece of data embedded in the source code that commits to the hash of a serialized UTXO set which is considered valid for some chain height. The final format of this commitment is still subject to debate because generating it is computationally expensive, and its structure affects how we store and transmit the serialized UTXO set to and from peers. But at the moment it is a simple SHA256-based hash of the UTXO set contents generated by the existing [`GetUTXOStats()` utility](https://github.com/bitcoin/bitcoin/blob/91a25d1e711bfc0617027eee18b9777ff368d6b9/src/rpc/blockchain.cpp#L944-L981).


### Okay... what's the use of these things?

We can use UTXO snapshots and the `assumeutxo` commitment to significantly reduce the amount of time it takes to bootstrap a usable Bitcoin node under a security model that seems acceptable.

Right now initial block download is a process that scales linearly with the size of the chain's history. It takes anywhere from 4 hours to multiple days to complete this process for a new installation of bitcoind, based upon hardware and network bandwidth. This process discourages users from running full nodes, instead incentivizing users to turn to clients with a reduced security model.

### How does snapshot loading work?

When a snapshot is loaded, it is deserialized into a full chainstate data structure, which includes a representation of the block chain (`chainActive`) and UTXO set (both on disk and cached in memory). This lives alongside the original chainstate that was extant before loading the snapshot. Before accepting a loaded snapshot, a headers chain must be retrieved from the peer network which includes the block hash of the last block encompassed by a snapshot (its "base").

Once the snapshot is loaded, the snapshot chainstate performs initial block download from the snapshot state to the network's tip. The system then allows operation as though IBD has completed, and the assumed-valid snapshot chainstate is treated as `chainActive`/`pcoinsTip`/et al.

After the snapshot chainstate reaches network tip, the original chainstate resumes the initial block download from before the snapshot was loaded in the background. This "background validation" process happens asynchronously from use of the active (snapshot) chainstate, allowing the system to service, for example, wallet operations. The purpose of this background validation is to retrieve all block files and fully validate the chain up to the start of the snapshot.

Until the background validation completes, we refuse to load wallets with a `bestblock` marker before the base of the snapshot since we don't have the block data necessary to perform a rescan.

Once the background validation completes, we throw its state away as the snapshot chainstate has now been proven fully valid. If for some reason the background validation yields a UTXO set hash that is different from what the snapshot advertised, we warn loudly and shut down.


## Security

### Is there a change in security model by introducing `assumeutxo`?

If we're talking about the degree to which developers are trusted to identify what is and isn't valid in Bitcoin: no, there is no material difference between use of assumeutxo and the degree that Bitcoin "trusts developers" now.

Currently, Bitcoin ships with hardcoded assumevalid values. These values identify certain blocks which, if seen in the headers chain, allow signature checking for any prior blocks to be skipped as a performance optimization.

> With assumevalid, you assume that the people who peer review your software are capable of running a full node that verifies every block up to and including the assumedvalid block.  This is no more trust than assuming that the people who peer review your software know the programming language it's written in and understand the consensus rules; indeed, it's arguably less trust because *nobody* completely understands a complex language like C++ and *nobody* probably understands every possible nuance of the consensus rules---yet almost anyone technical can start a node with -noassumevalid, wait a few hours, and check that `bitcoin-cli getblock $assume_valid_hash` returns has a `"confirmations"` field that's not -1.
>
> *Dave Harding*

Assumeutxo is a similar idea and would be specified in a more stringent way (in that it can't be specified via CLI). The hardcoded assumeutxo value would be proposed and reviewed in exactly the same way as assumevalid (via pull request), and would be proposed and merged with a sufficient lead time that would allow anyone interested to reproduce its value for themselves.

### But isn't a hash in the code that assumes validity of a chain basically like the developers deciding what the "right" chain is?

This value is no different than any other code change proposed by developers; in fact, it would be much easier to sneak in some kind of backhanded logic that implements a skewed notion of validity into another, more obscure part of the code, e.g. `CCoinsViewDB` could be modified to always attest to the existence of a coin under some special condition, or net could be modified to only communicate with certain peers on the network.

The clarity around an assumevalid/assumeutxo commitment value actually makes user participation much more straightforward because it's obvious how this optimization can be reviewed.

It's also worth noting that the existence of an assumevalid/utxo value doesn't preclude any other chain from being considered valid, it simply says "the software previously validated *this* particular chain."

### Okay, so there might be a theoretical equivalence, but are there any *practical* security differences with assumeutxo (vs. assumevalid)?

Yes, there is one practical security difference. Currently, if I wanted to trick someone into thinking I had coins that I don't on the honest network, I'd have to

- get them to start bitcoind with a bad `-assumevalid=` parameter,
- isolate their node from the honest network to prevent them from seeing the
  most-PoW headers chain, and
- build a PoW-valid chain that includes the existing [checkpoints](https://github.com/bitcoin/bitcoin/blob/91a25d1e711bfc0617027eee18b9777ff368d6b9/src/chainparams.cpp#L146-L160), and includes a block with an invalid coin assignment somewhere after the last checkpoint.

This obviously involves a bit of effort since the attacker needs to generate a chain of blocks along with the requisite PoW.

However, with assumeutxo if I can get the user to accept a malicious `assumeutxo` commitment value, most of the work is done. Modifying and serializing a false UTXO snapshot is quite easy -- no proof of work necessary.

### That sounds really bad - so all an attacker has to do is get a user to accept a bad assumeutxo value and feed them a poisoned snapshot?

Yes, that's all it takes.

As a result, the assumeutxo value will be embedded in the source code and we will not build a mechanism that enables specification of the `assumeutxo` value via commandline; the practical risk is just too high. If users prefer to specify an alternate value (not recommended), they can modify the source code and recompile.

The assumeutxo value will live in the source code; recall that if an attacker has means to affect the source code used to build a binary, they can already do anything they want.

### It seems like this is an argument for including a commitment to the assumeutxo value in a place where it could be enforced by consensus, say in the block headers. Should we do that?

Maybe in time, but not at the moment. Before we gain practical experience with use of UTXO snapshots, we don't know what the right commitment structure is. Making consensus changes is a very expensive process and should not be done until we're absolutely sure of what we want to commit to.

Down the road we may introduce such a commitment into a consensus-critical place, but for now we should design assumeutxo to be secure without the assumption that we will.

### Do you perform any extra validation on a loaded snapshot besides comparing its hash to the `assumeutxo` value?

Yes. After the UTXO snapshot is loaded and the chain syncs to the network tip, we begin an initial block download process in the background using separate data structures (i.e. a separate chainstate). This background IBD will download and validate all blocks up to the last block assumed valid by the snapshot (i.e. the "base" of the snapshot).

Once the background IBD completes, we will have validated all blocks in the previously assumed-valid chain we've been using since we loaded the snapshot. We can throw away the background validation `chainstate` data.



## Resource usage

### This extra chainstate that we're using for the background IBD -- doesn't that take up extra disk space and memory for a separate leveldb and cache?

Yes. Since we have to maintain a completely separate UTXO set to support background IBD that is simultaneous with use of the assumed-valid chain, we have to have an extra `CCoinsView*` hierarchy. This means temporarily keeping an extra `chainstate` (leveldb) directory on disk, and it means splitting the memory allocated per `-dbcache` to the in-memory coins cache.

I don't think this is a huge deal because it basically means (at the moment) an extra 3.2GB on disk. We can split the specified `-dbcache` memory ~80/20 between the `CCoinsViewCache` used for background IBD vs. the assumed-valid `chainActive`, since a sizable dbcache only provides a noticeable performance benefit during initial block download.

### Should we even run a background validation sync? If we accept the assumeutxo security model, why even do the IBD? If IBD isn't scalable long term, what's the point?

If we introduce assumeutxo with snapshots but do not perform IBD in the background, it's easy to imagine that almost anyone setting up a node will do so with a UTXO snapshot (since it's much quicker than conventional IBD), run using an assumed-valid chain, and will present itself to the network as a pruned node. In the limit, that results in an absence of nodes serving historical blocks to the network. This certainly isn't what we want, so it seems prudent to keep the background IBD on as a default.

Users constrained by hardware can of course use assumeutxo with pruning options.

Assumeutxo is a performance optimization. If we removed the IBD process in lieu of it, that would be a change in Bitcoin's security model. In the future, we may split the block download and connection/validation processes so that assumeutxo nodes can still serve blocks to the peer network without having to expend the computational resources needed to perform IBD-style validation.


### How will users and reviewers efficiently verify hashes of a given UTXO set?

Right now, computing the hash of the UTXO set at a certain height can be done by using the `gettxoutsetinfo` RPC command (i.e. `GetUTXOStats()`). It takes a few minutes to compute, and if you want to do it for an arbitrary height, you need to call `invalidateblock` to rewind to that point and then `reconsiderblock` afterwards to fast-forward back. Obviously, this interrupts normal operation.

This is inconvenient, but in the meantime we could modify `gettxoutsetinfo` to accept a height and at least abstract away the manual chain rewind and fast-forward via `invalidateblock`/`reconsiderblock`.

Longer-term, it's conceivable that we could use a node-local [rolling UTXO set hash](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2017-May/014337.html) to make the hash availability immediate. However, a rolling UTXO set hash is incompatible with assumeutxo commitment schemes that involve chunking snapshots (discussed below) and so the resulting assumeutxo value might have to be a tuple consisting of `(rolling_set_hash, split_snapshot_chunks_merkle_root)`.


## Snapshot storage and distribution

### How will snapshot distribution work?

Ultimately, users will obtain UTXO snapshots over the peer network. Before that is implemented, users may obtain UTXO snapshots that validate against the hardcoded assumeutxo hash from any individuals or CDNs offering them (see "*What are the steps to deployment?*" below).

Because snapshots are quite sizable, and because malicious peers may well lie about a large file they're offering being a valid snapshot, we need chunked storage and transmission of snapshots. It should be easy to validate each chunk.

Naively, one way we could do this is split each snapshot into *n* evenly-sized chunks. We could construct a Merkle tree using the hash of each chunk, and then the root of that tree would be the `assumeutxo` commitment value embedded in the source code. Each peer would choose some random value modulo *k* (such that *k* <= *n*) that determines what subset of chunks it was responsible for storing. Upon initialization, a peer wanting to obtain a snapshot would have to find *k* distinct peers, each providing a unique "stripe" of the snapshot data, to obtain all *n* chunks.

### That sounds nice and simple, but I bet there are problems.

Right. The issues with this approach are
- finding a full set of peers which offers all *k* stripes of the data is inconvenient, and
- this opens a minor DoS vector - to prevent initialization, an attacker has only to target all nodes offering one particular stripe of the data.

Instead, we could use [erasure coding](http://web.eecs.utk.edu/~plank/plank/classes/cs560/560/notes/Erasure/2004-ICL.pdf) to generate *n* + *m* chunks (where *m* is the number of extra coding chunks) and have the bootstrapping node retrieve any *n* + 𝛼 distinct chunks from its peers, where 𝛼 depends on the particular coding scheme used. Each node would still only store and serve a subset of the snapshot chunks.

### How many snapshots will a node have to store?

A node would store more than just the latest snapshot in order to help peers running legacy versions of the software. The exact number could be subject to debate, but I'd say storing some data for two historical snapshots in addition to the latest snapshot is probably reasonable.

For each snapshot, each node would only store some slice of the data (perhaps an 1/8th or so) based upon the exact parameters chosen of an erasure coding scheme above.

For reference, a UTXO snapshot at a recent tip is roughly 3.2GB. If we don't do anything clever and snapshots remain at this size, we might expect nodes to store 1.2GB (= 1/8 shard size * 3 snapshots * 3.2GB per snapshot) of snapshot data (assuming 8 peers required for snapshot bootstrap and 3 total snapshots stored).

### How will snapshots be available from peers for a new assumeutxo value that has just been released?

Good question - I haven't quite figured this one out yet. Presumably, we could have the code automatically generate snapshots every 6 months or so. In order to generate snapshots during runtime without impairing normal operations like new block reception, we'll probably have to refactor state-to-disk flushing to be asynchronous.

Sjors notes that we could generate snapshots periodically based on block height, which would be useful for more than transmission to peers:

> At fixed block intervals, the snapshots will [immediately] be useful even if they're not referenced in a release yet. They can be used for local backups to recover from block & chainstate data corruption. Each node can store the hash in a simple text file. When calling -reindex(chainstate) it looks for that text file. For pruned nodes -reindex could even just go back to the snapshot instead of all the way to genesis.  Similarly this can be helpful when a node needs to undo pruning to rescan an old wallet.



## Alternative approaches

### Instead of using UTXO snapshots that we have to store and send around, why not just have Bitcoin start in SPV mode (using e.g.  [BIP 157](https://github.com/bitcoin/bips/blob/master/bip-0157.mediawiki)) and do initial block download in the background?

This is an appealing idea, but there are some practical downsides.

For one, much less code is required to implement the assumeutxo approach than to build an SPV mode into Bitcoin Core. At the moment, Core doesn't have any code for acting as an SPV client and many subsystems assume the presence of a chainstate (i.e. a `CChain` blockchain object and a view of the UTXO set).

Maybe surprisingly, the assumeutxo approach allows us to reuse a lot of existing code within Bitcoin Core. Its implementation amounts to a series of refactorings (that should probably be done anyway to make writing tests easier) plus a few new smallish pieces of logic to handle multiple chainstates during initialization and network communication.

A good deal of new code will be required to use, say, [Reed-Solomon erasure coding](http://web.eecs.utk.edu/~plank/plank/classes/cs560/560/notes/Erasure/2004-ICL.pdf) to split snapshots for storage and peer transmission, but this sort of code can easily live at the periphery of the system, whereas building in an SPV mode would require numerous modifications to the "heart" of the software.

More new code is more engineering and review effort, and ultimately more risk. An SPV mode would also not allow the node to do a full validation of incoming blocks until the full chain has been downloaded and validated.

### Why not do something that doesn't require any modification to the Bitcoin Core code, like have people offer PGP-signed datadirs that you can download *a la* btcpayserver's [FastSync](https://github.com/btcpayserver/btcpayserver-docker/tree/master/contrib/FastSync)?

Distributing this sort of data outside of the core software has a number of practical downsides. If an individual or group were to encourage downloading chainstate data that is not somehow validated by the software itself, a number of security risks are introduced. The user begins to trust not just bitcoind, but the supplier of the data. The supplier of that data then needs to be scrutinized in addition to the software itself.

Futhermore, existing schemes like FastSync use PGP signatures to attest to the validity of the data they offer. Such signatures are very often ignored, and convincing users to validate them reliably is a sisyphean task.

Usage of assumeutxo or a similar scheme should be *safe by default* and ultimately the user should not have to perform any extra steps to benefit from the optimization in a secure way.


## Planning

### Okay, this assumeutxo stuff actually sounds pretty cool. What are the steps to deployment?

1. Implement the changes necessary to have multiple chainstates in use simultaneously. ([See PR](https://github.com/bitcoin/bitcoin/pull/15606))
1. Implement the creation and usage of UTXO snapshots via `dumptxoutset` and `loadtxoutset` along with a hardcoded `assumeutxo` hash. ([See PR](https://github.com/bitcoin/bitcoin/pull/15606))
1. Allow time for sophisticated end users to manually test snapshot usage via RPC.
1. Research an effective snapshot storage and distribution scheme.
1. Implement and deploy the decided-upon P2P snapshot distribution mechanism.
1. Let some time pass.
1. Consider whether or not a consensus change supporting a UTXO set hash makes any sense.

### If I want to help, what should I do next?

Review the code [here](https://github.com/bitcoin/bitcoin/pull/15606). Parts of it can and may be split out given the size of the current change, and your input there would be appreciated.

### How will this work with [accumulators](https://eprint.iacr.org/2018/1188), [UHS](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2018-May/015967.html), or [utreexo](https://www.youtube.com/watch?v=edRun-6ubCc) if those things come around?

If some alternate, space-saving scheme for representing the UTXO set becomes viable, it'll dovetail with assumeutxo nicely. It's easy to imagine that the assumeutxo value could simply become a merkle root of the utreexo forest, or the hash of an accumulator value. UTXO snapshots would reduce to just a few kilobytes instead of multiple gigabytes. The background IBD/validation feature will still be quite useful as we'll still want to do full validation in the background.


## Implementation questions

### How should we allocate memory (`-dbcache`) between the assumed-valid and background validation coins views?

From Sjors:

> I think it should allocate most memory to catching up from snapshot to the tip. This could be more than 6 months away. Once caught up, flush and allocate most memory to syncing from genesis. If a node is restarted during the process, sync the headers, if it's more than 24 hours behind, give most resources to catching up to tip, otherwise give them to catching up to the snapshot.
>
> For the very first version at least we need to make sure memory usage for both UTXO sets doesn't exceed `-dbcache + -maxpool`.

At the moment, the draft implementation does a [70/30 split](https://github.com/bitcoin/bitcoin/pull/15606/commits/83f13a754372579cd13a45a2052fd4e42ed24632#diff-c865a8939105e6350a50af02766291b7R1476) between background validation and snapshot.

### Why allow loading a snapshot through RPC - shouldn't it just be a startup command?

Loading the snapshot as a startup parameter would allow us to simplify various things (like management of chainstate data structures), but if we're eventually going to transmit snapshots over the P2P network, we'll need to have logic in place that allows loading snapshots after startup has completed. I think we should be sure to test this kind of operation before this feature sees default usage, and so `loadtxoutset` seems like the right approach during the RPC phase.
