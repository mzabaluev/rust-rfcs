- Feature Name: golden_chunk
- Start Date: 2017-12-04
- RFC PR: 
- Rust Issue: 

# Summary
[summary]: #summary

Add methods to the `Alloc` trait providing hints on preferable chunk
allocation sizes for a few common use cases.

# Motivation
[motivation]: #motivation

There is a number of dynamic data container designs that allocate memory in
equally-sized chunks that are filled with elements until there is a need
to allocate more chunks. This is done to reduce potentially expensive
allocations and/or large data relocations when the data structure is
repeatedly modified. Examples are non-reallocating segmented vectors,
also B-trees.
In implementation of such data structures it's often desirable to come up with
the chunk size that provides some good tradeoff between frequency of
time-consuming allocations and utilization of the allocated memory.
Many implementations take some power of two as a guesstimate, or try to make
the chunk size close to the size of an OS memory page. The optimal size,
however, is dependent on the allocator and is often somewhat less than the
nearest power of two due to memory overhead imposed by the allocator. The
size and offset of the data elements may also affect the optimal
chunk size.

## "fast" chunks: Optimizing allocation performance

Global allocators typically try to satisfy an allocation request from
available free chunks (of varying sizes), and if no suitable chunks are found,
request more memory from the OS. We'll use the moniker "morecore" for
the latter path, to borrow from decades-old terminology found inside glibc's
implementation of `malloc()`. As the block-structured containers essentially
implement their own specialized layer of memory management, it's beneficial
for their allocation patterns to consistently hit the "morecore" path in
the allocator after the supply of suitable free chunks is exhausted,
to make it perform the dirty work that only it can do and minimize shuffling of
heap data structures.

## "bulk" chunks: Minimizing allocation waste

Allocator waste comes in two flavors: internal data structure overhead
and heap fragmentation. An application that prefers most efficient utilization
of heap memory to faster allocations would need allocation patterns that do
not produce fragmented unallocated chunks on repeated allocations. Overhead
minimization has to be balanced against the application's tolerance for
overcommitted memory.

## Case study: musl malloc

[musl](https://www.musl-libc.org/) has a relatively simple heap
implementation derived from dlmalloc.
Free chunks (up to a certain large size which is managed with mmap)
are placed into bins indexed by the power of two fitting the size of the
bin's chunks. When a request cannot be satisfied from available free chunks,
more heap memory is requested from the OS with the `brk` system call,
with the increment rounded up to a multiple of the virtual memory page size,
and the chunk is allocated from the newly available memory.
There is a `2 * sizeof(size_t)` overhead in every allocated chunk.
So, a local waste/performance optimum may be expected in the largest
chunk size that fits in the memory page size together with the overhead.

## Case study: glibc malloc

GNU libc has a complicated implementation that is a much more evolved
descendant from dlmalloc. For our analysis, the most important is a
separation to "small" and "large" bins. Memory allocations of up to
1000 bytes are placed into "small" chunks which are faster to work
with because a small chunk is treated as of essentially the same size
as any others in its linearly indexed bin, and much of the logic regarding
chunk sizes is not performed on small chunks.

This is where the difference between "fast" and "bulk" optimizations may
become apparent: for faster allocation and deallocation, it may be optimal to
fill the largest possible "small" chunk, but the "bulk" optimization would
try to avoid as much chunk overhead as possible within the limit
on overcommitment, and so it may aim for a multiple of the memory page size
minus the overhead of the single chunk.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

New methods are to be added to the `Alloc` trait:

```rust
    fn fast_chunk_layout(&self, elem_layout: Layout) -> ChunkLayout;
    fn bulk_chunk_layout(&self, elem_layout: Layout,
                         overcommit_hint: usize) -> ChunkLayout;
    fn fast_array_layout<T>(&self) -> ChunkLayout;
    fn bulk_array_layout<T>(&self,
                            overcommit_hint: usize) -> ChunkLayout;
```

`fast_chunk_layout` returns the recommended allocation layout for a
contiguous chunk of instances of the given layout, for applications
that allocate and deallocate _often_. The returned chunk size is not
expected to be large; the size of the operating system's virtual memory page
can be a reasonable upper bound for the implementation, unless the element
layout is larger than that.

`bulk_chunk_layout` returns the recommended chunk layout for applications
that allocate _a lot_. The `overcommit_hint` parameter indicates how
many elements the application expects to have per chunk; passing 0 is a
valid way to suggest conservatively sized chunks (with the logic that 0
should not result in bigger chunks than 1). The implementation may choose
to recommend smaller chunks than hinted by the application.

The `*_array_layout` methods are type-parameteric sugar for their
`Layout`-taking counterparts.

An implementation never panics on valid layout inputs and is guaranteed to
return a layout of non-zero length. Note that it's perfectly valid for
the implementation to return the layout for a one-element chunk; it's up to
the application to impose a minimum chunk length if single-element chunks
are undesirable.

The structure `ChunkLayout` builds upon `Layout` to add information on
the number of elements in the chunk. It will have, at minimum, this public
API:

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct ChunkLayout { /* ... */ }

impl ChunkLayout {
    pub fn len(&self) -> usize { /* ... */ }
    pub fn elem(&self) -> Layout { /* ... */ }
}

impl From<ChunkLayout> for Layout { /* ... */ }
impl<'a> From<&'a ChunkLayout> for Layout { /* ... */ }
impl AsRef<Layout> for ChunkLayout { /* ... */ }
```

For applications that want chunks sized to a power of two,
more `Alloc` methods returning the layout and the exponent
(i.e. the bit shift value) can be made available:

```rust
    fn fast_chunk_layout_po2(&self, elem_layout: Layout) -> (ChunkLayout, u8);
    fn fast_array_layout_po2<T>(&self) -> (ChunkLayout, u8);
```

The added `Alloc` methods are implemented by default in an unspecified way
that is expected to give reasonably good performance with any general-purpose
allocator.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The default implementation of both `fast_chunk` and `bulk_chunk` will try
to fit the chunk in a target-tuned guess at the memory page size,
minus the commonly used overhead of `2 * size_of::<usize>()`, if no other
target-specific amount is found to be more optimal.

# Drawbacks
[drawbacks]: #drawbacks

The proposed design may be overengineered; some use case survey might be
necessary before stabilizing it.

A burden of fine tuning of the size hints is placed on the implementor of
`Alloc`, and it is easy to be guessed wrong or get bit-rotten later if e.g.
the underlying foreign library implementation changes, resulting in suboptimal
performance.

# Rationale and alternatives
[alternatives]: #alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
