- Feature Name: `block-manager`
- Start Date: 2018-03-13
- RFC PR: (leave this empty)
- Sawtooth Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC describes new architectural components of the validator that handle
the management of blocks correctly and help remove a known race condition that
can cause network nodes to fork.

# Motivation
[motivation]: #motivation

The RFC aims to improve Sawtooth in two ways:

1. By removing a race-condition  which can cause parent blocks required for
   validation of some block to be dropped while the block is transferred from
   the completer to the chain controller.
2. Providing additional guarantees about the blocks being processed.
3. Simplifying and generalizing the procedure for persisting blocks.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This RFC aims to improve the handling of blocks by introducing some new classes
and interfaces. The requirements for the design presented in this RFC are that
it:

- Shall store blocks for later
- Shall ensure integrity of blocks stored
- Shall provide a mechanism for preventing required blocks from being dropped
- Shall provide a mechanism for persisting to a store

The constraints for the design presented in this RFC are that:

- The API must work for both forking vs non-forking consensus mechanisms
- The API defined must be reusable, meaning it provides generic methods that
  make few assumptions about how the API will be used

The following new concepts are used by this design and are defined here:

For a collection of blocks, __integrity__ is defined to mean that every block
in the collection is accompanied by its parent.

A __branch__ is a collection of blocks such that:

1. The blocks are sorted in order of increasing block number
2. The branch has integrity

A __block tree__ is a collection of branches with a single common ancestor or
__root__.

The following is a visualization of an example block tree containing blocks
{0, A, B, C, ..., H}.

```
0 (root)
|
|\
A |
| E
|\
B |
| F
| |\
C | |
| G |
|   H
D
```

In the above example, the block tree contains multiple branches. They are (0,
A, B, C, D), (0, E), (0, A, F, G), and (0, A, F, H).

In order to satisfy the requirements and constraints of this RFC, the
__block manager__ class and __block store__ interface were created.

A __block store__ is an interface which supports storing and retrieving blocks.
It can be backed by any type of storage, for example disk or network. Because a
blockchain's length grows indefinitely, part of it eventually has to be
archived in some way. The __block store__ interface is intended to help solve
the problem of persisting parts of the __block tree__ in a way that does not
make assumptions about how it is persisted or what parts of the tree are
persisted.

The __block manager__ is a collection of __block tree__ s (though it usually
contains 1) that also supports methods for:

- Ensuring that blocks depended on by external objects are not removed.
- Persisting blocks to a __block store__.

In addition to the __block manager__ class and __block store__ interface, a
__commit store__ class was created The commit store is a __block store__
implementation that can be added to the block tree for persisting committed
blocks to disk. It also implements an LRU cache which holds some number of most
recently accessed blocks in memory. The __commit store__ replaces the what was
previously called the "block store".

## Block Manager API

The following is a listing of the __block manager__ methods and their function:

`put(branch: List<Block>)`

Atomically adds the given blocks to the tree.

The blocks passed must form a branch meaning they satisfy the following
conditions:

1. The first block's parent is present
2. The blocks are sorted in order of increasing block num
3. For any two sequential blocks in the list, the second block's parent is
   the first block

If any of these three conditions are not met, an exception is raised.

All blocks added will have a reference count of 1 if the operation is
successful, including the tip.

`get(block_ids: List<String>) -> Iter<Block>`

    Atomically get the blocks with the given ids. This passes through to any
    registered stores as necessary.

`branch(tip: String) -> Iter<Block>`

    Create an iterator over blocks on a given branch starting with the block
    with the id given in `tip` and traversing from the tip to the root.

`branch_diff(tip: String, exclude: String) -> Iter<Block>`

    Create an iterator over blocks on a given branch starting with the block
    with the id given in `tip` and traversing from the tip to the root,
    excluding blocks that are also in the branch starting with id in exclude.

### Holding Blocks

In order to avoid removing blocks that are in the process of being worked on,
the block manager provides an operation to place a `hold` on a block. When this
operation is called, the block manager makes a guarantee that the block will
not be dropped until corresponding `drop` is called. Consequently, when an
object that requests a hold is done with the block, it must call `drop` to
signal it is done with the block.

Ensuring that blocks depended on by external objects are not removed is handled
by the following methods:

`ref_block(block_id: String)`

    Ensure that the block with the given block id is not dropped until a
    corresponding `unref_block()` with the same block id is called.

`unref_block(block_id: String)`

    Release a previous `ref_block()` on a block. May result in the block being
    removed from the tree.

    Raises an exception if the reference count falls below 0, which indicates
    a bug in the application code.

The following example illustrates how the `ref_block` operation is used to
prevent blocks from being dropped when they are transferred from the completer
to the chain controller.

Assume that the chain controller has a block A and that the completer has just
completed a block B and would like to pass it to the chain controller. If the
completer were to just pass the chain controller B through a queue or shared
memory, the chain controller could decide to `unref_block` A before it receives
B, causing B to have a missing predecessor when it arrives at the chain
controller. Instead, the completer first places a `ref_block` on A, causing and
then it passes B to the chain controller. If the chain controller decides to
`unref_block` A now, the block manage will ensure it is not actually dropped,
since there is still an open hold on the block. Finally, when the chain
controller takes ownership of B, it can send a signal back to the completer
that it has been received and the completer can release its hold.

### Persisting Blocks

In order to support persisting blocks to an alternative not-in-memory location,
a generic __block store__ interface is defined and the __block manager__
supports adding any number of block stores and transferring ownership of the
blocks to these stores.

Persisting blocks is handled by the following methods:

`add_store(store_name: String, store: Store)`

    Take ownership of the given store and enable it for persisting blocks. The
    store must be referenced from other methods with `store_name`.

    Raises an exception if a store with `store_name` already exists.

`persist(head: String, store_name: String)`

    Atomically ensure that all blocks on the branch starting with head are
    in the store.

## Block Store Interface

The __block store__ interface is the interface that must be implemented by an
object in order for it to be added to the block tree as a "store". The
following is a listing of the interface's methods:

`put(blocks: List<Block>)`

    Atomically add the given blocks to the store.

`get(block_ids: List<String>) -> Iter<Block>`

    Atomically get the blocks with the given ids.

`iter() -> Iter<Block>`

    Create an iterator over all blocks in the store.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The following provides more detail on how the API present above is implemented.

## Internal Representation

Internally, the block manager consists of:

- A __main cache__ which, at a minimum, contains all blocks that have not been
  persisted.
- A map of store names to stores.

Internally, the commit store consists of two components:

- A database object which is responsible for persisting the committed fork to
  disk and retrieving committed blocks from disk.
- An __LRU cache__, which is responsible for ensuring that frequently accessed
  blocks do not incur database reads. All writes are done directly to disk, but
  this behavior could be changed in the future. The LRU cache keeps the last N
  recently used blocks, where "used" can mean either a read or a write.

## Ensuring Integrity

As is heavily implied by the API, the block manager uses __reference counting__
internally to guarantee integrity and to allow external components to place
holds on specific blocks. For every block managed by the block manager, a
reference count is maintained (sometimes implicitly).

This one reference count is a sum of both:

1. The count of all other blocks in the block manager that depend directly
   on this block
2. The count of all calls to `ref_block` minus the count of all calls to
   `unref_block`

In order to avoid keeping a reference count and copy of every persisted block
in the main cache, __anchors__ are placed in the main cache to indicate that
the rest of the branch has been persisted to some store. An __anchor__ is used
to represent a block that is in a store and not in the __main cache__ and
consists of:

- The block id of the block that is in the store
- A list of names of stores that the block is in
- A reference count of blocks that depend on the anchor but are not in a store
  the block is in

A block in a store and not the __main cache__ has an implicit reference count
of 1 if it is not the tip of the branch in the store. If it is the tip of the
branch, it has an implicit reference count of 0.

The following describes the algorithms used to ensure integrity for the
important operations on the block manager.

**Adding a branch**

The following algorithm is used to add a single block to the block manager:

1. If the first block's parent is not in the main cache and its parent is in
   some store:

  a. If the parent is the chain head, create a new anchor in the main cache for
     the parent with reference count 0.
  b. Else, create a new anchor in the main cache for the parent with reference
     count 1.

2. For each block in the branch:

  a. If the block's parent (can be an anchor) is in the main cache, add it to
     the main cache with a reference count of 0 and increment its parent's
     reference count by 1.
  b. Else, return an error.

3. Increment the last block's reference count by 1.

**Removing a block or branch**

There is no explicit `remove` operation for the block manager. Instead, the
`unref_block` operation is used to signal the an object is done with a block.
When a reference count for a block falls to 0, it is no longer accessible to
client code and will be purged.

The algorithm for unref'ing a block is:

1. If the block is in a store, stop.
2. If the block is in main cache:

    a. Decrement the block's reference count
    b. If the block's reference count is less than 0, return an error.
    c. If the block's reference count is 0, delete it from the main cache.
    d. If the block's parent is an anchor, decrement its reference count
       and if the reference count falls to 0, remove the anchor. Stop.
    e. Else, repeat a-e with its parent.

3. Return a "not found" error

**Ensuring Holds are Released**

Rather than requiring objects that place holds on blocks to remember to release
them when they are done, this can be done automatically depending on the
language used.

In Python, the `ref_block_ctx` operation returns a context manager to use with
a `with` block. The `ref_block` operation is performed in `__enter__` and the
`unref_block` done in `__exit__`.

    with manager.ref_block_ctx(block_id) as block: # block is held and retrieved
        process_block(block)
        # block is dropped at the end of the block

In Rust the `ref_block_ctx` operation returns a struct that implements `Drop`
to handle the `unref_block` operation.

    {
        let block_ctx = manager.ref_block_ctx(block_id); // block is held and retrieved
        process_block(block_ctx.block);
    } // block is dropped when block_ctx goes out of scope

This can also be used to ensure that entire branches are not dropped.

# Drawbacks
[drawbacks]: #drawbacks

There is no reason not to do this.

# Rationale and alternatives
[alternatives]: #alternatives

Using a design based on reference counting is the only known way to ensure that
required blocks are not accidentally dropped. This design does not make
assumptions about the meaning of added stores and so may be slightly less
efficient, but the benefit is the API is more general and better satisfies
anticipated future use cases.

# Prior art
[prior-art]: #prior-art

The previous solution to the problem this design solves used a time cache that
has resulted in many race conditions and a poorly performing system. It also
did not provide a way for external objects to ensure required blocks were not
dropped.

# Unresolved questions
[unresolved]: #unresolved-questions

- Can blocks be allowed to live in both a store and the main cache, or do they
  have to live in either the main cache or one or more stores? Can blocks
  live in both places and still have the block manager be memory/storage
  efficient?
- What is the algorithm for correctly persisting a block to a store?
