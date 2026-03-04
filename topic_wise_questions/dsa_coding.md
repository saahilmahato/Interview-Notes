# 🧠 Elite Software Engineer Interview: Data Structures & Algorithms

---

## SECTION 1: Arrays & Strings

---

**Q1. Given an array of integers, find the maximum product subarray.**

**Sample Answer:**

The naive approach is O(n²) — try every subarray. But we can do it in O(n) time, O(1) space by tracking both the maximum and minimum product ending at each position. The minimum matters because two negatives multiply to a positive.

```python
def max_product_subarray(nums):
    max_prod = min_prod = result = nums[0]
    
    for n in nums[1:]:
        candidates = (n, max_prod * n, min_prod * n)
        max_prod = max(candidates)
        min_prod = min(candidates)
        result = max(result, max_prod)
    
    return result
```

Key insight: at each index, the max product is either the element itself (starting fresh), or extending the previous max or min product. We carry the min because `min * negative = large positive`.

Edge cases: all negatives (e.g., `[-2, -3, -4]` → answer is 12), single element, zeros (zeros reset the running product, which the "start fresh" branch handles naturally).

---

**Q2. How would you find all triplets in an array that sum to zero? What's the optimal complexity?**

**Sample Answer:**

The optimal solution is O(n²) time, O(1) extra space (ignoring output).

```python
def three_sum(nums):
    nums.sort()  # O(n log n)
    result = []
    
    for i in range(len(nums) - 2):
        if i > 0 and nums[i] == nums[i-1]:  # skip duplicates
            continue
        
        left, right = i + 1, len(nums) - 1
        while left < right:
            s = nums[i] + nums[left] + nums[right]
            if s == 0:
                result.append([nums[i], nums[left], nums[right]])
                while left < right and nums[left] == nums[left+1]: left += 1
                while left < right and nums[right] == nums[right-1]: right -= 1
                left += 1
                right -= 1
            elif s < 0:
                left += 1
            else:
                right -= 1
    
    return result
```

We sort first, then fix one element and use the two-pointer technique on the rest. The duplicate-skipping logic is critical — interviewers love to see that handled correctly. You cannot do better than O(n²) in the worst case because the output itself can be O(n²) triplets.

---

**Q3. Explain the sliding window technique and when to apply it.**

**Sample Answer:**

Sliding window is applicable when you need to find a contiguous subarray or substring that satisfies some condition — maximum sum, shortest/longest window, contains certain characters, etc.

There are two flavors:

**Fixed-size window:** move a window of size k across the array.
```python
def max_sum_k(arr, k):
    window_sum = sum(arr[:k])
    max_sum = window_sum
    for i in range(k, len(arr)):
        window_sum += arr[i] - arr[i - k]
        max_sum = max(max_sum, window_sum)
    return max_sum
```

**Variable-size window:** expand the right pointer, shrink the left pointer when a condition is violated.
```python
def longest_substring_no_repeat(s):
    char_set = set()
    left = max_len = 0
    for right in range(len(s)):
        while s[right] in char_set:
            char_set.remove(s[left])
            left += 1
        char_set.add(s[right])
        max_len = max(max_len, right - left + 1)
    return max_len
```

The key invariant is: the window always satisfies the constraint. When it doesn't, shrink from the left. This converts an O(n²) brute force into O(n).

A more advanced variant: "minimum window substring" where you track character frequencies with a hashmap and a `formed` counter to know when the window is valid.

---

**Q4. How do you find the median of a data stream in O(log n) per insertion and O(1) per query?**

**Sample Answer:**

Use two heaps: a max-heap for the lower half and a min-heap for the upper half. Maintain the invariant that their sizes differ by at most 1.

```python
import heapq

class MedianFinder:
    def __init__(self):
        self.low = []   # max-heap (negate values)
        self.high = []  # min-heap
    
    def add_num(self, num):
        heapq.heappush(self.low, -num)
        
        # Ensure max of low <= min of high
        if self.low and self.high and (-self.low[0] > self.high[0]):
            heapq.heappush(self.high, -heapq.heappop(self.low))
        
        # Balance sizes
        if len(self.low) > len(self.high) + 1:
            heapq.heappush(self.high, -heapq.heappop(self.low))
        elif len(self.high) > len(self.low):
            heapq.heappush(self.low, -heapq.heappop(self.high))
    
    def find_median(self):
        if len(self.low) > len(self.high):
            return -self.low[0]
        return (-self.low[0] + self.high[0]) / 2
```

Python only has min-heaps natively, so we negate values to simulate a max-heap. This is a classic design problem that tests knowledge of heap invariants and lazy balancing.

---

## SECTION 2: Linked Lists

---

**Q5. Detect and find the start of a cycle in a linked list.**

**Sample Answer:**

Floyd's Tortoise and Hare algorithm, in two phases:

**Phase 1 — Detect:** slow moves 1 step, fast moves 2 steps. If they meet, there's a cycle.

**Phase 2 — Find start:** reset slow to head, keep fast at meeting point. Both move 1 step at a time. Where they meet is the cycle start.

```python
def detect_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow == fast:
            # Cycle detected
            slow = head
            while slow != fast:
                slow = slow.next
                fast = fast.next
            return slow  # cycle start
    return None
```

**Why does Phase 2 work?** Let F = distance from head to cycle start, C = cycle length, k = distance from cycle start to meeting point. When they meet: the slow pointer has traveled F + k. The fast pointer has traveled F + k + mC for some integer m. Since fast travels twice as far: 2(F+k) = F+k+mC → F+k = mC → F = mC - k. So if slow resets to head and both move at speed 1, after F steps, slow is at cycle start and fast (which traveled mC - k more steps from meeting point) is also at cycle start.

---

**Q6. Reverse a linked list in groups of k.**

**Sample Answer:**

```python
def reverse_k_group(head, k):
    # Check if k nodes remain
    count, node = 0, head
    while node and count < k:
        node = node.next
        count += 1
    
    if count < k:
        return head  # don't reverse partial group
    
    # Reverse k nodes
    prev, curr = None, head
    for _ in range(k):
        nxt = curr.next
        curr.next = prev
        prev = curr
        curr = nxt
    
    # head is now the tail of reversed group
    head.next = reverse_k_group(curr, k)
    return prev
```

Time: O(n), Space: O(n/k) recursion stack. For iterative, maintain a tail pointer of each reversed segment and connect it to the next reversed segment.

The tricky part is the "don't reverse if fewer than k nodes remain" condition — this requires lookahead before committing to reversing.

---

**Q7. Merge k sorted linked lists efficiently.**

**Sample Answer:**

The optimal approach is a min-heap.

```python
import heapq

def merge_k_lists(lists):
    heap = []
    for i, node in enumerate(lists):
        if node:
            heapq.heappush(heap, (node.val, i, node))
    
    dummy = curr = ListNode(0)
    while heap:
        val, i, node = heapq.heappop(heap)
        curr.next = node
        curr = curr.next
        if node.next:
            heapq.heappush(heap, (node.next.val, i, node.next))
    
    return dummy.next
```

Time: O(N log k) where N = total nodes, k = number of lists. The heap always has at most k elements.

The `i` in the tuple breaks ties when values are equal (Python compares tuple elements left to right, and comparing ListNodes would fail).

Alternative: divide and conquer — merge lists in pairs, then pairs of pairs. Also O(N log k) but higher constant. The heap is generally preferred in practice.

---

## SECTION 3: Trees & Graphs

---

**Q8. Serialize and deserialize a binary tree.**

**Sample Answer:**

```python
class Codec:
    def serialize(self, root):
        result = []
        def dfs(node):
            if not node:
                result.append('#')
                return
            result.append(str(node.val))
            dfs(node.left)
            dfs(node.right)
        dfs(root)
        return ','.join(result)
    
    def deserialize(self, data):
        tokens = iter(data.split(','))
        def dfs():
            val = next(tokens)
            if val == '#':
                return None
            node = TreeNode(int(val))
            node.left = dfs()
            node.right = dfs()
            return node
        return dfs()
```

I use preorder DFS with `#` as null markers. The key insight is that preorder serialization (root, left, right) with null markers is sufficient to uniquely reconstruct the tree — unlike inorder alone which is ambiguous.

Using `iter()` and `next()` is elegant because it maintains shared state across recursive calls without an index variable.

BFS (level-order) serialization is also valid and more natural for humans to read, but the recursive DFS solution is cleaner code.

---

**Q9. Find the lowest common ancestor (LCA) of two nodes in a binary tree (not a BST).**

**Sample Answer:**

```python
def lowest_common_ancestor(root, p, q):
    if not root or root == p or root == q:
        return root
    
    left = lowest_common_ancestor(root.left, p, q)
    right = lowest_common_ancestor(root.right, p, q)
    
    if left and right:
        return root  # p and q are in different subtrees
    return left or right  # both in same subtree
```

The logic: recurse down both subtrees. If a node finds either p or q, it returns it. If both subtrees return non-null, the current node is the LCA. If only one side returns non-null, propagate it upward.

For a BST, you can exploit ordering: if both p and q are less than root, go left; if both greater, go right; otherwise root is LCA. O(h) time vs O(n) for general tree.

For the follow-up where LCA is needed repeatedly (offline queries), Tarjan's offline LCA algorithm using Union-Find gives O(α(n)) per query after O(n) preprocessing.

---

**Q10. Implement Dijkstra's algorithm and explain why it doesn't work with negative edges.**

**Sample Answer:**

```python
import heapq
from collections import defaultdict

def dijkstra(graph, src, n):
    dist = [float('inf')] * n
    dist[src] = 0
    heap = [(0, src)]  # (distance, node)
    
    while heap:
        d, u = heapq.heappop(heap)
        
        if d > dist[u]:  # stale entry
            continue
        
        for v, w in graph[u]:
            if dist[u] + w < dist[v]:
                dist[v] = dist[u] + w
                heapq.heappush(heap, (dist[v], v))
    
    return dist
```

Time: O((V + E) log V) with a binary heap.

**Why negative edges break it:** Dijkstra's correctness relies on the greedy invariant — once a node is popped from the heap, its distance is finalized. With negative edges, a later path could improve an already-finalized node's distance, violating this invariant.

Example: A→B (weight 1), A→C (weight 3), C→B (weight -3). Dijkstra finalizes B at distance 1, but the true shortest path A→C→B has distance 0.

**Fix:** Use Bellman-Ford for graphs with negative edges (O(VE)), which relaxes all edges V-1 times and can detect negative cycles. For dense graphs with negative weights, Johnson's algorithm reweights edges to make them non-negative, then runs Dijkstra from every vertex.

---

**Q11. Given a graph, find all strongly connected components (SCCs).**

**Sample Answer:**

Kosaraju's algorithm — two DFS passes:

```python
def kosaraju(graph, n):
    visited = [False] * n
    finish_order = []
    
    def dfs1(u):
        visited[u] = True
        for v in graph[u]:
            if not visited[v]:
                dfs1(v)
        finish_order.append(u)
    
    for i in range(n):
        if not visited[i]:
            dfs1(i)
    
    # Build reversed graph
    rev = [[] for _ in range(n)]
    for u in range(n):
        for v in graph[u]:
            rev[v].append(u)
    
    visited = [False] * n
    sccs = []
    
    def dfs2(u, scc):
        visited[u] = True
        scc.append(u)
        for v in rev[u]:
            if not visited[v]:
                dfs2(v, scc)
    
    for u in reversed(finish_order):
        if not visited[u]:
            scc = []
            dfs2(u, scc)
            sccs.append(scc)
    
    return sccs
```

**Why it works:** The first DFS determines finish times. The node that finishes last in DFS1 is a root of an SCC. On the reversed graph, that root can only reach its own SCC (because the cross-SCC edges are now reversed). So DFS2 from the highest-finish-time node collects exactly one SCC.

Alternative: Tarjan's algorithm does it in a single DFS pass using a stack and low-link values, which is more cache-friendly and elegant, though harder to implement correctly.

---

**Q12. How would you detect a cycle in a directed graph vs. an undirected graph?**

**Sample Answer:**

**Undirected graph:** DFS, track parent. If you visit a neighbor that's already visited and isn't the parent, there's a cycle. Union-Find is another elegant approach — union edges as you process them; if two endpoints are already in the same component, adding that edge creates a cycle.

**Directed graph:** DFS with three-color marking:
- White (0): unvisited
- Gray (1): in current DFS path (recursion stack)
- Black (2): fully processed

A back edge (gray → gray) means a cycle.

```python
def has_cycle_directed(graph, n):
    color = [0] * n
    
    def dfs(u):
        color[u] = 1  # gray
        for v in graph[u]:
            if color[v] == 1:
                return True  # back edge → cycle
            if color[v] == 0 and dfs(v):
                return True
        color[u] = 2  # black
        return False
    
    return any(dfs(i) for i in range(n) if color[i] == 0)
```

The key distinction: in undirected graphs, any visited non-parent neighbor is a cycle. In directed graphs, only a neighbor on the *current path* (gray) constitutes a cycle — a cross edge to an already-finished node (black) is not a cycle.

Topological sort via Kahn's algorithm (BFS-based) is another way: if after processing, not all nodes are in the topological order, a cycle exists.

---

## SECTION 4: Dynamic Programming

---

**Q13. Explain the difference between memoization and tabulation. When would you prefer each?**

**Sample Answer:**

Both are DP techniques to avoid recomputing subproblems.

**Memoization (top-down):** Start with the recursive solution, add a cache.
```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n):
    if n <= 1: return n
    return fib(n-1) + fib(n-2)
```

**Tabulation (bottom-up):** Fill a table iteratively starting from base cases.
```python
def fib(n):
    dp = [0, 1]
    for i in range(2, n+1):
        dp.append(dp[-1] + dp[-2])
    return dp[n]
    # Optimized: just track last two values → O(1) space
```

**When to prefer memoization:**
- The subproblem space is sparse (not all subproblems need to be solved)
- The recurrence is complex and hard to express as a loop order
- You want to convert a recursive solution quickly

**When to prefer tabulation:**
- You need to optimize space (rolling array technique)
- You want to avoid Python recursion depth limits
- Performance matters (no function call overhead, better cache locality)
- You need the full DP table for reconstruction

In interviews, memoization is usually faster to write correctly. In production, tabulation is usually faster to execute.

---

**Q14. Solve the 0/1 Knapsack problem and analyze its complexity. Is it truly polynomial?**

**Sample Answer:**

```python
def knapsack(weights, values, capacity):
    n = len(weights)
    # dp[i][w] = max value using first i items, capacity w
    dp = [[0] * (capacity + 1) for _ in range(n + 1)]
    
    for i in range(1, n + 1):
        for w in range(capacity + 1):
            dp[i][w] = dp[i-1][w]  # skip item i
            if weights[i-1] <= w:
                dp[i][w] = max(dp[i][w], 
                               dp[i-1][w - weights[i-1]] + values[i-1])
    
    # Reconstruct solution
    items = []
    w = capacity
    for i in range(n, 0, -1):
        if dp[i][w] != dp[i-1][w]:
            items.append(i-1)
            w -= weights[i-1]
    
    return dp[n][capacity], items
```

Time: O(nW), Space: O(nW), reducible to O(W) with a 1D array (iterate w backwards).

**Is it polynomial?** No — it's *pseudo-polynomial*. The complexity O(nW) depends on the *value* of W, not the number of bits needed to represent W. If W is represented in b bits, W = 2^b, making the true complexity O(n · 2^b) — exponential in the input size. 0/1 Knapsack is NP-complete. This is a subtle but critical distinction that separates strong candidates.

---

**Q15. Solve the Edit Distance (Levenshtein Distance) problem.**

**Sample Answer:**

```python
def edit_distance(word1, word2):
    m, n = len(word1), len(word2)
    # dp[i][j] = edit distance between word1[:i] and word2[:j]
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    
    for i in range(m + 1): dp[i][0] = i
    for j in range(n + 1): dp[0][j] = j
    
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if word1[i-1] == word2[j-1]:
                dp[i][j] = dp[i-1][j-1]
            else:
                dp[i][j] = 1 + min(
                    dp[i-1][j],    # delete from word1
                    dp[i][j-1],    # insert into word1
                    dp[i-1][j-1]   # replace
                )
    
    return dp[m][n]
```

Space optimization: since each row only depends on the previous row, we can use two 1D arrays — or even one array with careful ordering.

Applications: spell checkers, DNA sequence alignment (Smith-Waterman), plagiarism detection, version control diff algorithms.

The recurrence is elegant: if characters match, cost is 0 and we take the diagonal. Otherwise, we take the minimum cost among the three operations plus 1.

---

**Q16. What is the longest increasing subsequence (LIS) and what are its optimal solutions?**

**Sample Answer:**

**O(n²) DP:**
```python
def lis_n2(nums):
    n = len(nums)
    dp = [1] * n
    for i in range(1, n):
        for j in range(i):
            if nums[j] < nums[i]:
                dp[i] = max(dp[i], dp[j] + 1)
    return max(dp)
```

**O(n log n) — Patience Sorting:**
```python
import bisect

def lis_nlogn(nums):
    tails = []  # tails[i] = smallest tail of all IS of length i+1
    for num in nums:
        pos = bisect.bisect_left(tails, num)
        if pos == len(tails):
            tails.append(num)
        else:
            tails[pos] = num
    return len(tails)
```

**Why does patience sorting work?** `tails` is always sorted. For each number, binary search for the leftmost tail it can replace (or extend). `tails` doesn't store an actual subsequence — it stores the optimal "frontier." The length of `tails` is always the LIS length.

To reconstruct the actual subsequence, maintain a `parent` array and track which index each `tails` entry corresponds to.

**Follow-up — LIS variants:**
- Longest non-decreasing subsequence: use `bisect_right`
- Number of LIS: requires augmenting the O(n²) DP with a count array
- 2D LIS (Russian Doll Envelopes): sort by width ascending, then find LIS on height

---

**Q17. Explain interval DP with an example.**

**Sample Answer:**

Interval DP solves problems over contiguous subarrays/substrings, where the answer for an interval `[i, j]` depends on answers for smaller intervals.

**Classic example: Burst Balloons**

Given balloons with values, bursting balloon i gives coins `nums[i-1] * nums[i] * nums[i+1]`. Maximize total coins.

The trick: instead of thinking about which balloon to burst first, think about which to burst *last* within interval `[i, j]`.

```python
def max_coins(nums):
    nums = [1] + nums + [1]  # add boundary balloons
    n = len(nums)
    dp = [[0] * n for _ in range(n)]
    
    for length in range(2, n):  # interval length
        for left in range(0, n - length):
            right = left + length
            for k in range(left + 1, right):  # last balloon to burst
                dp[left][right] = max(
                    dp[left][right],
                    dp[left][k] + nums[left] * nums[k] * nums[right] + dp[k][right]
                )
    
    return dp[0][n-1]
```

Other interval DP problems: Matrix Chain Multiplication, Palindrome Partitioning, Optimal Binary Search Tree. The common pattern is: enumerate all intervals by length (smallest first), then try all split points within the interval.

---

## SECTION 5: Heaps, Stacks, Queues

---

**Q18. Design a stack that supports push, pop, top, and retrieving the minimum element in O(1).**

**Sample Answer:**

```python
class MinStack:
    def __init__(self):
        self.stack = []
        self.min_stack = []
    
    def push(self, val):
        self.stack.append(val)
        if not self.min_stack or val <= self.min_stack[-1]:
            self.min_stack.append(val)
    
    def pop(self):
        val = self.stack.pop()
        if val == self.min_stack[-1]:
            self.min_stack.pop()
    
    def top(self):
        return self.stack[-1]
    
    def get_min(self):
        return self.min_stack[-1]
```

The min_stack tracks minimums lazily — only push to it when the new value is ≤ current minimum, and pop from it only when the main stack pops that exact minimum value.

Space: O(n) worst case (all pushes are decreasing). An optimization: store `(value, min_at_this_point)` tuples in one stack, eliminating the second stack at the cost of doubled memory per entry.

---

**Q19. Explain monotonic stacks — when and why to use them.**

**Sample Answer:**

A monotonic stack maintains elements in strictly increasing or decreasing order. When a new element violates the monotonicity, we pop elements (and process them) until the invariant is restored.

**Classic use case: Next Greater Element**
```python
def next_greater(nums):
    result = [-1] * len(nums)
    stack = []  # stores indices
    
    for i, num in enumerate(nums):
        while stack and nums[stack[-1]] < num:
            idx = stack.pop()
            result[idx] = num
        stack.append(i)
    
    return result
```

**Harder use case: Largest Rectangle in Histogram**
```python
def largest_rectangle(heights):
    stack = []  # monotonic increasing stack of indices
    max_area = 0
    heights.append(0)  # sentinel
    
    for i, h in enumerate(heights):
        start = i
        while stack and heights[stack[-1]] > h:
            idx = stack.pop()
            width = i - (stack[-1] + 1 if stack else 0)
            max_area = max(max_area, heights[idx] * width)
            start = idx
        stack.append(start)
    
    return max_area
```

Key insight: when we pop a bar, we know its "right boundary" (current index) and "left boundary" (new stack top + 1). This gives us the maximum width for that bar height.

Monotonic stacks are also used in: Trapping Rain Water, Daily Temperatures, Sum of Subarray Minimums, Stock Span Problem. The pattern is always: "for each element, find the nearest element (to left/right) that is greater/smaller."

---

**Q20. How does a priority queue work internally, and what are the time complexities of its operations?**

**Sample Answer:**

A priority queue is typically implemented as a binary heap — a complete binary tree stored in an array where `parent = (i-1)//2`, `left = 2i+1`, `right = 2i+2`.

| Operation | Binary Heap | Fibonacci Heap |
|-----------|-------------|----------------|
| Insert | O(log n) | O(1) amortized |
| Find-min | O(1) | O(1) |
| Extract-min | O(log n) | O(log n) amortized |
| Decrease-key | O(log n) | O(1) amortized |
| Merge | O(n) | O(1) |

**Heapify (build heap from array):** O(n) — not O(n log n). The trick is that most nodes are near the leaves and require very few swaps. Formally: sum of heights = O(n).

**Why Fibonacci Heap matters:** Dijkstra's algorithm with a binary heap is O((V+E) log V). With a Fibonacci heap, it's O(E + V log V) — better for dense graphs. In practice, Fibonacci heaps have high constants and are rarely used.

**Heap vs BST for priority queue:** Heap gives O(1) find-min but O(n) search. BST gives O(log n) for everything including search and arbitrary deletion. Use a heap when you only need min/max; use a BST (like Python's `sortedcontainers.SortedList`) when you need ordered traversal or arbitrary element removal.

---

## SECTION 6: Hashing & Search

---

**Q21. How would you design a hash map from scratch?**

**Sample Answer:**

```python
class HashMap:
    def __init__(self, capacity=16, load_factor=0.75):
        self.capacity = capacity
        self.load_factor = load_factor
        self.size = 0
        self.buckets = [[] for _ in range(capacity)]
    
    def _hash(self, key):
        return hash(key) % self.capacity
    
    def put(self, key, value):
        idx = self._hash(key)
        for i, (k, v) in enumerate(self.buckets[idx]):
            if k == key:
                self.buckets[idx][i] = (key, value)
                return
        self.buckets[idx].append((key, value))
        self.size += 1
        if self.size / self.capacity > self.load_factor:
            self._resize()
    
    def get(self, key):
        idx = self._hash(key)
        for k, v in self.buckets[idx]:
            if k == key:
                return v
        return None
    
    def _resize(self):
        old_buckets = self.buckets
        self.capacity *= 2
        self.buckets = [[] for _ in range(self.capacity)]
        self.size = 0
        for bucket in old_buckets:
            for k, v in bucket:
                self.put(k, v)
```

**Key design decisions:**
- **Collision resolution:** chaining (above) vs open addressing (linear probing, quadratic probing, double hashing). Open addressing has better cache performance; chaining is simpler and handles high load factors gracefully.
- **Load factor:** 0.75 is Java's HashMap default — balances space and time. Above this, expected chain length grows superlinearly.
- **Hash function:** Must be deterministic, uniformly distribute keys, and be fast. Cryptographic hashes are too slow. MurmurHash or xxHash are common in practice.
- **Resize:** Double capacity and rehash everything — amortized O(1) per insert.

**Worst case vs amortized:** Worst case for put/get is O(n) (all keys collide). With a good hash function, expected O(1). Java 8+ converts chains to red-black trees when length > 8, giving O(log n) worst case.

---

**Q22. Explain binary search and all the ways it can go wrong.**

**Sample Answer:**

```python
def binary_search(arr, target):
    left, right = 0, len(arr) - 1
    while left <= right:
        mid = left + (right - left) // 2  # avoid integer overflow
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    return -1
```

**Common mistakes:**

1. **Integer overflow:** `mid = (left + right) // 2` overflows in languages with fixed-size integers when left + right > INT_MAX. Use `left + (right - left) // 2`.

2. **Infinite loop:** `right = mid` instead of `right = mid - 1` when using `left < right` loop condition.

3. **Off-by-one:** Using `left < right` vs `left <= right` — which is correct depends on whether `right` is inclusive or exclusive.

4. **Wrong answer:** Finding *a* occurrence vs *first* occurrence vs *last* occurrence.

**Finding leftmost occurrence:**
```python
def lower_bound(arr, target):
    left, right = 0, len(arr)
    while left < right:
        mid = left + (right - left) // 2
        if arr[mid] < target:
            left = mid + 1
        else:
            right = mid
    return left
```

**Binary search on answer space:** Many problems reduce to "find the minimum X such that condition(X) is true." If condition is monotonic, binary search over the answer.
```python
# Minimum capacity to ship packages in D days
def ship_capacity(weights, D):
    left, right = max(weights), sum(weights)
    while left < right:
        mid = (left + right) // 2
        if can_ship(weights, D, mid):
            right = mid
        else:
            left = mid + 1
    return left
```

---

## SECTION 7: Tries & Advanced Structures

---

**Q23. Implement a Trie and discuss its advantages over a HashMap for string operations.**

**Sample Answer:**

```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False
        self.count = 0  # words passing through this node

class Trie:
    def __init__(self):
        self.root = TrieNode()
    
    def insert(self, word):
        node = self.root
        for ch in word:
            if ch not in node.children:
                node.children[ch] = TrieNode()
            node = node.children[ch]
            node.count += 1
        node.is_end = True
    
    def search(self, word):
        node = self._find(word)
        return node is not None and node.is_end
    
    def starts_with(self, prefix):
        return self._find(prefix) is not None
    
    def count_prefix(self, prefix):
        node = self._find(prefix)
        return node.count if node else 0
    
    def _find(self, prefix):
        node = self.root
        for ch in prefix:
            if ch not in node.children:
                return None
            node = node.children[ch]
        return node
```

**Trie vs HashMap:**
- HashMap lookup: O(m) per key where m = word length (hashing the string is O(m))
- Trie lookup: O(m) per key — same asymptotically
- **Trie wins for:** prefix queries (starts_with is O(p) where p = prefix length), autocomplete, sorted enumeration of all keys, longest common prefix
- **HashMap wins for:** simpler implementation, lower constant factors, arbitrary key types

**Advanced variants:**
- **Compressed Trie (Patricia/Radix Tree):** Merge single-child nodes — better space for sparse tries
- **Ternary Search Tree:** BST-based trie, better cache performance
- **Aho-Corasick:** Trie + failure links, multi-pattern string matching in O(n + m + z) where z = matches

---

## SECTION 8: Sorting & Complexity

---

**Q24. Compare all major sorting algorithms — when would you use each in practice?**

**Sample Answer:**

| Algorithm | Best | Average | Worst | Space | Stable |
|-----------|------|---------|-------|-------|--------|
| Quicksort | O(n log n) | O(n log n) | O(n²) | O(log n) | No |
| Mergesort | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes |
| Heapsort | O(n log n) | O(n log n) | O(n log n) | O(1) | No |
| Timsort | O(n) | O(n log n) | O(n log n) | O(n) | Yes |
| Counting Sort | O(n+k) | O(n+k) | O(n+k) | O(k) | Yes |
| Radix Sort | O(nk) | O(nk) | O(nk) | O(n+k) | Yes |

**In practice:**
- **Quicksort:** Cache-friendly, low constants. Used in C's `qsort`. Use 3-way partition for many duplicates. Random pivot to avoid O(n²) worst case.
- **Mergesort:** Preferred for linked lists (no random access needed). Used when stability is required and extra memory is acceptable.
- **Timsort:** Python and Java's default. Exploits natural runs in real-world data. Hybrid of merge + insertion sort.
- **Heapsort:** When guaranteed O(n log n) in-place is required (no extra memory, no worst case). Rarely used in practice due to cache inefficiency.
- **Counting/Radix:** When key range is bounded. Sorting integers in [0, 10^6] — counting sort is O(n). Sorting strings by length then character — radix sort.

**Introsort:** Quicksort that falls back to Heapsort when recursion depth exceeds log n — gets the best of both worlds. Used in C++ STL `std::sort`.

---

**Q25. Prove that comparison-based sorting cannot do better than O(n log n).**

**Sample Answer:**

The proof uses the decision tree model.

Any comparison-based sort can be modeled as a binary decision tree where each internal node is a comparison (a[i] < a[j]?), and each leaf is a permutation of the input.

For n elements, there are n! possible orderings. A binary tree with L leaves has height ≥ log₂(L). Therefore, the decision tree must have height ≥ log₂(n!).

By Stirling's approximation:
```
log₂(n!) = log₂(1 · 2 · 3 · ... · n)
          ≥ log₂((n/2)^(n/2))    [taking only the top half terms]
          = (n/2) · log₂(n/2)
          = Ω(n log n)
```

Therefore, any comparison-based sorting algorithm must make Ω(n log n) comparisons in the worst case.

This is why counting sort, radix sort, and bucket sort can beat this bound — they don't use comparisons, they exploit structure in the keys (e.g., integer values, bounded range).

---

## SECTION 9: Design & System-Level DSA

---

**Q26. Design an LRU Cache with O(1) get and put.**

**Sample Answer:**

Combine a HashMap and a Doubly Linked List.

```python
class LRUCache:
    class Node:
        def __init__(self, key=0, val=0):
            self.key, self.val = key, val
            self.prev = self.next = None
    
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}  # key → node
        # Sentinel head and tail
        self.head, self.tail = self.Node(), self.Node()
        self.head.next = self.tail
        self.tail.prev = self.head
    
    def _remove(self, node):
        node.prev.next = node.next
        node.next.prev = node.prev
    
    def _insert_front(self, node):
        node.next = self.head.next
        node.prev = self.head
        self.head.next.prev = node
        self.head.next = node
    
    def get(self, key):
        if key not in self.cache:
            return -1
        node = self.cache[key]
        self._remove(node)
        self._insert_front(node)
        return node.val
    
    def put(self, key, value):
        if key in self.cache:
            self._remove(self.cache[key])
        node = self.Node(key, value)
        self._insert_front(node)
        self.cache[key] = node
        if len(self.cache) > self.capacity:
            lru = self.tail.prev
            self._remove(lru)
            del self.cache[lru.key]
```

The HashMap gives O(1) access to any node. The doubly linked list gives O(1) removal and insertion. Sentinels eliminate all the edge cases of empty list or single element.

**Follow-up — LFU Cache:** Uses two HashMaps (key→(value,freq) and freq→OrderedDict of keys) and tracks minimum frequency. O(1) for all operations.

---

**Q27. How would you find the kth largest element in an unseen stream of integers with limited memory?**

**Sample Answer:**

Maintain a min-heap of size k.

```python
import heapq

class KthLargest:
    def __init__(self, k):
        self.k = k
        self.heap = []  # min-heap of size k
    
    def add(self, val):
        heapq.heappush(self.heap, val)
        if len(self.heap) > self.k:
            heapq.heappop(self.heap)
        return self.heap[0]  # kth largest = min of top-k
```

Space: O(k). Each add: O(log k). The minimum of the k largest elements seen so far is always the kth largest.

**If you need the exact kth largest from a static array:** Use Quickselect — O(n) average, O(n²) worst (mitigated with random pivot). Median-of-medians gives O(n) worst case but with a high constant.

```python
import random

def quickselect(nums, k):
    # Find kth largest = (n-k)th smallest
    target = len(nums) - k
    
    def select(left, right):
        pivot_idx = random.randint(left, right)
        nums[pivot_idx], nums[right] = nums[right], nums[pivot_idx]
        pivot = nums[right]
        store = left
        for i in range(left, right):
            if nums[i] <= pivot:
                nums[store], nums[i] = nums[i], nums[store]
                store += 1
        nums[store], nums[right] = nums[right], nums[store]
        if store == target: return nums[store]
        elif store < target: return select(store+1, right)
        else: return select(left, store-1)
    
    return select(0, len(nums)-1)
```

---

## SECTION 10: Conceptual & Big-Picture

---

**Q28. What is amortized analysis? Explain with an example.**

**Sample Answer:**

Amortized analysis gives the average cost per operation over a sequence of operations, even if some individual operations are expensive.

**Dynamic array (append):** Most appends are O(1). Occasionally, when the array is full, we resize — copy all n elements, O(n). But we double the capacity, so the next resize won't happen until n more inserts.

Charge each element $2: $1 for its own insertion, $1 saved to pay for its future copy during resize. By the time we resize to 2n, we have n * $1 = n credits to pay for copying n elements. Total cost for n appends = O(n), so amortized O(1) per append.

This is the **accounting/banker's method**. Other methods:
- **Aggregate analysis:** Total cost of n operations / n
- **Potential method:** Define Φ(state), amortized cost = actual cost + ΔΦ

**Why it matters:** Without amortized analysis, you'd incorrectly say dynamic arrays are O(n) per append in the worst case and avoid them. Amortized analysis proves they're just as good as arrays in practice.

---

**Q29. Explain P vs NP in practical terms. Give examples of NP-complete problems and how we deal with them in practice.**

**Sample Answer:**

- **P:** Problems solvable in polynomial time (O(n^k) for some k). Sorting, shortest path, matrix multiplication.
- **NP:** Problems where a proposed solution can be *verified* in polynomial time. The question is whether every NP problem can also be *solved* in polynomial time.
- **NP-complete:** The hardest problems in NP. Every other NP problem reduces to them in polynomial time. If you solve one NP-complete problem in poly time, you solve them all.
- **NP-hard:** At least as hard as NP-complete, but not necessarily in NP (e.g., Halting Problem).

**NP-complete examples:** TSP (decision version), 3-SAT, Graph Coloring, Subset Sum, 0/1 Knapsack, Hamiltonian Cycle.

**How we deal with them in practice:**
1. **Approximation algorithms:** TSP with triangle inequality → 2-approximation (Christofides gives 1.5x). Vertex Cover → 2-approximation.
2. **Heuristics:** Greedy, simulated annealing, genetic algorithms. No guarantees but fast and often good enough.
3. **Exponential algorithms with pruning:** Branch and bound, backtracking. Works when n is small or constraints are tight.
4. **Special cases:** Many NP-complete problems have polynomial solutions for restricted inputs (e.g., TSP on trees, graph coloring for planar graphs).
5. **Fixed-parameter tractability (FPT):** Algorithms exponential only in a "parameter" k, polynomial in n. Useful when k is small.

Most real-world NP-complete instances (job scheduling, network routing) are solved this way — not optimally, but satisfactorily.

---

**Q30. Given unlimited time, what is the most beautiful or surprising algorithm you know, and why?**

**Sample Answer (what an elite engineer might say):**

The most surprising to me is **the Fast Fourier Transform (FFT)** — specifically Cooley-Tukey.

Multiplying two n-degree polynomials naively is O(n²). The insight is: evaluate both polynomials at n points (point-value representation), multiply the values pointwise (O(n)), then interpolate back. The magic is that if you choose the evaluation points to be the nth roots of unity, the evaluation and interpolation steps have recursive structure that reduces to O(n log n).

```
DFT(x) — split into even and odd indexed coefficients:
DFT(x) = DFT_even(x²) + x * DFT_odd(x²)
```

This halves the problem size at each step — classic divide and conquer. The "surprising" part is that complex numbers and roots of unity are the key to unlocking a 2× speedup at each recursion level.

Why it's beautiful: it connects number theory (roots of unity), complex analysis, divide and conquer, and has applications in signal processing, polynomial multiplication, large integer multiplication (Schönhage-Strassen), image compression (JPEG), and even quantum computing (Quantum Fourier Transform is a core subroutine in Shor's algorithm for factoring). A purely mathematical algorithm that reshapes multiple industries.

---

*30 questions across 10 DSA domains. An elite candidate doesn't just recite solutions — they articulate **why** the algorithm works, **when** to apply it, **what tradeoffs** it makes, and **where** it appears in real systems.*