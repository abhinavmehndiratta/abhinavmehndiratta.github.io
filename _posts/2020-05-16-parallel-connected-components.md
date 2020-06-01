---
layout: post
title:  "Experiments with parallel connected components"
---
<img src="../assets/cc.png">
A connected component of an undirected graph is a maximal set of nodes such that each pair of nodes is connected by a path. Connected components can be identified serially using simple traversal (BFS/DFS) or the Union-Find algorithm in `O(|V|+|E|)` time. Generally, Union-Find performs better and I'll be using it to compare benchmarks with the parallel code.

I tried two different approaches based on multi-threading for finding weakly connected components. I'll try to describe them here.

## Hooking/Pointer Jumping
Connected components can be viewed as a problem of placing nodes in disjoint sets, where nodes belonging to the same set are connected by a path. We use any array `Parent` to denote the membership of vertex `v` in set `Parent[v]`. Initially `Parent[v] = v` for each vertex.

The basic idea behind this approach is to contract the graph in each iteration by chosing a subset of edges(u => v) and marking `Parent[u] = Parent[v]`. These edges are then removed for the next iteration. This is demonstrated below -

<img src="../assets/red.png">

There are many different ways to do this, each of these methods reduce the number of vertices to atleast half after each iteration, which results in `log(|V|)` number of steps, where each step takes a maximum of `O(|V|+|E|)` time.

**Edge contraction:**
Ideally, after each step the edges(u => v) where u, v belong the same set should be removed. However, edge contraction is an expensive operation and while the number of vertices are halved after each iteration there is no guarantee that the number of edges will be reduced by the same factor.Therfore, I didn't use contraction for the tests. Although, edge contraction might be useful for huge graphs with uniform degree distribution.

**Partitioning:**
The first step is partitioning the graph and assigning almost equal number of edges to each thread. LightGraphs stores a graph in the form of sorted adjacency lists, so it is not possible to directly use `@threads` macro for `edges(g)` since we cannot do `getindex` operation on the iterator. Although we could write an edge iterator which supports `@threads`, I'm not sure if it'll be effecient since each `getindex` call will take `log(|V|)` time. So, I used a greedy function that partitions the vertex set based of degree.

### Random Hooking
Basically how this works is you perform an unbiased coin toss and assign a value of HEAD/TAIL to every vertex. Then you iterate over all edges and check if one end 

### Deterministic Hooking


### Pointer Jumping
This method is based on *"Connected-Components algorithms for Mesh-Connected Parallel Computers
Goddard, Kumar and Prins"*.


## Parallel Lock-Free Disjoint Set
The serial implementation of Disjoint-set with Path Compression is really efficient. The complexity of `find` and `union` operations is even smaller than `O(log(N))`. In fact, amortized time complexity effectively becomes small constant. So although I wasn't expecting to see a significant speedup using this approach, I decided to try it anyway.

This implementation is based on *"Wait-free Parallel Algorithms for the Union-Find Problem by Richard J. Anderson and Heather Woll"*, with a slight modification; it uses randomized linking, i.e. instead of using `rank` for performing the `union` operation it uses a random value assigned to each vertex (Id array in the code below) to chose the parent (the one with higher value of Id becomes the new parent). This reduces the overhead of atomic operations involved in updating `rank`, while maintaining the same time complexity. We use `One-try splitting` for path compression that tries once to change the parent of a node to its grandparent, after which it moves on to the next node on the find path. This approach is described in detail in *"A Randomized Concurrent Algorithm for Disjoint Set Union by V. Jayanti and E. Tarjan"*.

Here is the code for our new thread-safe data structure ParallelDisjointSet -

{% highlight julia %}

struct ParallelDisjointSet{T <: Integer}
    Parent::Vector{Atomic{T}}
    Id::Vector{T}
end

ParallelDisjointSet{T}(n::Integer) where T =
    ParallelDisjointSet([Atomic{T}(i) for i in 1:n], shuffle(T(1): T(n)))

function find(D::ParallelDisjointSet{T}, x::T) where T
    Parent = D.Parent
    u = x
    while true
        v = Parent[u][]
        w = Parent[v][]
        v == w && return v
        atomic_cas!(Parent[u], v, w)
        u = v
    end
end

function union(D::ParallelDisjointSet{T}, x::T, y::T) where T
    Parent = D.Parent
    Id = D.Id
    u = x
    v = y
    while true
        u = find(D, u)
        v = find(D, v)
        u == v && return
        if Id[u] < Id[v]
            atomic_cas!(Parent[u], u, v) == u && return
        else
            atomic_cas!(Parent[v], v, u) == v && return
        end
    end
end

{% endhighlight %}

`atomic_cas!(x, cmp, newval)` atomically compares the value in x with cmp. If equal, it writes the newval to x, else leaves x unmodified. It returns the old value stored in x. By comparing the returned value to cmp, we check if the operation was successful.

## Benchmarks
**On 6 cores**


Clearly, Pointer Jumping outperforms every other approach.
