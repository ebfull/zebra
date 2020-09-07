# Treestate

- Feature Name: treestate
- Start Date: 2020-08-31
- Design PR: [ZcashFoundation/zebra#983](https://github.com/ZcashFoundation/zebra/issues/983)
- Zebra Issue: [ZcashFoundation/zebra#958](https://github.com/ZcashFoundation/zebra/issues/958)

# Summary
[summary]: #summary

To validate blocks involving shielded transactions, we have to check the
computed treestate from the included transactions against the block header
metadata (for Sapling) or previously finalized state (for Sprout). This document
describes how we compute and manage that data, assuming a finalized state
service as described in the [State Updates RFC](https://zebra.zfnd.org/dev/rfcs/0005-state-updates.md).


# Motivation
[motivation]: #motivation

Block validation requires checking that the treestate of the block (consisting
of the note commitment tree and nullifier set) is consistent with the metadata
we have in the block header (the root of the note commitment tree) or previously
finalized state (for Sprout).


# Definitions
[definitions]: #definitions

Many terms used here are defined in the [Zcash Protocol Specification](https://zips.z.cash/protocol/protocol.pdf)

**notes**: Represents a value bound to a shielded payment address (public key)
which is spendable by the recipient who holds the spending key corresponding to
a given shielded payment address.

**nullifiers**: Revealed by Spend descriptions when its associated note is spent.

**nullifier set**: The set of unique nullifiers revealed by any transactions
within a block. Nullifiers are enforced to be unique within a valid block chain
by commiting to previous treestates in Spend descriptions, in order to prevent
double-spends.

**note commitments**: Pedersen commitment to the values consisting a note. One
should not be able to construct a note from its commitment.

**note commitment tree**: An incremental Merkle tree of fixed depth used to
store note commitments that JoinSplit transfers or Spend transfers produce. It
is used to express the existence of value and the capability to spend it. It is
not the job of this tree to protect against double-spending, as it is
append-only: that's what the nullifier set is for.

**note position**: The index of a note commitment at the leafmost layer,
counting leftmost to rightmost. The [position in the tree is determined by the
order of transactions in the block](https://zips.z.cash/protocol/canopy.pdf#transactions).

**root**: The layer 0 node of a merkle tree.

**anchor**: A Merkle tree root of a note commitment tree. It uniquely identifies
a note commitment tree state given the assumed security properties of the Merkle
tree’s hash function.  Since the nullifier set is always updated together with
the note commitment tree, this also identifies a particular state of the
associated nullier set.

**spend descriptions**: A shielded Sapling transfer that spends a note. Includes
an anchor of some previous block's note commitment tree.

**output descriptions**: A shielded Sapling transfer that creates a
note. Includes the u-coordinate of the note commitment itself.

**joinsplit**: A shielded transfer that can spend Sprout notes and transparent
value, and create new Sprout notes and transparent value, in one Groth16 proof
statement.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

As blocks are validated, the note commitments revealed by all the transcations
within that block are used to construct note commitment trees, with the note
commitments aligned in their note positions in the bottom layer of the Sprout or
Sapling note tree from the left-most leaf to the right-most in transaction order
in the block. So the Sprout note commitments revealed by the first
JoinSplit<Groth16> in a block would take note position 0 in the Sprout note
commitment tree, for example. Once all the transactions in a block are parsed
and the notes for each tree collected in their appropriate positions, the root
of each tree is computed. While the trees are being built, the respective block
nullifier sets are updated in memory as note nullifiers are revealed. If the
rest of the block is validated according to consensus rules, that root is
committed to its own datastructure via our state service (Sprout anchors,
Sapling anchors). Sapling block validation includes comparing the specified
FinalSaplingRoot in its block header to the root of the Sapling note commitment
tree that we have just computed to make sure they match.

As the transactions within a block are parsed, Sapling shielded transactions
including Spend descriptions and Output descriptions describe the spending and
creation of Zcash Sapling notes, and JoinSplit-on-Groth16 descriptions to
transfer/spend/create Sprout notes and transparent value. JoinSplit and Spend
descriptions specify an anchor, which references a previous note commitment tree
root: for Spend's, this is a previous block's anchor as defined in their block
header, for JoinSplit's, it may be a previous block's anchor or the root
produced by a strictly previous JoinSplit description in its transaction. For
Spend's, this is convenient because we can query our state service for
previously finalized Sapling block anchors, and if they are found, then that
[consensus check](https://zips.z.cash/protocol/canopy.pdf#spendsandoutputs) has
been satisfied and the Spend description can be validated independently. For
JoinSplit's, if it's not a previously finalized block anchor, it must be the
treestate anchor of previous JoinSplit in this transaction, and we have to wait
for that one to be parsed and its root computed to check that ours is
valid. Luckily, it can only be a previous JoinSplit in this transaction, and are
[usually the immediately previous one](zcashd), so the set of candidate anchors
is smaller for earlier JoinSplit's in a transcation, but larger for the later
ones. For these JoinSplit's, they can be validated independently of their
anchor's finalization status as long as the final check of the anchor is done,
when available, such as at the Transaction level after all the JoinSplit's have
finished validating everything that can be validated without the context of
their anchor's finalization state.

So for each transaction, for both Spend descriptions and JoinSplit's, we can
pre-emptively try to do our consensus check by looking up the anchors in our
finalized that first. For Spend's, we then trigger the remaining validation and
when that finishes we are full done with those. For JoinSplit's, the anchor
state check may pass early if it's a previous block Sprout note commitment tree
root, but it may fail because it's an earlier JoinSplit's root instead, so once
the JoinSplit validates independently of the anchor, we wait for all candidate
previous JoinSplit's in that transaction finish validating before doing the
anchor consensus check again, but against the output treestate roots of earlier
JoinSplit's.

Both Sprout and Sapling note commitment trees must be computed for the whole
block to validate, but Sprout, we need to compute interstitial treestates in
between JoinSplit's in order to do the final consensus check for each/all
JoinSplit's, not just for the whole block, like in Sapling.

For Sapling, at the block layer, we can iterate over all the transactions in
order and if they have Spend's and/or Output's, we update our Nullifer set for
the block as nullifiers are revealed in Spend descriptions, and update our note
commitment tree as note commitments are revealed in Output descriptions, adding
them as leaves in positions according to their order as they appear transaction
to transction, output to output, in the block. This can be done independent of
the transaction validations. When the Sapling transactions are all validated,
the note commitment tree root should be computed: this is the anchor for this
block. For Sapling and Blossom blocks, we need to check that this root matches
the `RootHash` bytes in this block's header, as the `FinalSaplingRoot`. Once all
other consensus and validation checks are done, this will be saved down to our
finalized state to our `sapling_anchors` set, making it available for lookup by
other Sapling descriptions in future transactions.

For Sprout, we must compute/update interstitial note commitment trees between
JoinSplit's that may reference an earlier one's root as its anchor. If we do
this at the transaction layer, we can iterate throught all the JoinSplit's and
compute the Sprout note commitment tree and nullifier set similar to how we do
the Sapling ones as described above, but at each state change (ie,
per-JoinSplit) we note the root and cache it for lookup later. As the
JoinSplit's are validated without context, we check for its specified anchor
amongst the interstitial roots we've already calculated (according to the spec,
these interstitial roots don't have to be finalized or the result of an
independently validated JoinSplit, they just must refer to any prior JoinSplit
root in the same transaction). So we only have to wait for our previous root to
be computed via any of our candidates, which in the worst case is waiting for
all of them to be computed for the last JoinSplit. If our JoinSplit's defined
root pops out, that JoinSplit passes that check.

To finalize the block, the Sprout and Sapling treestates are the ones resulting
from the last transaction in the block, and determines the Sprout and Sapling
anchors that will be associated with this block as we commit it to our finalized
state. The Sprout and Sapling nullifiers revealed in the block will be merged
with the exising ones in our finalized state (ie, it should strictly grow over
time).


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation


# Drawbacks
[drawbacks]: #drawbacks



# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives


# Prior art
[prior-art]: #prior-art


# Unresolved questions
[unresolved-questions]: #unresolved-questions


# Future possibilities
[future-possibilities]: #future-possibilities
