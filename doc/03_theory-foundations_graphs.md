# Philosophers Project: A Graph-Based Approach to Concurrency

I’m exploring a graph-based resource allocation approach. My goal is to transform this classic problem into an exercise in formal methods, graph theory, and concurrent systems design. 

Baba (back to basics) > development of framework:



---

## graphs

A graph \( G = (V, E) \) is a mathematical structure where:
- \( V \) is a set of vertices (nodes), representing entities.
- \( E \subseteq V \times V \) is a set of edges, representing relationships between entities.

Formally:
- If \( E \) consists of unordered pairs \( \{u, v\} \), the graph is undirected.
- If \( E \) consists of ordered pairs \( (u, v) \), the graph is directed.

In the Dining Philosophers problem:
- \( V = P \cup F \), where \( P \) is the set of philosophers and \( F \) is the set of forks.
- \( E \) represents resource relationships:
  - Directed edges \( (p_i, f_j) \) in a request graph indicate philosopher \( p_i \) wants fork \( f_j \).
  - Directed edges \( (p_i, f_j) \) in an allocation graph indicate \( p_i \) holds \( f_j \).

Graphs are powerful - they abstract complex systems into a structure I can analyze with algorithms
— deadlock detection becomes a graph traversal problem.



## Bipartite Graphs: Structure and Relevance

A graph \( G = (V, E) \) is bipartite if \( V \) can be partitioned into two disjoint sets \( V_1 \) and \( V_2 \) such that:
- \( V_1 \cap V_2 = \emptyset \)
- \( V_1 \cup V_2 = V \)
- Every edge \( e \in E \) connects a vertex in \( V_1 \) to a vertex in \( V_2 \).

In my project:
- \( V_1 = P \) (philosophers)
- \( V_2 = F \) (forks)
- Edges represent "holds" or "wants" relationships, inherently bipartite because philosophers only interact with forks, not other philosophers directly.

This bipartiteness is key: it constrains the graph’s structure, making certain properties (like maximum matching) easier to compute. 
A cycle in this graph requires an even length (alternating between \( V_1 \) and \( V_2 \)), which aligns with deadlock conditions
e.g., \( P_0 \to F_0 \to P_1 \to F_1 \to P_0 \).



## Why Graphs for Dining Philosophers?

The problem is a resource allocation challenge:
- Philosophers (processes/threads) compete for forks (resources).
- Deadlock occurs when a circular wait forms.
- Starvation occurs when a philosopher is perpetually denied resources.

A graph model captures this:
- **Allocation Graph**: Tracks current resource ownership.
- **Request Graph**: Tracks pending resource requests.
- **Combined Wait-For Graph**: Merges allocation and request edges to reveal dependencies.

A cycle in the wait-for graph indicates deadlock 

e.g., \( P_0 \) waits for \( F_1 \), held by \( P_1 \), who waits for \( F_0 \), held by \( P_0 \)
My graph-based approach aims to detect and prevent these cycles proactively.




## Representing Graphs in Code. theory > practice

several options for encoding this graph in C:

### 1. Adjacency Matrix
A matrix \( A \) where \( A[i][j] = 1 \) if there’s an edge from node \( i \) to node \( j \), else 0.
- For \( n \) philosophers and \( n \) forks, \( |V| = 2n \), so \( A \) is \( 2n \times 2n \).
- Space complexity: \( O(n^2) \).
- Time complexity for edge lookup: \( O(1) \).

```c
typedef struct s_allocation_graph {
    int **adjacency_matrix;    // [2n][2n] matrix: philosophers + forks
    int **request_matrix;      // Separate matrix for requests
    pthread_mutex_t graph_lock;
    int num_nodes;             // 2n
} t_allocation_graph;



2. Adjacency List

Each node maintains a list of neighbors.

	Space complexity: ( O(|V| + |E|) ), better for sparse graphs.
	Time complexity for edge lookup: ( O(deg(v)) ), where ( deg(v) ) is the degree of vertex ( v ).
	For my small, dense graph (max ( 2n ) edges), the adjacency matrix wins for simplicity and constant-time edge checks.


3. Edge List

A list of ( (u, v) ) pairs.

	Space: ( O(|E|) ).

Less practical here due to slower lookups.
I’ll stick with matrices for their clarity in cycle detection and safety algorithms.




## Cycle Detection: The Heart of Deadlock Prevention

A cycle in my wait-for graph signals deadlock. 
I’ll use Depth-First Search (DFS) to detect it:


bool has_cycle(t_allocation_graph *graph, int v, bool *visited, bool *rec_stack) {
    visited[v] = true;
    rec_stack[v] = true;

    for (int u = 0; u < graph->num_nodes; u++) {
        if (graph->adjacency_matrix[v][u]) {
            if (!visited[u] && has_cycle(graph, u, visited, rec_stack))
                return true;
            else if (rec_stack[u])
                return true;
        }
    }
    rec_stack[v] = false;
    return false;
}

bool would_create_cycle(t_allocation_graph *graph, int philo_id, int fork_id) {
    bool visited[graph->num_nodes];
    bool rec_stack[graph->num_nodes];
    memset(visited, 0, sizeof(visited));
    memset(rec_stack, 0, sizeof(rec_stack));
    
    // Temporarily add edge
    graph->adjacency_matrix[philo_id][fork_id] = 1;
    bool result = has_cycle(graph, philo_id, visited, rec_stack);
    graph->adjacency_matrix[philo_id][fork_id] = 0; // Roll back
    return result;
}


Complexity: ( O(|V| + |E|) ) per call.

Rec_stack: Tracks nodes in the current DFS path to detect back edges.
This is rigorous: it catches cycles before they solidify into deadlocks.




## Safety Checking: Banker’s Algorithm in a Graph Context

Dijkstra’s Banker’s Algorithm ensures a state is safe
i.e., there exists an execution order where all philosophers can eat without deadlock. Here’s my adaptation:


bool is_safe_state(t_allocation_graph *graph) {
    int work[graph->num_resources];       // Available forks
    bool finish[graph->num_philosophers]; // Completion status
    int i, j;

    // Initialize: assume all forks are available minus allocated ones
    for (i = 0; i < graph->num_resources; i++) {
        work[i] = 1; // Fork initially free
        for (j = 0; j < graph->num_philosophers; j++)
            work[i] -= graph->adjacency_matrix[j][i + graph->num_philosophers];
    }
    memset(finish, 0, sizeof(finish));

    // Find a philosopher who can finish
    bool progress;
    do {
        progress = false;
        for (i = 0; i < graph->num_philosophers; i++) {
            if (!finish[i]) {
                bool can_eat = true;
                for (j = 0; j < graph->num_resources; j++) {
                    if (graph->request_matrix[i][j + graph->num_philosophers] > work[j]) {
                        can_eat = false;
                        break;
                    }
                }
                if (can_eat) {
                    finish[i] = true;
                    progress = true;
                    for (j = 0; j < graph->num_resources; j++)
                        work[j] += graph->adjacency_matrix[i][j + graph->num_philosophers];
                }
            }
        }
    } while (progress);

    // Safe if all philosophers finished
    for (i = 0; i < graph->num_philosophers; i++)
        if (!finish[i]) return false;
    return true;
}


Complexity: ( O(n^2 m) ), where ( n ) is philosophers, ( m ) is resources.

Intuition: Simulate resource release until everyone eats or we’re stuck.




## Graph Visualization: Seeing the Problem


an ASCII representation for 3 philosophers (( P_0, P_1, P_2 )) and 3 forks (( F_0, F_1, F_2 )) in a deadlock state:


   P0 -----> F1
   |         |
   v         v
   F0 <----- P1
   |         |
   v         v
   P2 -----> F2
   |         |
   v         v
   F0 <----- P2 (cycle back to F0)

( P_0 ) holds ( F_0 ), requests ( F_1 ).
( P_1 ) holds ( F_1 ), requests ( F_2 ).
( P_2 ) holds ( F_2 ), requests ( F_0 ).
Cycle: ( P_0 \to F_1 \to P_1 \to F_2 \to P_2 \to F_0 \to P_0 ).



## Why This Approach?

Theoretical Rigor
Formal Verification: I can prove deadlock freedom using graph properties (e.g., acyclicity).
Graph Theory: I’m applying directed acyclic graphs (DAGs) and bipartite matching principles.
Complexity Analysis: Every algorithm has clear time/space bounds.

Systems Design
Scalability: Extends to multi-resource problems beyond forks.
Concurrency: Works for threads (mutexes) and processes (shared memory).
Robustness: Prevents both deadlock and starvation with careful tuning.

Intellectual interest
I’m not just coding—I’m learning to reason about systems in a way that fucking interests me
bridging practical C programming with theoretical computer science.



## Implementation Challenges

### Performance
Overhead: Graph operations are ( O(n^2) ) vs. ( O(1) ) mutex locks.
Mitigation: Optimize matrix access with bitsets or sparse representations.
Contention: Single graph_lock risks bottlenecks.
Solution: Fine-grained locking per philosopher or lock-free updates.

### Complexity
Correctness: Subtle bugs in cycle detection or safety checks could ruin guarantees.
Strategy: Extensive unit tests + formal invariants (e.g., no philosopher holds >2 forks).
Scalability: Large ( n ) blows up matrix size.
Fix: Switch to adjacency lists for ( n > 100 ).

Bonus Part
Shared Memory: Use shm_open and mmap for graph matrices.
Semaphores: Replace pthread_mutex_t with sem_t for inter-process sync.
