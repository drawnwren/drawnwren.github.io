+++
title = "it's like riding a bike doesn't mean the same thing for birds"
slug = "bikes-and-birds"
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


