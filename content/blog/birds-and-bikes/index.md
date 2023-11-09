+++
title = "it's like riding a bike doesn't mean the same thing for birds"
slug = "bikes-and-birds"
date = "2023-11-09"
+++

In this post, I examine the LeetCode problem, [1057. Campus Bikes](https://leetcode.com/problems/campus-bikes/description/).

>On a campus represented on the X-Y plane, there are n workers and m bikes, with n <= m.
>
>You are given an array workers of length n where workers[i] = [xi, yi] is the position of the ith worker. You are also given an array bikes of length m where bikes[j] = [xj, yj] is the position of the jth bike. All the given positions are unique.
>
>Assign a bike to each worker. Among the available bikes and workers, we choose the (workeri, bikej) pair with the shortest Manhattan distance between each other and assign the bike to that worker.
>
>If there are multiple (workeri, bikej) pairs with the same shortest Manhattan distance, we choose the pair with the smallest worker index. If there are multiple ways to do that, we choose the pair with the smallest bike index. Repeat this process until there are no available workers.
>
>Return an array answer of length n, where answer[i] is the index (0-indexed) of the bike that the ith worker is assigned to.
>
>The Manhattan distance between two points p1 and p2 is Manhattan(p1, p2) = |p1.x - p2.x| + |p1.y - p2.y|.

LeetCode present a couple of solutions, namely (with `N` as the number of
workers, `M` as the number of bikes, and `K` as the maximum possible
`dist(worker,bike)` where `dist` is Manhattan distance as defined above): 

- Generate all possible `(worker, bike)` pairs and sort them in order. Then
  iterate over the sorted pairs and assign a worker to each pair. Time
  Complexity: `O(NM log(NM))`
- Bucket Sort: `O(NM + K)`
- Priority Queue: `O(NM log(M))`

#### A different approach

We're going to explore a modified BFS algorithm to solve this problem. We look
at the following pseudo-code:

1. define `grid: Vec<Vec<Option<i32>>` s.t. `grid[i][j] = Some(idx)` i.f.f. there exists a bike at `i,j`
2. use a `seen` array for each worker (we can use `seen[i][j][k]` to denote
   when worker ,`i`, has seen `grid[i][j]`)
3. Initialize state variables to track which bicycle is assigned to which
   worker. i.e. (`let mut ans = vec![i32:N]`, s.t. `ans[i] = j` implies
   worker `i` will receive bike `j`)
4. Instantiate a queue of 4-pairs, `(worker, i, j, dist)`, to perform bfs. We maintain two invariants.
    -  for each pair `(w_a, i,j, d_a)`, there exist no pairs `(w_b, k,l, d_b)`
       s.t. `w_b > w_a` and `d_b < d_a`. That is, because of breadth-first
       traversal every item after the current item in the queue is either from
       a later worker and dist of that item is >= the current item or it is
       from an earlier worker and dist > the current item.
3. for each `worker`:
    1. We insert `(i, y_i, x_i, 0)` into the queue where `y_i, x_i` are the row
       and column of the `i`th worker's position.
    2. We assign `seen[i][y_i][x_i] = true`
4. We now perform bfs, 
    1. pop an item: `(worker, y,x, dist)` from the queue,
    2. if worker has found a bike and `grid[found[worker]]` is empty, 
        continue
    2. if the item is from a worker which hasn't found a bike,
        1. if `y,x` is a bike, optimistically update the worker's found bike to 
            this bike
        2. o/w - put the neighbors on the queue
    3. o/w,
        1. if `y,x` corresponds to a bike:
            - if that bike's distance is == the found bike,
            - update the worker's found bike to be the bike with the lower idx
    4. if `dist != last_dist` or `worker != last_worker`
        1. if worker has found a bike and that bike is still on the grid,
        2. remove it from the grid
            
###### Wat

Why have I done this? For every ordering of items, there exists a traversal
through a graph that represents them. We let each edge `u -> v` represent that
`u < v`. In `O(NM)` time, we should be able to construct a graph `G` s.t.
`dist(u,v) = x` i.f.f. `w(e -> v) = x`. We then add a source node which points
to `w_0` and a dest which is arrived at from `w_n`. So long as we maintain
sub-ordering within the edge sets (which may require and additional `log M`), the shortest path from  source -> destination 
is the ordering we are searching for. BFS will get us a shortest path in 
`O(N + M)` which yields a total running time of `O(NM log(M) + N + M)`. 

The question is whether that can be improved by avoiding the ordering of the
edge sets. 
