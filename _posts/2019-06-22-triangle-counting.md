---
layout: post
title:  "Triangle Counting"
---

Triangle counting is an important problem in graph mining. Two frequently used metrics in complex network analysis that require the count of triangles are the clustering coefficients & the transitivity ratio of the graph. Triangle counting is used in several real-world applications, such as detection of spamming activity, uncovering the hidden thematic structure of the web & link recommendation in online social networks.
Clustering coefficient is used as an index for measuring the concentration of clusters in graphs & its tendency to decompose into communities. It has also been demonstrated that the age of a community is related to the density of triangles i.e., when a group has just formed, people pull in their like-minded friends, but the number of triangles is relatively small. If A brings in friends B & C, it may well be that B & C do not know each other. As the community matures, B & C may interact because of their membership in the community. Thus, there is a good chance that at sometime the triangle {A,B,C} will be completed.

In this post, I'll implement an algorithm for counting triangles in a graph using [SuiteSparseGraphBLAS][ssgb] in Julia. The past couple of weeks I've been working on creating a simpler interface for creating GraphBLAS matrices, vectors & descriptors. I'll be using those new interface functions here to create GraphBLAS objects.

First we create a new GraphBLAS matrix using the facebook graph from SNAP Datasets.
{% highlight julia %}
julia> using SuiteSparseGraphBLAS, SparseArrays, LightGraphs, SNAPDatasets, BenchmarkTools

julia> g = loadsnap(:facebook_combined)
{4039, 88234} undirected simple Int64 graph

julia> I, J, X = SparseArrays.findnz(adjacency_matrix(g));

julia> GrB_init(GrB_NONBLOCKING)
GrB_SUCCESS::GrB_Info = 0

julia> A = GrB_Matrix(I, J, X)
GrB_Matrix{Int64}
{% endhighlight %}

Now we'll write the `count_triangles` function. For more details on the algorithm refer [here][KokkosKernels].
{% highlight julia %}
julia> function count_triangles(A)
               L = LowerTriangular(A)
               C = GrB_Matrix(Int64, size(A)...)

               # Descriptor for mxm
               desc_tb = GrB_Descriptor(Dict(GrB_INP1 => GrB_TRAN)) # transpose the second matrix

               GrB_mxm(C, L, GrB_NULL, GxB_PLUS_TIMES_INT64, L, L, desc_tb) # C<L> = L ∗.+ L'
               ntriangles = GrB_reduce(GrB_NULL, GxB_PLUS_INT64_MONOID, C, GrB_NULL)

               GrB_free(C)
               GrB_free(L)

               return ntriangles
       end
count_triangles (generic function with 1 method)

julia> @btime count_triangles(A)
  19.860 ms (129 allocations: 5.33 KiB)
1612010
{% endhighlight %}

Julia's standard library doesn't support masked matrix operations, so we'll use another method which is similar to the above algorithm in order to compare benchmarks.
{% highlight julia %}
julia> using LightGraphs, SNAPDatasets, LinearAlgebra, BenchmarkTools

julia> adj = adjacency_matrix(loadsnap(:facebook_combined));

julia> function tricount(A)
               L = tril(A)
               ntri = sum(sum((L*L).*L))
               return ntri
       end
tricount (generic function with 1 method)

julia> @btime tricount(adj)
  30.221 ms (9869 allocations: 14.03 MiB)
1612010
{% endhighlight %}

Seems like the masked version is a bit faster.

This is just a simple & fast approach to triangle counting & there might be better ways of implementing this using Julia's standard library or GraphBLAS but the purpose here is to illustrate the new higher level Julian interface for GraphBLAS.

[ssgb]:https://github.com/abhinavmehndiratta/SuiteSparseGraphBLAS.jl
[KokkosKernels]:http://faculty.cse.tamu.edu/davis/GraphBLAS_files/Davis_HPEC18.pdf
