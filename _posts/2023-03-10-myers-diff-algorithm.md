---
title: Myers Diff Algorithm
author: mingqing
date: 2023-03-10 15:13:00 +0800
categories: [Algorithm]
tags: [myers, diff, algorithm, lcs, git]
math: true
mermaid: true
---

起初源于刚接触`git diff`时，对命令的输入结果无法理解。故查阅了相关资料，从`--diff-algorithm={patience|minimal|histogram|myers}`可以看出命令内置了多种算法，默认为myers。

抱着好奇心搜寻，发现中文互联网少有文章理解透彻。结合myers的[原论文](http://www.xmailserver.org/diff2.pdf)以及相关[英文资料](https://blog.jcoglan.com/2017/02/12/the-myers-diff-algorithm-part-1/)从经典的动态规划LCS出发，一步一步优化状态和复杂度，谈谈我的理解。

___

## **Problem Modeled**

Diff between two strings is modeled as a graph search problem.

```
String a = "ABCABBA"
String b = "CBABAC"
```

```
    A     B     C     A     B     B     A
    o-----o-----o-----o-----o-----o-----o-----o   0
    |     |     | \   |     |     |     |     |
C   |     |     |  \  |     |     |     |     |
    |     |     |   \ |     |     |     |     |
    o-----o-----o-----o-----o-----o-----o-----o   1
    |     | \   |     |     | \   | \   |     |
B   |     |  \  |     |     |  \  |  \  |     |
    |     |   \ |     |     |   \ |   \ |     |
    o-----o-----o-----o-----o-----o-----o-----o   2
    | \   |     |     | \   |     |     | \   |
A   |  \  |     |     |  \  |     |     |  \  |
    |   \ |     |     |   \ |     |     |   \ |
    o-----o-----o-----o-----o-----o-----o-----o   3
    |     | \   |     |     | \   | \   |     |
B   |     |  \  |     |     |  \  |  \  |     |
    |     |   \ |     |     |   \ |   \ |     |
    o-----o-----o-----o-----o-----o-----o-----o   4
    | \   |     |     | \   |     |     | \   |
A   |  \  |     |     |  \  |     |     |  \  |
    |   \ |     |     |   \ |     |     |   \ |
    o-----o-----o-----o-----o-----o-----o-----o   5
    |     |     | \   |     |     |     |     |
C   |     |     |  \  |     |     |     |     |
    |     |     |   \ |     |     |     |     |
    o-----o-----o-----o-----o-----o-----o-----o   6
    0     1     2     3     4     5     6     7
```

It recorded a trace through the graph to <mark>find the shortest path from (0,0) to the bottom-right corner(7,6).</mark>

> 这里还是属于基本的LCS动态规划范畴，在路径图中的移动即为状态转移：以A字符串为源的删除（向右）、插入（向下）和纳入相同字符范畴（对角线）
{: .prompt-tip }

```
0,0 --- 1,0 --- 3,1 --- 5,2 --- 7,3
|       |       |
|       |       |
0,1     2,2     5,4 --- 7,5
|               |       |
|               |       |
2,4 --- 4,5     5,5     7,6
|       |       |
|       |       |
3,6     4,6     5,6
```

Now, having seen how the graph search works, we’re going to change the representation slightly to get us toward how the Myers algorithm actually works. Imagine that we take the above graph walk and render it rotated by 45 degrees.

```
k   | d    0     1     2     3     4     5  
----+--------------------------------------
 4  |                             7,3
    |                           /
 3  |                       5,2
    |                     /
 2  |                 3,1         7,5
    |               /     \     /     \
 1  |           1,0         5,4         7,6
    |         /     \           \
 0  |     0,0         2,2         5,5
    |         \                       \
-1  |           0,1         4,5         5,6
    |               \     /     \
-2  |                 2,4         4,6
    |                     \
-3  |                       3,6
```

> 这里做了一轮映射，把区分不同状态的二元组`[x, y]`映射为了`[k,d]`，其中d为前进深度（步数），k为x，y两者之差
{: .prompt-tip }

The number along the **horizontal axis**, ***d***, is the **depth we’ve reached in the graph**, i.e. how many moves we’ve made so far, remembering that diagonal moves are free. The number along the **vertical axis** we call ***k*,** and notice that for every move in the graph, ***k* = *x* − *y*** for each move on that row.

Moving rightward increases *x*, and so increases *k* by 1. Moving downward increases *y* and so decreases *k* by 1. Moving diagonally increases both *x* and *y*, and so it keeps *k* the same. So, each time we make a rightward or downward move followed by a chain of diagonals, we either increment or decrement *k* by 1. <mark>What we are recording is the furthest through the edit graph we can reach for each value of k, at each step.</mark>

## **Modeling Simplification**

1. don’t need to store *y* since it can be calculated from the values of *k* and *x*.
2. don’t need to store the direction of the move taken at each step, since it can backtrack to find which single path out of the many we’ve explored.
3. *x* values in the $$d_{th}$$ round depend only on those in the $$(d-1)_{th}$$ round, and because each round alternately modifies either the odd or the even *k* positions, each round does not modify the values it depends on from the previous round. Therefore the *x* values can be stored in a single flat array.

> 这里的优化主要是空间上的，关键在于前面对状态描述的二元组变换，也就是 ***k*** 这个变量。因为无论行进路线如何，对于每一个转移， ***k*** 的值只有[+1, -1]两种。进而利用这种奇偶性来压缩空间复杂度，**即一元数组来储存 *k*, *d* 双重遍历过程中的x坐标值**。
{: .prompt-tip }

```
    |      0     1     2     3     4     5
----+--------------------------------------
 4  |                              7
    |
 3  |                        5
    |
 2  |                  3           7
    |
 1  |            1           5           7
    |
 0  |      0           2           5
    |
-1  |            0           4           5
    |
-2  |                  2           4
    |
-3  |                        3
```

```
      k |   -3    -2    -1     0     1     2     3     4
--------+-----------------------------------------------
  d = 0 |                      0
        |
  d = 1 |                0     0     1
        |
  d = 2 |          2     0     2     1     3
        |
  d = 3 |    3     2     4     2     5     3     5
        |
  d = 4 |    3     4     4     5     5     7     5     7
        |
  d = 5 |    3     4     5     5     7     7     5     7
```

## **Minimal Version Working Code**

### Arguments

we need to find the shortest edit path

- `n`: size of String a.
- `m`: size of String b.
- `max`: the most number of moves we might need to make.

```ruby
def shortest_edit
    n, m = @a.size, @b.size
    max = n + m
```

### Initialization

- `v[]`: set up an array to store the latest value of `x` for each `k`.
- `k`: valued in [-max, +max] mapping to [0, 2*max+1].
- **initialization**: `v[1] =  0`
  - so that the iteration for *d* = 0 picks *x* = 0. We need to treat the *d* = 0 iteration just the same as the later iterations **since we might be allowed to move diagonally immediately.** Setting `v[1] = 0` makes the algorithm behave as though it begins with a virtual move downwards from (*x*, *y*) = (0,−1).

```ruby
v = Array.new(2 * max + 1)
v[1] = 0
```

### Loop

- choosing whether to move downward or rightward from the previous round.
  - If `k` is `-d`, or if it’s not `d` and the `k + 1` value is greater than the `k - 1` value, then we move downward.
  - Otherwise we move rightward and take `x` as **one greater** than the previous `k - 1` value. We calculate `y` from this chosen `x` value and the current `k`.
- Having taken a single step rightward or downward, **we see if we can take any diagonal steps**.
- Finally, we return the current value of `d` if we’ve reached the bottom-right position, telling the caller the minimum number of edits required to convert from `a` to `b`.

```ruby
(0 .. max).step do |d|
    (-d .. d).step(2) do |k|
    # Move downward or rightward
        # Graph Boundary(Base Condition)
        if k == -d or (k != d and v[k-1] < v[k+1])
            x = v[k + 1]
        else
            x = v[k - 1] + 1
        end
        y = x - k
    # Judge if can take any diagonal steps
    while x < n and y < m and @a[x].text == @b[y].text
        x, y = x + 1, y + 1
    end

    v[k] = x
    return d if x >= n and y > = m
end
```

### Backtrack

Once we’ve reached the end, we can look at the data we’ve recorded in reverse to figure out which single path led to the result(previos node who **have higer x value** can lead to current node in short distance).

These positions are enough to figure out the diagonal moves between each position: we decrement both *x* and *y* until one of them is equal to the values in the previous position, and then we end up a single downward or rightward move away from that position.

Rather than just recording the latest *x* value for each *k*, we need to keep a trace of the array before each turn of the outer loop.

We add the variable `trace` to store these snapshots of `v`, and push a copy of `v` into it on each loop. Rather than just returning the number of moves required, we now return this trace to the caller.

```ruby
v = Arrray.new(2 * max + 1)
v[1] = 0
trace = 0

(0 .. max).step dp |d|
    trace << v.clone
    (-d .. d).step(2) do |k|
    # calculate the next move...
    return trace if x >= n and y >= m
```

## **Space Complexity Improve**

1. $$O(N+M)$$
- Using only a single array of size proportional to N + M, the sum.
- of the lengths of the two strings to store just the length

2. $$O(N+M)^2$$
- to store the edit path, the worst case it will take N+M moves to traverse the graph.
- because the tarverse walks from the top-left to the bottom-right of the graph in a single pass.
  
### Improvement: Linear Space Version

- use **divide and conquer** approach.
- The edit sequence is made up of a series of what Myers calls *snakes*.
- finding the middle snake of a possible edit path, using the endpoints of that to divide the original graph into two smaller regions.
- it then works recursively on these regions until they're so small that no further work is required.
