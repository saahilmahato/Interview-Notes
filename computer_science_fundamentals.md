# Computer Science Fundamentals — Interview Q&A

---

## 1. DATA STRUCTURES

**Q: What is the difference between an Array and a Linked List?**
A: An array stores elements in contiguous memory locations, allowing O(1) random access by index but O(n) insertion/deletion (due to shifting). A linked list stores elements in nodes with pointers to the next node, allowing O(1) insertion/deletion at known positions but O(n) access by index. Arrays are better for read-heavy workloads; linked lists for frequent insertions/deletions.

---

**Q: What are the types of Linked Lists?**
A:
- **Singly Linked List** – Each node points to the next node.
- **Doubly Linked List** – Each node points to both next and previous nodes.
- **Circular Linked List** – The last node points back to the first node (can be singly or doubly).

---

**Q: What is a Stack? Where is it used?**
A: A stack is a LIFO (Last In, First Out) data structure. Operations: `push` (insert), `pop` (remove top), `peek` (view top). Used in: function call management (call stack), expression evaluation, undo/redo features, backtracking algorithms, and parsing.

---

**Q: What is a Queue? What are its variants?**
A: A queue is a FIFO (First In, First Out) data structure. Variants include:
- **Circular Queue** – The end connects back to the front to reuse space.
- **Deque (Double-Ended Queue)** – Insertion/deletion from both ends.
- **Priority Queue** – Elements are dequeued by priority, not arrival order (typically backed by a heap).

---

**Q: What is a Hash Table? How does it handle collisions?**
A: A hash table maps keys to values using a hash function. Collisions (two keys mapping to the same bucket) are handled via:
- **Chaining** – Each bucket holds a linked list of entries.
- **Open Addressing** – On collision, probe for the next available slot (linear probing, quadratic probing, double hashing).

Average time complexity for insert/search/delete is O(1); worst case O(n).

---

**Q: What is a Binary Tree vs a Binary Search Tree (BST)?**
A: A **Binary Tree** is a tree where each node has at most two children (left and right). A **BST** is a binary tree with the additional property: left subtree values < node value < right subtree values. BSTs allow O(log n) search, insert, delete on average, but degrade to O(n) if unbalanced.

---

**Q: What is a Balanced BST? Name some examples.**
A: A balanced BST maintains its height as O(log n) to guarantee efficient operations. Examples:
- **AVL Tree** – Strictly balanced; height difference between left and right subtrees ≤ 1. Uses rotations after insertions/deletions.
- **Red-Black Tree** – Less strictly balanced; uses color properties (red/black) and rotations. Faster insertions/deletions than AVL. Used in Java's `TreeMap`, C++ `std::map`.
- **B-Tree / B+ Tree** – Used in databases and file systems; multi-way balanced tree.

---

**Q: What is a Heap? What are its types?**
A: A heap is a complete binary tree satisfying the heap property:
- **Max-Heap** – Parent ≥ children; root is the maximum.
- **Min-Heap** – Parent ≤ children; root is the minimum.
Used to implement priority queues. Heapify is O(n); insert/delete is O(log n).

---

**Q: What is a Graph? What are ways to represent it?**
A: A graph is a set of vertices (nodes) and edges. Representations:
- **Adjacency Matrix** – 2D array; O(1) edge lookup; O(V²) space. Good for dense graphs.
- **Adjacency List** – Array of lists; O(V+E) space. Good for sparse graphs.
- **Edge List** – List of all edges; useful in some algorithms like Kruskal's.

---

**Q: What is the difference between a Tree and a Graph?**
A: A tree is a connected, acyclic undirected graph with N nodes and exactly N-1 edges. Every tree is a graph, but not every graph is a tree. Graphs can have cycles, disconnected components, and multiple paths between nodes.

---

**Q: What is a Trie?**
A: A Trie (prefix tree) is a tree-like data structure where each node represents a character. It is used for efficient string search, autocomplete, and spell checking. Insert and search are O(L) where L is the length of the string. More space-efficient alternatives include Compressed Tries (Patricia Trees).

---

**Q: What is the difference between a Stack and a Heap in memory?**
A: The **Stack** is for static memory allocation — stores function call frames, local variables, and return addresses. It's managed automatically (LIFO). The **Heap** is for dynamic memory allocation — `malloc`/`new` allocate here. The programmer manages it (or the garbage collector does). The stack is faster but limited in size; the heap is larger but slower due to allocation overhead.

---

**Q: What is a Bloom Filter?**
A: A Bloom filter is a space-efficient probabilistic data structure that tests whether an element is in a set. It can have **false positives** (says element is present when it's not) but **never false negatives**. It uses multiple hash functions and a bit array. Used in databases, caches, and network routers to avoid expensive lookups.

---

**Q: What is a Skip List?**
A: A skip list is a layered linked list where each layer is a subset of the one below, enabling O(log n) average search, insert, and delete. It is an alternative to balanced BSTs and is used in Redis (sorted sets) and LevelDB.

---

## 2. ALGORITHMS & COMPLEXITY

**Q: What is Big O notation? What does it measure?**
A: Big O notation describes the **upper bound** of an algorithm's time or space complexity as input size grows. It focuses on the dominant term and ignores constants. Common complexities:
- O(1) – Constant
- O(log n) – Logarithmic
- O(n) – Linear
- O(n log n) – Linearithmic
- O(n²) – Quadratic
- O(2ⁿ) – Exponential
- O(n!) – Factorial

---

**Q: What is the difference between Big O, Big Ω, and Big Θ?**
A:
- **Big O (O)** – Upper bound (worst case).
- **Big Ω (Omega)** – Lower bound (best case).
- **Big Θ (Theta)** – Tight bound (both upper and lower; average/exact case).

---

**Q: What is the difference between BFS and DFS?**
A:
| | BFS | DFS |
|---|---|---|
| Structure | Queue | Stack (or recursion) |
| Order | Level by level | Depth first |
| Use case | Shortest path (unweighted), connected components | Cycle detection, topological sort, maze solving |
| Memory | More (stores all nodes at a level) | Less (stores path to current node) |

---

**Q: What are the common sorting algorithms and their complexities?**
A:
| Algorithm | Best | Average | Worst | Space | Stable? |
|---|---|---|---|---|---|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | Yes |
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) | No |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | Yes |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | No |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | No |
| Counting Sort | O(n+k) | O(n+k) | O(n+k) | O(k) | Yes |
| Radix Sort | O(nk) | O(nk) | O(nk) | O(n+k) | Yes |

---

**Q: What is a stable sort? Why does it matter?**
A: A stable sort maintains the relative order of elements with equal keys. It matters when sorting by multiple criteria — e.g., sorting employees first by department, then by name. Merge Sort and Insertion Sort are stable; Quick Sort and Heap Sort are not (by default).

---

**Q: What is Divide and Conquer? Give examples.**
A: Divide and Conquer breaks a problem into smaller subproblems, solves them recursively, and combines results. Examples: Merge Sort, Quick Sort, Binary Search, Fast Fourier Transform (FFT), Strassen's Matrix Multiplication.

---

**Q: What is Dynamic Programming (DP)?**
A: DP solves problems by breaking them into overlapping subproblems and storing results to avoid redundant computation (memoization or tabulation). Two key properties:
- **Optimal Substructure** – Optimal solution can be built from optimal solutions of subproblems.
- **Overlapping Subproblems** – Subproblems recur multiple times.
Examples: Fibonacci, Knapsack, Longest Common Subsequence, Edit Distance, Coin Change.

---

**Q: What is the difference between Memoization and Tabulation?**
A:
- **Memoization (Top-Down)** – Recursive approach; cache results of subproblems as they're computed.
- **Tabulation (Bottom-Up)** – Iterative approach; fill a table starting from base cases up to the final answer.
Tabulation is usually faster (no recursion overhead); memoization is more intuitive.

---

**Q: What is a Greedy Algorithm?**
A: A greedy algorithm makes the locally optimal choice at each step, hoping it leads to a global optimum. It doesn't always yield the optimal solution. Examples where greedy works: Activity Selection, Huffman Coding, Dijkstra's (non-negative weights), Kruskal's/Prim's (MST). Counterexample: 0/1 Knapsack (requires DP).

---

**Q: What is Binary Search? What are its requirements?**
A: Binary Search finds a target in a **sorted** array by repeatedly halving the search space. Time complexity: O(log n). Requirement: The collection must be sorted and allow random access (array, not linked list). It can also be applied to answer spaces (e.g., "find the minimum value satisfying a condition").

---

**Q: What is a Shortest Path Algorithm? Name types.**
A:
- **Dijkstra's** – Single source, non-negative weights; O((V+E) log V) with a min-heap.
- **Bellman-Ford** – Single source, handles negative weights; detects negative cycles; O(VE).
- **Floyd-Warshall** – All-pairs shortest path; O(V³).
- **A\*** – Heuristic-based; used in pathfinding (games, GPS).

---

**Q: What is a Minimum Spanning Tree (MST)?**
A: An MST of a connected, weighted, undirected graph is a tree that connects all vertices with minimum total edge weight and no cycles. Algorithms:
- **Kruskal's** – Sort edges by weight; add edge if it doesn't form a cycle (uses Union-Find); O(E log E).
- **Prim's** – Start from a vertex; greedily add the cheapest edge to the MST; O(E log V) with heap.

---

**Q: What is the difference between recursion and iteration?**
A: Recursion calls itself, using the call stack for state. It's often more readable for tree/graph problems. Iteration uses loops, which are generally more memory-efficient. Recursion risks stack overflow for deep calls. Any recursive solution can be converted to iterative using an explicit stack.

---

**Q: What is Backtracking?**
A: Backtracking is a refined brute-force approach that builds candidate solutions incrementally and abandons (backtracks) partial solutions as soon as they can't lead to a valid answer. Used in: N-Queens, Sudoku solver, permutations/combinations, constraint satisfaction problems.

---

**Q: What is the P vs NP problem?**
A: **P** is the class of problems solvable in polynomial time. **NP** is the class of problems where a proposed solution can be *verified* in polynomial time. The question is whether P = NP (every verifiable problem can also be solved efficiently). Most believe P ≠ NP, but it's unproven. **NP-Hard** problems are at least as hard as the hardest NP problems; **NP-Complete** are both NP and NP-Hard (e.g., SAT, Travelling Salesman, Knapsack).

---

**Q: What is a Hash Collision and how is it resolved?**
A: A collision occurs when two keys produce the same hash. Resolution strategies:
- **Separate Chaining** – Each slot holds a linked list of colliding entries.
- **Linear Probing** – Check next slot sequentially.
- **Quadratic Probing** – Check slots at quadratic intervals.
- **Double Hashing** – Use a second hash function to determine the probe step.

---

## 3. OPERATING SYSTEMS

**Q: What is an Operating System?**
A: An OS is system software that manages hardware resources (CPU, memory, I/O) and provides services to programs. Core functions: process management, memory management, file system management, I/O management, security, and networking.

---

**Q: What is a Process vs a Thread?**
A:
- **Process** – An independent program in execution with its own memory space (code, heap, stack, data). Processes are isolated from each other.
- **Thread** – A lightweight unit of execution within a process. Threads share the same memory space (heap, code, data) but have their own stack and registers.
Threads are cheaper to create and context-switch than processes.

---

**Q: What is a Context Switch?**
A: A context switch is when the CPU switches from executing one process/thread to another. The OS saves the current state (registers, program counter, stack pointer) into the PCB (Process Control Block) and loads the state of the next process. It's a necessary overhead for multitasking.

---

**Q: What are the different states of a Process?**
A: A process can be in:
- **New** – Being created.
- **Ready** – Waiting to be assigned to a CPU.
- **Running** – Currently executing on a CPU.
- **Waiting/Blocked** – Waiting for an I/O event or resource.
- **Terminated** – Finished execution.

---

**Q: What is a Deadlock? What are its four necessary conditions?**
A: A deadlock is when two or more processes are stuck waiting for each other to release resources, indefinitely. The **four Coffman conditions** (all must hold simultaneously):
1. **Mutual Exclusion** – Resource held by only one process at a time.
2. **Hold and Wait** – Process holds a resource while waiting for another.
3. **No Preemption** – Resources can't be forcibly taken.
4. **Circular Wait** – Circular chain of processes each waiting on the next.

---

**Q: How can deadlocks be handled?**
A:
- **Prevention** – Eliminate one of the four conditions (e.g., impose resource ordering to prevent circular wait).
- **Avoidance** – Use algorithms like the **Banker's Algorithm** to only grant resources if safe state is maintained.
- **Detection & Recovery** – Allow deadlocks, detect via resource allocation graphs, then recover by killing a process or preempting resources.
- **Ignorance (Ostrich Algorithm)** – Pretend deadlocks don't happen (used by most OSes for simplicity).

---

**Q: What is CPU Scheduling? Name common algorithms.**
A: CPU scheduling determines which process runs next. Algorithms:
- **FCFS (First Come First Served)** – Simple, non-preemptive; can cause convoy effect.
- **SJF (Shortest Job First)** – Optimal for minimum average wait time; can cause starvation.
- **Round Robin** – Each process gets a time quantum; preemptive; fair.
- **Priority Scheduling** – Based on priority; can cause starvation (solved by aging).
- **Multilevel Queue / Feedback Queue** – Multiple queues for different priority classes.

---

**Q: What is Virtual Memory?**
A: Virtual memory allows processes to use more memory than physically available by using disk space as an extension of RAM. Each process gets a virtual address space mapped to physical memory via a **page table**. The MMU (Memory Management Unit) handles translation. When a needed page isn't in RAM, a **page fault** occurs and the OS loads it from disk.

---

**Q: What is Paging vs Segmentation?**
A:
- **Paging** – Divides memory into fixed-size blocks (pages in virtual memory, frames in physical). Eliminates external fragmentation but causes internal fragmentation.
- **Segmentation** – Divides memory into variable-size logical segments (code, stack, heap). Can cause external fragmentation but maps more naturally to program structure.
Modern systems often combine both (segmented paging).

---

**Q: What is Thrashing?**
A: Thrashing occurs when a system spends more time swapping pages in and out of memory (page faults) than executing processes. It happens when the combined working set of all processes exceeds available physical memory. Solution: reduce multiprogramming, use working set model.

---

**Q: What is the difference between Internal and External Fragmentation?**
A:
- **Internal Fragmentation** – Wasted space inside an allocated block (e.g., allocated 8KB when only 5KB needed).
- **External Fragmentation** – Free memory exists but is broken into small non-contiguous chunks that can't satisfy a large request.

---

**Q: What are Semaphores and Mutexes?**
A:
- **Mutex (Mutual Exclusion Lock)** – Binary lock; only the thread that locked it can unlock it. Used for exclusive access to a resource.
- **Semaphore** – An integer counter. A **binary semaphore** is similar to a mutex but any thread can signal it. A **counting semaphore** controls access to a pool of N resources. Used for synchronization and signaling.

---

**Q: What is a Race Condition?**
A: A race condition occurs when the behavior of a program depends on the relative timing/ordering of concurrent operations (threads/processes) accessing shared data. The outcome is unpredictable. Prevented by synchronization mechanisms (mutex, semaphore, atomic operations).

---

**Q: What is a System Call?**
A: A system call is the interface between a user-space program and the OS kernel. It transitions from user mode to kernel mode to request OS services like file I/O (`read`, `write`), process control (`fork`, `exec`, `wait`), memory allocation (`mmap`), and networking (`socket`).

---

**Q: What is the difference between User Mode and Kernel Mode?**
A:
- **User Mode** – Restricted access; cannot directly access hardware or critical memory areas. Most applications run here.
- **Kernel Mode** – Full access to hardware and system resources. The OS kernel runs here.
The switch happens via system calls, interrupts, or exceptions.

---

**Q: What is an Interrupt?**
A: An interrupt is a signal to the CPU that an event needs immediate attention. Types:
- **Hardware Interrupts** – From devices (keyboard, disk, network).
- **Software Interrupts / Traps** – Triggered by programs (system calls, exceptions like divide by zero).
The CPU saves its state, jumps to the Interrupt Service Routine (ISR), executes it, and resumes.

---

**Q: What is the difference between a Monolithic Kernel and a Microkernel?**
A:
- **Monolithic Kernel** – All OS services (file system, drivers, networking) run in kernel space. Faster but less modular. Examples: Linux, Unix.
- **Microkernel** – Only essential services (IPC, scheduling, basic memory) in kernel; everything else in user space. Slower (more IPC overhead) but more stable and secure. Examples: Mach, Minix, QNX.

---

**Q: What is the difference between Multiprogramming, Multitasking, and Multiprocessing?**
A:
- **Multiprogramming** – Multiple programs loaded in memory; CPU switches when one is waiting for I/O. Maximizes CPU utilization.
- **Multitasking** – Rapid CPU switching between processes (time-sharing), giving the illusion of simultaneous execution.
- **Multiprocessing** – Multiple CPUs/cores genuinely executing processes in parallel simultaneously.

---

**Q: What is IPC (Inter-Process Communication)?**
A: IPC mechanisms allow processes to communicate and synchronize. Methods:
- Pipes (anonymous and named)
- Message Queues
- Shared Memory (fastest)
- Semaphores
- Sockets
- Signals

---

## 4. COMPUTER NETWORKS

**Q: What is the OSI Model? Name all 7 layers.**
A: The OSI (Open Systems Interconnection) model is a conceptual framework for network communication:
1. **Physical** – Bits over physical medium (cables, signals)
2. **Data Link** – Frames, MAC addresses, error detection (Ethernet, Wi-Fi)
3. **Network** – Packets, IP addressing, routing (IP, ICMP)
4. **Transport** – Segments, end-to-end communication, reliability (TCP, UDP)
5. **Session** – Managing sessions/connections between applications
6. **Presentation** – Data format, encryption, compression (SSL/TLS at this layer conceptually)
7. **Application** – User-facing protocols (HTTP, FTP, DNS, SMTP)

---

**Q: What is the TCP/IP Model? How does it map to OSI?**
A: The TCP/IP model has 4 layers:
1. **Network Access / Link** – OSI Physical + Data Link
2. **Internet** – OSI Network (IP)
3. **Transport** – OSI Transport (TCP, UDP)
4. **Application** – OSI Session + Presentation + Application

---

**Q: What is the difference between TCP and UDP?**
A:
| | TCP | UDP |
|---|---|---|
| Connection | Connection-oriented (3-way handshake) | Connectionless |
| Reliability | Guaranteed delivery, ordering, error checking | No guarantees |
| Speed | Slower | Faster |
| Use case | HTTP, FTP, email, SSH | DNS, streaming, VoIP, gaming |
| Overhead | Higher | Lower |

---

**Q: Explain the TCP 3-way handshake.**
A:
1. **SYN** – Client sends a SYN (synchronize) packet to the server.
2. **SYN-ACK** – Server responds with SYN-ACK (synchronize-acknowledge).
3. **ACK** – Client sends an ACK (acknowledge).
Connection is now established. Teardown uses a 4-way FIN handshake.

---

**Q: What is the DNS? How does it work?**
A: DNS (Domain Name System) translates domain names (e.g., `google.com`) to IP addresses. Resolution process:
1. Browser checks local cache.
2. Queries the OS resolver / local DNS cache.
3. Queries a Recursive Resolver (usually from ISP).
4. Resolver queries Root Name Servers → TLD Servers (`.com`) → Authoritative Name Server.
5. IP address returned and cached.

---

**Q: What is HTTP vs HTTPS?**
A: HTTP (HyperText Transfer Protocol) is the foundation of web communication, transferring data in plaintext. HTTPS adds a TLS (Transport Layer Security) layer, providing encryption, authentication, and data integrity. HTTPS uses port 443; HTTP uses port 80.

---

**Q: What is the difference between HTTP/1.1, HTTP/2, and HTTP/3?**
A:
- **HTTP/1.1** – Text-based; one request per connection (with persistent connections and pipelining but head-of-line blocking).
- **HTTP/2** – Binary; multiplexing (multiple streams over one connection); header compression (HPACK); server push.
- **HTTP/3** – Runs over QUIC (UDP-based) instead of TCP; eliminates head-of-line blocking at the transport layer; faster handshakes.

---

**Q: What is a MAC address vs an IP address?**
A:
- **MAC Address** – Hardware address assigned to a NIC; 48-bit; used within a local network (Layer 2).
- **IP Address** – Logical address assigned by network configuration; 32-bit (IPv4) or 128-bit (IPv6); used for routing across networks (Layer 3).

---

**Q: What is NAT (Network Address Translation)?**
A: NAT maps private IP addresses within a local network to a single public IP address. It conserves IPv4 addresses and adds a layer of security. A router with NAT modifies packet headers to track which internal device made a request.

---

**Q: What is the difference between a Router, Switch, and Hub?**
A:
- **Hub** – Layer 1; broadcasts data to all ports; dumb device; creates collision domains.
- **Switch** – Layer 2; forwards frames to specific ports using MAC address tables; creates separate collision domains per port.
- **Router** – Layer 3; routes packets between different networks using IP addresses; connects LANs to the internet.

---

**Q: What is a Subnet? What is CIDR notation?**
A: A subnet is a logical subdivision of an IP network. CIDR (Classless Inter-Domain Routing) notation represents a subnet as an IP address followed by a prefix length (e.g., `192.168.1.0/24` means the first 24 bits are the network address and 8 bits for hosts, allowing 254 usable host addresses).

---

**Q: What is a Firewall?**
A: A firewall monitors and controls network traffic based on security rules. Types:
- **Packet Filtering** – Inspects packet headers (IP, port, protocol).
- **Stateful Inspection** – Tracks the state of connections.
- **Application Layer / Proxy Firewall** – Deep packet inspection at Layer 7.
- **Next-Gen Firewall (NGFW)** – Combines traditional firewall with IDS/IPS, application awareness.

---

**Q: What is the difference between a CDN and a Load Balancer?**
A:
- **CDN (Content Delivery Network)** – Distributes static content (images, JS, CSS) to edge servers geographically close to users, reducing latency.
- **Load Balancer** – Distributes incoming requests across multiple servers to ensure no single server is overwhelmed; improves availability and throughput.

---

**Q: What is WebSocket?**
A: WebSocket is a protocol providing full-duplex, persistent communication over a single TCP connection. Unlike HTTP, the server can push messages to the client without a request. Used in: real-time chat, live notifications, multiplayer games, collaborative tools.

---

**Q: What is ARP (Address Resolution Protocol)?**
A: ARP maps an IP address to a MAC address within a local network. When a device wants to communicate with another on the same subnet, it broadcasts an ARP request ("Who has IP X?"). The owner of that IP replies with its MAC address.

---

**Q: What is the difference between IPv4 and IPv6?**
A:
- **IPv4** – 32-bit addresses (~4.3 billion); written in dotted decimal (e.g., `192.168.1.1`).
- **IPv6** – 128-bit addresses (~3.4 × 10³⁸); written in hex (e.g., `2001:0db8::1`); features: no NAT needed, built-in IPSec, simplified headers, auto-configuration.

---

## 5. DATABASES

**Q: What is the difference between SQL and NoSQL databases?**
A:
| | SQL (Relational) | NoSQL |
|---|---|---|
| Schema | Fixed, predefined schema | Flexible/dynamic schema |
| Data model | Tables with rows/columns | Document, key-value, column, graph |
| ACID | Strong ACID guarantees | Often eventual consistency (BASE) |
| Scaling | Vertical (primarily) | Horizontal |
| Examples | MySQL, PostgreSQL, Oracle | MongoDB, Cassandra, Redis, DynamoDB |

---

**Q: What are ACID properties?**
A:
- **Atomicity** – A transaction is all-or-nothing; if any part fails, the whole transaction rolls back.
- **Consistency** – A transaction brings the database from one valid state to another; all rules/constraints must be satisfied.
- **Isolation** – Concurrent transactions execute as if they were sequential; intermediate states are not visible.
- **Durability** – Once committed, a transaction persists even after system failure (written to disk).

---

**Q: What are the different types of SQL Joins?**
A:
- **INNER JOIN** – Returns rows with matching values in both tables.
- **LEFT JOIN (LEFT OUTER)** – All rows from left table + matching rows from right (nulls if no match).
- **RIGHT JOIN** – All rows from right + matching from left.
- **FULL OUTER JOIN** – All rows from both tables; nulls where no match.
- **CROSS JOIN** – Cartesian product of both tables.
- **SELF JOIN** – Table joined with itself.

---

**Q: What is a Primary Key vs a Foreign Key?**
A:
- **Primary Key** – Uniquely identifies each row in a table; cannot be NULL; a table can have only one.
- **Foreign Key** – A column referencing the primary key of another table; enforces referential integrity.

---

**Q: What is Normalization? What are the Normal Forms?**
A: Normalization organizes a database to reduce redundancy and improve integrity by decomposing tables. Key Normal Forms:
- **1NF** – Atomic values; no repeating groups.
- **2NF** – 1NF + no partial dependency (non-key attribute depends on part of composite key).
- **3NF** – 2NF + no transitive dependency (non-key attribute depends on another non-key attribute).
- **BCNF** – Stronger form of 3NF.
- **4NF / 5NF** – Handle multi-valued and join dependencies.

---

**Q: What is Denormalization? When is it used?**
A: Denormalization intentionally introduces redundancy to improve read performance by reducing joins. Used in read-heavy systems, data warehouses, and OLAP. The trade-off is increased storage and complexity in maintaining data consistency.

---

**Q: What is an Index? What are the types?**
A: An index is a data structure (typically a B-tree or hash) that speeds up data retrieval at the cost of additional storage and slower writes. Types:
- **Clustered Index** – Determines physical order of data; one per table (usually primary key).
- **Non-Clustered Index** – Separate structure pointing to rows; multiple per table.
- **Composite Index** – Index on multiple columns.
- **Unique Index** – Enforces uniqueness.
- **Full-Text Index** – For text search.
- **Covering Index** – Contains all columns needed by a query; no need to access the table.

---

**Q: What is a Transaction Isolation Level?**
A: Isolation levels control how transactions interact with concurrent transactions:
- **Read Uncommitted** – Dirty reads possible; lowest isolation.
- **Read Committed** – No dirty reads; phantom reads possible.
- **Repeatable Read** – No dirty/non-repeatable reads; phantom reads possible.
- **Serializable** – Full isolation; highest level; slowest.

---

**Q: What are common concurrency problems in databases?**
A:
- **Dirty Read** – Reading uncommitted data from another transaction.
- **Non-Repeatable Read** – Same query returns different results within a transaction (another committed an update).
- **Phantom Read** – New rows appear in repeated queries (another committed an insert).
- **Lost Update** – Two transactions update the same row; one overwrites the other.

---

**Q: What is the difference between a View and a Materialized View?**
A:
- **View** – A virtual table defined by a query; data is fetched from base tables at query time; always up to date.
- **Materialized View** – A stored snapshot of the query result; physically saved; faster reads but can be stale; must be refreshed.

---

**Q: What is a Stored Procedure vs a Function vs a Trigger?**
A:
- **Stored Procedure** – Pre-compiled SQL statements stored in the DB; can have side effects, no return value required; called explicitly.
- **Function** – Must return a value; can be used in SELECT statements; typically no side effects.
- **Trigger** – Automatically executes in response to INSERT, UPDATE, or DELETE events on a table.

---

**Q: What is CAP Theorem?**
A: CAP Theorem states that a distributed data store can only guarantee **two of three** properties:
- **Consistency** – Every read gets the most recent write.
- **Availability** – Every request receives a response (not necessarily the latest data).
- **Partition Tolerance** – System continues operating despite network partitions.
Since partitions are inevitable in distributed systems, you typically choose between CP (e.g., HBase, Zookeeper) or AP (e.g., Cassandra, DynamoDB).

---

**Q: What is the difference between OLTP and OLAP?**
A:
| | OLTP | OLAP |
|---|---|---|
| Purpose | Daily transactions | Analytics/reporting |
| Operations | INSERT, UPDATE, DELETE | Complex SELECT, aggregations |
| Schema | Highly normalized | Denormalized (star/snowflake) |
| Queries | Simple, fast | Complex, slow |
| Examples | Banking, e-commerce | Data warehouses, BI tools |

---

**Q: What is Sharding?**
A: Sharding (horizontal partitioning) splits a large database into smaller, independent pieces (shards) distributed across multiple servers. Each shard holds a subset of the data (e.g., by user ID range or hash). Improves scalability but complicates cross-shard queries and transactions.

---

**Q: What is Replication in databases?**
A: Replication copies data from a primary (master) database to one or more replicas (slaves). Types:
- **Synchronous** – Write is confirmed only after all replicas are updated; strong consistency but slower.
- **Asynchronous** – Write confirmed after primary is updated; replicas catch up eventually; faster but potential data loss.
Used for high availability, fault tolerance, and read scaling.

---

## 6. OBJECT-ORIENTED PROGRAMMING (OOP)

**Q: What are the four pillars of OOP?**
A:
1. **Encapsulation** – Bundling data and methods into a class; hiding internal state via access modifiers.
2. **Abstraction** – Exposing only essential features; hiding implementation details (via abstract classes/interfaces).
3. **Inheritance** – A class (child) inherits properties and methods from another class (parent), promoting code reuse.
4. **Polymorphism** – The ability of different objects to respond to the same interface/method in different ways (method overriding, method overloading).

---

**Q: What is the difference between Overloading and Overriding?**
A:
- **Method Overloading** – Same method name, different parameters (type/number) in the same class. Resolved at compile time (static/compile-time polymorphism).
- **Method Overriding** – Subclass provides a specific implementation of a method defined in the parent class. Resolved at runtime (dynamic/runtime polymorphism).

---

**Q: What is the difference between an Abstract Class and an Interface?**
A:
| | Abstract Class | Interface |
|---|---|---|
| Instantiation | Cannot be instantiated | Cannot be instantiated |
| Methods | Can have implemented methods | All abstract (Java 8+ allows default methods) |
| Variables | Can have instance variables | Constants only (public static final) |
| Inheritance | Single inheritance | Multiple interfaces can be implemented |
| Use case | "is-a" with shared code | "can-do" contract |

---

**Q: What is the difference between Composition and Inheritance?**
A:
- **Inheritance** – "Is-a" relationship; subclass extends parent class. Tight coupling; changes in parent affect child.
- **Composition** – "Has-a" relationship; a class contains an instance of another class. Preferred for flexibility; reduces coupling. Example: a `Car` has-a `Engine` rather than Car extends Engine.
"Favor composition over inheritance" is a common design principle.

---

**Q: What is the SOLID principle?**
A:
- **S – Single Responsibility** – A class should have one reason to change.
- **O – Open/Closed** – Open for extension, closed for modification.
- **L – Liskov Substitution** – Subclasses should be substitutable for their parent class without breaking the program.
- **I – Interface Segregation** – Many specific interfaces are better than one general-purpose interface.
- **D – Dependency Inversion** – Depend on abstractions, not concretions.

---

**Q: What is a Design Pattern? Name the categories.**
A: Design patterns are reusable solutions to commonly occurring software design problems. Categories (Gang of Four):
- **Creational** – Object creation (Singleton, Factory, Abstract Factory, Builder, Prototype)
- **Structural** – Object composition (Adapter, Decorator, Facade, Proxy, Composite, Bridge, Flyweight)
- **Behavioral** – Object interaction (Observer, Strategy, Command, Iterator, Template Method, Chain of Responsibility, State)

---

**Q: What is the Singleton Pattern?**
A: Singleton ensures a class has only one instance and provides a global access point to it. Used for: logging, config, thread pools, database connections. Concerns: can be hard to test (global state), needs thread-safe implementation (double-checked locking or enum singleton in Java).

---

**Q: What is the Observer Pattern?**
A: Observer defines a one-to-many dependency: when one object (Subject) changes state, all its dependents (Observers) are notified automatically. Used in: event systems, pub-sub, MVC (model notifies views), UI frameworks.

---

**Q: What is the Factory Pattern?**
A: Factory defines an interface for creating an object but lets subclasses decide which class to instantiate. Decouples object creation from usage. Variants: Simple Factory, Factory Method, Abstract Factory (creates families of related objects).

---

**Q: What is Dependency Injection (DI)?**
A: DI is a technique where a class receives its dependencies from an external source rather than creating them itself. Types: Constructor Injection, Setter Injection, Interface Injection. Benefits: reduces coupling, improves testability (easy to mock dependencies). DI frameworks: Spring (Java), Angular (TS).

---

**Q: What is coupling and cohesion?**
A:
- **Coupling** – Degree of interdependence between modules. **Low coupling** is desirable; changes in one module minimally affect others.
- **Cohesion** – Degree to which elements of a module belong together. **High cohesion** is desirable; a class has a single, well-defined responsibility.
Goal: High cohesion, low coupling.

---

**Q: What is the difference between a shallow copy and a deep copy?**
A:
- **Shallow Copy** – Creates a new object but copies references to the same nested objects. Changes to nested objects reflect in both copies.
- **Deep Copy** – Creates a new object and recursively copies all nested objects. Completely independent copy.

---

## 7. COMPUTER ARCHITECTURE

**Q: What is the Von Neumann Architecture?**
A: Von Neumann architecture is the foundational design of modern computers, consisting of: CPU (ALU + Control Unit + Registers), Memory (stores both data and instructions), I/O devices, and a shared bus connecting them. The key feature is the stored-program concept — instructions and data share the same memory.

---

**Q: What is a CPU? What are its components?**
A: CPU (Central Processing Unit) is the brain of the computer:
- **ALU (Arithmetic Logic Unit)** – Performs arithmetic and logical operations.
- **Control Unit (CU)** – Fetches, decodes, and executes instructions.
- **Registers** – Extremely fast, small memory (PC, SP, IR, general-purpose registers).
- **Cache (L1, L2, L3)** – Hierarchical fast memory between CPU and RAM.

---

**Q: What is the CPU instruction cycle (Fetch-Decode-Execute)?**
A:
1. **Fetch** – Retrieve instruction from memory at address in Program Counter (PC); increment PC.
2. **Decode** – Control Unit decodes the instruction to determine operation and operands.
3. **Execute** – ALU or other units carry out the instruction.
4. **Write-back** – Result stored in register or memory.

---

**Q: What is Cache Memory? What are L1, L2, L3 caches?**
A: Cache is fast SRAM between the CPU and main memory that stores frequently accessed data. Hierarchy:
- **L1** – Smallest (~32KB), fastest, per core, stores most recently used data/instructions.
- **L2** – Larger (~256KB), slightly slower, per core.
- **L3** – Largest (~several MB), shared across cores, slower than L1/L2 but faster than RAM.
Cache exploits **locality of reference** (temporal and spatial locality).

---

**Q: What is Cache Coherence?**
A: In multi-core processors, each core has its own cache. Cache coherence ensures all caches have a consistent view of shared memory. Protocols like **MESI** (Modified, Exclusive, Shared, Invalid) manage this by tracking the state of cache lines.

---

**Q: What is pipelining in CPUs?**
A: Pipelining overlaps execution of multiple instructions by dividing the instruction cycle into stages (Fetch, Decode, Execute, Write-back). While one instruction is executing, the next is being decoded, and the next is being fetched. Improves throughput. Hazards (data, control, structural) can stall the pipeline.

---

**Q: What is the difference between RISC and CISC?**
A:
| | RISC | CISC |
|---|---|---|
| Instructions | Simple, fixed-size | Complex, variable-size |
| Execution | 1 clock cycle per instruction | Multiple cycles |
| Registers | Many | Fewer |
| Memory access | Load/store only | Any instruction |
| Examples | ARM, MIPS | x86 |

Modern x86 CPUs translate CISC instructions internally to RISC-like micro-ops.

---

**Q: What is the difference between 32-bit and 64-bit architectures?**
A: The bit width refers to the size of CPU registers and the addressable memory:
- **32-bit** – Can address 2³² = 4 GB of RAM.
- **64-bit** – Can address 2⁶⁴ bytes (theoretically; practical limits apply); better performance for math-intensive tasks; can run 32-bit software.

---

**Q: What is DMA (Direct Memory Access)?**
A: DMA allows peripheral devices (disk, GPU, NIC) to transfer data directly to/from memory without CPU involvement. The CPU initiates the transfer then works on other tasks; the DMA controller handles the transfer and signals the CPU via interrupt when done. Greatly improves I/O performance.

---

**Q: What is Endianness?**
A: Endianness describes the byte order in memory:
- **Big-Endian** – Most significant byte stored at lowest address. (Network byte order uses this.)
- **Little-Endian** – Least significant byte stored at lowest address. (x86 architecture uses this.)
Important for data exchange between different systems.

---

## 8. PROGRAMMING LANGUAGES & COMPILERS

**Q: What is the difference between a Compiler and an Interpreter?**
A:
- **Compiler** – Translates entire source code to machine code before execution. Faster execution; errors found at compile time. Examples: C, C++, Go.
- **Interpreter** – Translates and executes source code line by line at runtime. Slower; easier debugging. Examples: Python, Ruby, JavaScript (traditional).
Some languages use both (e.g., Java compiles to bytecode, then JVM interprets/JIT-compiles).

---

**Q: What is JIT (Just-In-Time) compilation?**
A: JIT compilation is a hybrid approach where code is compiled to machine code at runtime (not ahead of time). It can optimize based on actual runtime behavior (hot paths). Used in: Java HotSpot JVM, .NET CLR, V8 JavaScript engine.

---

**Q: What are the phases of a compiler?**
A:
1. **Lexical Analysis** – Tokenizes source code (lexemes → tokens).
2. **Syntax Analysis / Parsing** – Builds Abstract Syntax Tree (AST) from tokens; checks grammar.
3. **Semantic Analysis** – Type checking, scope resolution.
4. **Intermediate Code Generation** – Produces IR (e.g., three-address code).
5. **Optimization** – Improves IR (constant folding, dead code elimination).
6. **Code Generation** – Produces target machine code.

---

**Q: What is Garbage Collection? What are common algorithms?**
A: GC automatically reclaims memory occupied by objects no longer reachable by the program. Algorithms:
- **Reference Counting** – Track references to each object; collect when count reaches 0. Problem: can't handle circular references.
- **Mark and Sweep** – Mark all reachable objects; sweep and free the unmarked.
- **Generational GC** – Split heap into young/old generations; most objects die young; collect young gen frequently (used in JVM, Python).
- **Copying GC** – Copy live objects to a new space; old space is freed entirely.

---

**Q: What is the difference between a statically typed and a dynamically typed language?**
A:
- **Static Typing** – Types are checked at compile time; variables have fixed types. Examples: Java, C++, Go. More runtime performance, early error detection.
- **Dynamic Typing** – Types are checked at runtime; variables can hold any type. Examples: Python, JavaScript, Ruby. More flexible, faster to write.

---

**Q: What is the difference between pass-by-value and pass-by-reference?**
A:
- **Pass-by-Value** – A copy of the variable is passed; changes don't affect the original.
- **Pass-by-Reference** – A reference (address) to the variable is passed; changes affect the original.
Java is always pass-by-value, but for objects, the value passed is the reference (so the object's contents can be modified, but not the reference itself).

---

**Q: What is a closure?**
A: A closure is a function that captures and remembers the variables from its enclosing scope even after the outer function has returned. Used for: data encapsulation, callbacks, partial application. Found in JavaScript, Python, functional languages.

---

**Q: What is a pointer vs a reference?**
A:
- **Pointer** – Stores the memory address of a variable. Can be null, reassigned, and used for arithmetic. Explicit in C/C++.
- **Reference** – An alias for another variable. Must be initialized at declaration, cannot be null, cannot be reassigned to refer to another variable. Safer than raw pointers.

---

**Q: What is memory management in C/C++ vs Java?**
A:
- **C/C++** – Manual memory management; programmer uses `malloc`/`free` or `new`/`delete`. Can lead to memory leaks, dangling pointers, double frees.
- **Java** – Automatic memory management via Garbage Collector. No manual freeing needed, but has GC overhead and less predictable pauses.

---

**Q: What is the difference between concurrency and parallelism?**
A:
- **Concurrency** – Multiple tasks make progress by interleaving execution; not necessarily at the same instant. Deals with structure.
- **Parallelism** – Multiple tasks execute simultaneously on multiple cores/processors. Deals with execution.
Concurrency is about dealing with many things at once; parallelism is about doing many things at once.

---

**Q: What is a thread-safe function / data structure?**
A: Thread-safe code works correctly when called from multiple threads simultaneously without data corruption or race conditions. Achieved via locks (mutexes), atomic operations, immutable data, or thread-local storage.

---

## 9. SOFTWARE ENGINEERING

**Q: What is the difference between Black-box and White-box testing?**
A:
- **Black-box Testing** – Tester has no knowledge of internal implementation; tests based on inputs/outputs and specifications. Examples: functional testing, acceptance testing.
- **White-box Testing** – Tester knows internal code structure; tests paths, branches, and conditions. Examples: unit testing, code coverage analysis.

---

**Q: What are the types of software testing?**
A:
- **Unit Testing** – Individual components/functions.
- **Integration Testing** – Interaction between components.
- **System Testing** – Full system behavior.
- **Acceptance Testing (UAT)** – Validates against business requirements.
- **Regression Testing** – Ensures new changes don't break existing functionality.
- **Performance Testing** – Load, stress, and scalability testing.
- **Smoke Testing** – Basic sanity check after a build.

---

**Q: What is TDD (Test-Driven Development)?**
A: TDD is a development practice where you write failing tests *before* writing the code to pass them. Cycle: **Red** (write failing test) → **Green** (write minimal code to pass) → **Refactor** (clean code). Leads to better-designed, testable, and documented code.

---

**Q: What is the difference between Agile and Waterfall?**
A:
| | Waterfall | Agile |
|---|---|---|
| Approach | Sequential phases | Iterative sprints |
| Flexibility | Low; requirements fixed | High; adapts to change |
| Delivery | End of project | Incremental (each sprint) |
| Feedback | Late | Continuous |
| Best for | Well-defined, stable requirements | Evolving, unclear requirements |

---

**Q: What is CI/CD?**
A:
- **Continuous Integration (CI)** – Developers frequently merge code to a shared branch; automated builds and tests run on each merge to catch issues early.
- **Continuous Delivery (CD)** – Code is automatically built and tested; can be deployed to production with minimal manual steps.
- **Continuous Deployment** – Every passing build is automatically deployed to production.
Tools: Jenkins, GitHub Actions, CircleCI, GitLab CI.

---

**Q: What is version control? What is the difference between Git merge and rebase?**
A: Version control tracks changes to code over time. Git is the most popular system.
- **Merge** – Combines histories of two branches, creating a merge commit. Preserves history; non-destructive.
- **Rebase** – Moves or replays commits from one branch onto another, creating a linear history. Cleaner history but rewrites commits; avoid on shared branches.

---

**Q: What is REST? What are its constraints?**
A: REST (Representational State Transfer) is an architectural style for designing networked APIs. Constraints:
1. **Client-Server** – Separation of concerns.
2. **Stateless** – Each request contains all info needed; server stores no client state.
3. **Cacheable** – Responses must define if cacheable.
4. **Uniform Interface** – Consistent resource-based URLs, HTTP methods.
5. **Layered System** – Client doesn't know if it's talking directly to the server.
6. **Code on Demand** (optional) – Server can send executable code.

---

**Q: What is the difference between REST and GraphQL?**
A:
| | REST | GraphQL |
|---|---|---|
| Endpoints | Multiple; one per resource | Single endpoint |
| Data fetching | Over-fetch or under-fetch | Fetch exactly what you need |
| Versioning | /v1, /v2 | Schema evolves; no versioning |
| Use case | Simple CRUD APIs | Complex, interconnected data needs |

---

**Q: What is microservices architecture?**
A: Microservices decompose an application into small, independently deployable services, each responsible for a specific business capability. Services communicate via APIs (REST, gRPC) or messaging (Kafka, RabbitMQ). Benefits: independent scaling, deployment, and tech choices. Challenges: distributed system complexity, network latency, eventual consistency.

---

**Q: What is the difference between monolithic and microservices architecture?**
A:
| | Monolithic | Microservices |
|---|---|---|
| Deployment | Single deployable unit | Many independent services |
| Scalability | Scale entire app | Scale individual services |
| Development | Simpler initially | Complex; needs DevOps maturity |
| Failure | One bug can crash all | Fault isolation |
| Tech stack | Uniform | Polyglot possible |

---

**Q: What is Docker?**
A: Docker is a containerization platform that packages an application and its dependencies into a container — a lightweight, portable, isolated unit. Unlike VMs, containers share the host OS kernel, making them faster and smaller. Key concepts: Dockerfile (build instructions), Image (immutable template), Container (running instance), Docker Hub (registry).

---

**Q: What is the difference between a Container and a Virtual Machine?**
A:
| | Container | Virtual Machine |
|---|---|---|
| OS | Shares host OS kernel | Full guest OS |
| Size | MBs | GBs |
| Startup | Seconds | Minutes |
| Isolation | Process-level | Hardware-level |
| Performance | Near-native | More overhead |

---

**Q: What is Kubernetes?**
A: Kubernetes (K8s) is a container orchestration platform that automates deployment, scaling, and management of containerized applications. Key concepts: Pod (group of containers), Service (stable endpoint), Deployment (desired state), Node (machine), Cluster (group of nodes), Ingress (routing external traffic).

---

## 10. SECURITY

**Q: What is the difference between Authentication and Authorization?**
A:
- **Authentication** – Verifying *who you are* (identity). Examples: password, biometrics, MFA.
- **Authorization** – Verifying *what you are allowed to do* (permissions). Examples: RBAC, ACLs.
You must authenticate before you can be authorized.

---

**Q: What is SQL Injection? How is it prevented?**
A: SQL Injection is an attack where malicious SQL code is inserted into an input field, manipulating the backend query. Example: entering `' OR '1'='1` bypasses a login. Prevention:
- Use **parameterized queries / prepared statements**.
- Input validation and sanitization.
- Least privilege DB accounts.
- ORMs (which use parameterized queries by default).

---

**Q: What is XSS (Cross-Site Scripting)?**
A: XSS injects malicious scripts into web pages viewed by other users. Types:
- **Stored XSS** – Malicious script saved in the database.
- **Reflected XSS** – Script in URL/request reflected back.
- **DOM-based XSS** – Script executes via client-side JS.
Prevention: Output encoding/escaping, Content Security Policy (CSP), HTTP-only cookies.

---

**Q: What is CSRF (Cross-Site Request Forgery)?**
A: CSRF tricks an authenticated user's browser into making an unwanted request to a site they're logged into. Prevention: CSRF tokens (unique per session/request), `SameSite` cookie attribute, checking `Origin`/`Referer` headers.

---

**Q: What is the difference between symmetric and asymmetric encryption?**
A:
- **Symmetric** – Same key for encryption and decryption. Faster; key distribution is a problem. Examples: AES, DES.
- **Asymmetric** – Public key encrypts, private key decrypts (or vice versa for signatures). Slower but solves key distribution. Examples: RSA, ECC.
TLS uses asymmetric encryption for key exchange, then symmetric for the actual data.

---

**Q: What is Hashing vs Encryption?**
A:
- **Hashing** – One-way function; cannot be reversed. Used for storing passwords, checksums. Examples: SHA-256, bcrypt.
- **Encryption** – Two-way; can be decrypted with the right key. Used for secure data transmission/storage.
Passwords should be **hashed** (with salt + bcrypt/Argon2), never encrypted.

---

**Q: What is a JWT (JSON Web Token)?**
A: JWT is a compact, self-contained token for transmitting claims between parties. Structure: `Header.Payload.Signature` (Base64Url encoded). The signature verifies the token hasn't been tampered with. Used in stateless authentication — the server doesn't need to store session state. Important: JWTs are not encrypted by default (just signed); sensitive data shouldn't be in the payload unless using JWE.

---

**Q: What is HTTPS / TLS? How does TLS work?**
A: TLS (Transport Layer Security) encrypts communication between client and server. TLS Handshake:
1. Client sends `ClientHello` (TLS version, cipher suites).
2. Server responds with certificate (public key) and chosen cipher suite.
3. Client verifies certificate against CAs (Certificate Authorities).
4. Key exchange (e.g., Diffie-Hellman) to establish a session key.
5. Encrypted communication begins using symmetric encryption.

---

**Q: What is the principle of least privilege?**
A: Every user, program, or system component should have only the minimum access necessary to perform its function. Reduces attack surface and limits damage from breaches.

---

## 11. SYSTEM DESIGN FUNDAMENTALS

**Q: What is horizontal scaling vs vertical scaling?**
A:
- **Vertical Scaling (Scale Up)** – Add more CPU/RAM to an existing machine. Simpler but has hardware limits and single point of failure.
- **Horizontal Scaling (Scale Out)** – Add more machines. More complex but unlimited scale; requires load balancing and stateless design.

---

**Q: What is a Load Balancer? What algorithms does it use?**
A: A load balancer distributes traffic across multiple servers. Algorithms:
- **Round Robin** – Distributes requests sequentially.
- **Least Connections** – Sends to server with fewest active connections.
- **IP Hash** – Hashes client IP to consistently route to the same server (sticky sessions).
- **Weighted Round Robin** – Servers with higher capacity get more traffic.

---

**Q: What is caching? What are common caching strategies?**
A: Caching stores frequently accessed data in fast storage (memory) to reduce latency and backend load. Strategies:
- **Cache-aside (Lazy Loading)** – App checks cache first; on miss, fetches from DB and populates cache.
- **Write-through** – Write to cache and DB simultaneously.
- **Write-back (Write-behind)** – Write to cache immediately; sync to DB asynchronously.
- **Read-through** – Cache fetches from DB on miss automatically.
Eviction policies: LRU (Least Recently Used), LFU (Least Frequently Used), TTL.

---

**Q: What is a Message Queue? Why is it used?**
A: A message queue allows asynchronous communication between services. The producer sends messages to a queue; the consumer processes them at its own pace. Benefits: decouples services, handles traffic spikes (buffering), improves resilience (consumer can be down temporarily). Examples: RabbitMQ, Kafka, SQS.

---

**Q: What is the difference between a Message Queue and a Pub-Sub system?**
A:
- **Message Queue** – One-to-one; a message is consumed by one consumer.
- **Pub-Sub (Publish-Subscribe)** – One-to-many; a message published to a topic is delivered to all subscribers. Examples: Kafka topics, Google Pub/Sub, SNS.

---

**Q: What is eventual consistency?**
A: In a distributed system, eventual consistency guarantees that if no new updates are made to a piece of data, all replicas will *eventually* converge to the same value. It trades immediate consistency for availability and performance. Used in: DynamoDB, Cassandra, DNS.

---

**Q: What is a CDN and how does it work?**
A: A CDN (Content Delivery Network) is a distributed network of servers (Points of Presence / PoPs) that cache and serve content closer to users geographically. When a user requests content, they are routed to the nearest PoP. If content is cached (cache hit), it's served directly; otherwise, the PoP fetches from origin (cache miss), caches it, then serves it. Examples: Cloudflare, AWS CloudFront, Akamai.

---

**Q: What is a Reverse Proxy?**
A: A reverse proxy sits between clients and backend servers. It receives requests, forwards them to the appropriate server, and returns the response. Benefits: load balancing, SSL termination, caching, DDoS protection, hiding backend topology. Examples: Nginx, HAProxy, Cloudflare.

---

## 12. MISCELLANEOUS / GENERAL CS

**Q: What is the difference between a process and a program?**
A: A **program** is a static set of instructions stored on disk. A **process** is a program in execution — it has memory, resources, a state, and a process ID. One program can have multiple processes.

---

**Q: What is Unicode vs ASCII?**
A:
- **ASCII** – 7-bit encoding; 128 characters; English letters, digits, and symbols only.
- **Unicode** – Universal character set supporting 100,000+ characters from all languages. Implementations: UTF-8 (variable width, ASCII-compatible, dominant on the web), UTF-16 (used internally in Windows, Java, JavaScript), UTF-32 (fixed 4 bytes per character).

---

**Q: What is the difference between a 'null', 'undefined', and '0' in programming?**
A: While language-dependent:
- **null** – Intentional absence of a value; explicitly assigned.
- **undefined** – Variable declared but not assigned a value (JavaScript); uninitialized.
- **0** – The integer zero; a valid, defined value.
In most languages, `null` and `0` are different types; in some contexts (C), `NULL` is just `0` as a pointer.

---

**Q: What is a regular expression (regex)?**
A: A regex is a sequence of characters defining a search pattern. Used for string matching, validation, and parsing. Key symbols: `.` (any char), `*` (0+), `+` (1+), `?` (0 or 1), `^` (start), `$` (end), `[]` (character class), `|` (or), `\d` (digit), `\w` (word char).

---

**Q: What is the difference between synchronous and asynchronous programming?**
A:
- **Synchronous** – Operations execute sequentially; each waits for the previous to complete. Simpler; blocks the thread.
- **Asynchronous** – Operations can start without waiting for previous ones to complete. Non-blocking; uses callbacks, promises, async/await. Essential for I/O-heavy tasks.

---

**Q: What is a race condition vs a deadlock?**
A:
- **Race Condition** – Outcome depends on unpredictable timing of concurrent operations; can cause data corruption.
- **Deadlock** – Two or more processes are permanently blocked waiting for each other.
Both are concurrency bugs; race conditions can be intermittent and hard to reproduce.

---

**Q: What is the difference between a compile-time error and a runtime error?**
A:
- **Compile-time error** – Detected by the compiler before execution (syntax errors, type errors). The program won't compile.
- **Runtime error** – Occurs during execution (null pointer dereference, division by zero, stack overflow). The program compiles but crashes or behaves incorrectly.

---

**Q: What is a memory leak?**
A: A memory leak occurs when a program allocates memory but fails to release it when no longer needed. Over time, this exhausts available memory. Common in C/C++ (forgetting to `free`/`delete`), or in managed languages when references are unintentionally kept alive (e.g., event listeners not removed).

---

**Q: What is tail recursion?**
A: Tail recursion is a form of recursion where the recursive call is the last operation in the function. Compilers/interpreters can optimize it into iteration (tail call optimization / TCO), avoiding stack overflow. Not all languages support TCO (Python doesn't; Scheme and some functional languages do).

---

**Q: What is the difference between a library and a framework?**
A:
- **Library** – A collection of reusable functions; *you* call the library. You are in control of the flow. Example: NumPy, jQuery.
- **Framework** – Defines the structure of your application; *it* calls your code (Inversion of Control). Example: Django, Spring, Angular.
"Don't call us, we'll call you" — the Hollywood Principle describes frameworks.

---

**Q: What is an API?**
A: An API (Application Programming Interface) is a set of defined rules that allows different software applications to communicate. It abstracts the underlying implementation and exposes only necessary functionality. Types: Web APIs (REST, GraphQL, gRPC), OS APIs, Library APIs.

---

**Q: What is the difference between TCP/IP and UDP/IP ports?**
A: Ports are logical endpoints within a host, identified by a 16-bit number (0–65535). Common well-known ports: HTTP (80), HTTPS (443), FTP (21), SSH (22), DNS (53), SMTP (25). Ports 0–1023 are "well-known"; 1024–49151 are registered; 49152–65535 are dynamic/ephemeral.

---

**Q: What is Base64 encoding? Is it encryption?**
A: Base64 encodes binary data into ASCII text using 64 printable characters. It is **not** encryption — it provides no security; it's purely a data encoding format for transmitting binary data in text-based protocols (email, HTTP, JWT). It increases data size by ~33%.

---

**Q: What is the difference between `==` and `===` in JavaScript?**
A: `==` (loose equality) performs type coercion before comparison (`"5" == 5` is `true`). `===` (strict equality) checks both value and type with no coercion (`"5" === 5` is `false`). Always prefer `===` to avoid unexpected behavior.

---

**Q: What is a race-free program?**
A: A program is race-free if shared data is always protected by proper synchronization so that no concurrent access can lead to an inconsistent state. Tools: mutexes, atomic operations, immutable data structures, thread-local storage.

---

**Q: What is the difference between a monad and a functor (functional programming)?**
A:
- **Functor** – A container that can be mapped over; supports `map` (apply a function to the value inside). Examples: arrays, `Maybe`.
- **Monad** – A functor that also supports `flatMap`/`bind` to chain operations while handling context (e.g., null handling, side effects, async). Examples: `Promise`, `Maybe`, `Either`, `List`. Monads allow composing computations that include effects without nesting.

---

This covers the breadth of CS fundamentals across every major domain. Good luck with your interviews! 🚀