---
title: Look-through paths in directed cyclic graphs
date: 2023-07-12 14:27:31 +0530
categories: [Technical]
tags: [technical, algorithms] # TAG names should always be lowercase
summary: Computing the product of edge-weights along each path in a directed cyclic graph.
math: true
---

Look-throughs are paths in a graph, along with aggregations on edge metadata, the simplest example being the product of edge weights along the path. These are especially useful in graphs that depict hierarchy or ownership. Consider a graph of investment funds where each fund can invest in stocks, or other funds. Say a fund $A$ invests $40\%$ of it's funds in another fund $B$, and the remaining $60\%$ in a stock $S1$. $B$ further invests $30\%$ of its value in stock $S1$ and the remaining in stock $S2$. This can then be modeled as a directed graph where the weight of each edge $X \to Y$ is the composition of $Y$ in $X$.

In the above scenario, $A$'s net investment in $S1$ is:

$$
\underbrace{0.6}_\text{It's direct investment in S1} + \underbrace{0.4 * 0.3}_\text{It's investment in S1 through B} = 0.72
$$

## What do look-through paths tell us?

As an example, consider the graph below. Each edge $X → Y$ in this graph depicts composition (that is, $X$ is composed of weight % of $Y$). Additionally, say $A$ has a net value of 100$. By performing look-through, we can get answers to the following questions:

1. The lowest-level components of $A$ are clearly $L1$, $L2$, and $L3$. But how much of $A$ is actually $L1$ (or $L2$ or $L3$)?
2. Assuming that $A$'s value is just the sum of the value of its constituents, how much value comes from $L1$, $L2$, and $L3$ respectively? (This would just entail multiplying the net value, with weights along the paths to the leaf nodes.)
3. How much value comes from $L2$ via $B$?

![Fig 1](/assets/img/posts/Lookthrough/fig_1.svg)
_Fig 1: A sample graph for which look-through paths are to be computed_

Look-through computation for the most part just comes down to graph traversal, while calculating the product of edge weights. However, the complexity arises from the fact that the graph may have a cycle, for example, the one formed by $C1$, $C2$, and $C3$ in Fig 1. Note that the edge weights are _fractional_. Otherwise, the product of the inifinte paths might tend to infinity.

## Detecting Cycles

_Cycles_, are essentially [strongly connected components](https://en.wikipedia.org/wiki/Strongly_connected_component) (SCCs henceforth) containing more than one node. There are many well-known algorithms for identifying the SCCs of a graph - [Tarjan's](https://en.wikipedia.org/wiki/Tarjan%27s_strongly_connected_components_algorithm) and [Kosaraju's](https://en.wikipedia.org/wiki/Kosaraju%27s_algorithm), for instance.

## A note on paths in an SCC

Between any two nodes in an SCC, there are an infinite number of paths. In the example above, the paths from $C1$ to $C2$ are:

- $C1 \to C2$
- $C1 \to C2 \to C3 \to C1 \to C2$
- $C1 \to C2 \to C3 \to C1 \to C2 \to C3 \to C1 \to C2$ and so on

or, $C1 \to \underbrace{(C2 \to C3 \to C1)}_\text{n times} \to C2$, where n $\in [0, \infty)$.

Given that any two nodes in an SCC are connected by infinitely many paths, identifying individual paths is futile. What one might be interested to know instead are net weights through an SCC given an entry and exit point. That is, if a look-through path enters the SCC through $C1$ and exits through $C2$, what would the net weight then be through that SCC? In the example above, the look-through path $A \to C1 \to \\{ C1 \leftrightarrow C2 \leftrightarrow C3 \\} \to C2 \to L2$ depicts the aggregate of all paths entering the SCC through $C1$, and exiting through $C2$. To get the net weight of this path, we first need to find the weight of $C1 \to \\{C1 \leftrightarrow C2 \leftrightarrow C3\\} \to C3$.

## Computing path-weights for entry-exit pairs

In order to compute path-weights within an SCC for each entry-exit pairs, consider the SCC as a stand-alone graph, with a few additional dummy nodes to depict exits. In the graph below, $C1^\ast$, $C2^\ast$, and $C3^\ast$ are dummy nodes representing all transitions leaving the SCC through $C1$, $C2$, and $C3$ respectively. Note that the weight of $C1 \to C1^\ast$ here is the sum of weights of $C1 \to X$ transitions where $X \notin \\{C1, C2, C3\\}$.

![Fig 2.a](/assets/img/posts/Lookthrough/fig_2a.svg)
_Fig 2.a: The cycle in Fig 1 as a stand-alone graph, with dummy nodes depicting exits from the SCC_

Consider the adjacency matrix, $M$ of the above graph. $M_{C1, C2^\ast}$ is the weight of the edge (or path of length 1) from $C1$ to $C2^\ast$. Basic graph theory tells us that the matrix $M^k$ gives us net weights of paths of length $k$. For example, $M^k_{C1, C2^\ast}$ is the net weight of all paths of length $k$ from $C1$ to $C2^\ast$.
Hence, the net weight of all paths from $C1 \to C2^\ast$ (that is, entering the SCC through $C1$ and leaving through $C2$) would be $\sum M^k_{C1, C2^\ast}$ where $k \in [1, \infty)$. That’s a bit complicated. However, we can make this simpler with some trickery.

Assume the graph has three edges of weight $1.0$ that go from each dummy node — $C1^\ast$, $C2^\ast$, and $C3^\ast$ — to itself as shown below. For this modified graph, for every path of length $k$ from $X \to Y$ where $X \in \\{C1, C2, C3\\}$ and $Y \in \\{C1^\ast, C2^\ast, C3^\ast\\}$, there is an equivalent path of length $k + 1$ with same weight, obtained by appending the $Y \to Y$ edge of weight $1$.

![Fig 2.b](/assets/img/posts/Lookthrough/fig_2b.svg)
_Fig 2.b: The graph in Fig 2.a, but with dummy edges of weight 1.0 going from each dummy exit node to itself_

In general, if $M$ and $N$ are the adjacency matrices of the graphs in Fig 2.a and Fig 2.b respectively, then

$$
\sum M^k_{X, Y} = N^k_{X, Y}
$$

where $X \in \\{C1, C2, C3\\}$ and $Y \in \\{C1^\ast, C2^\ast, C3^\ast\\}$.

The net weight of all paths from $C1 \to C2^\ast$ was found to be $\sum M^k_{C1, C2^\ast}$ where $k \in [1 to \infty)$. Now, this can be simplified to

$$
\lim_{n \to \infty} N^k_{C1, C2^\ast}
$$

Moreover, the modified graph in Fig 2.b is similar to an [absorbing Markov chain](https://en.wikipedia.org/wiki/Absorbing_Markov_chain) and the same derivation for [absorbing probabilities](https://en.wikipedia.org/wiki/Absorbing_Markov_chain#Absorbing_probabilities) can be used to find the above limit, reducing the entire exercise to three matrix operations.

The net weights of paths through the SCC for each entry-exit pair is obtained from the matrix $B = R(1 - Q)-1$ where:

- $R$ is the matrix of edges from the original nodes to the dummy exit (or absorbing) nodes and
- $Q$ is the matrix of edges from the original, non-terminal (or transient) nodes.

## Deriving the formula

The adjacency matrix of the graph in Fig 2.b is given below:

$$
\begin{array}{c|cccccc}
 & C1 & C2 & C3 & C1^\ast & C2^\ast & C3^\ast \\
\hline
C1 & \color{red}{0} & \color{red}{0.4} & \color{red}{0} & \color{maroon}{0.6} & \color{maroon}{0} & \color{maroon}{0} \\
C2 & \color{red}{0} & \color{red}{0} & \color{red}{0.2} & \color{maroon}{0} & \color{maroon}{0.8} & \color{maroon}{0} \\
C3 & \color{red}{0.5} & \color{red}{0} & \color{red}{0} & \color{maroon}{0} & \color{maroon}{0} & \color{maroon}{0.5} \\
C1^\ast & \color{green}{0} & \color{green}{0} & \color{green}{0} & \color{fuchsia}{1} & \color{fuchsia}{0} & \color{fuchsia}{0} \\
C2^\ast & \color{green}{0} & \color{green}{0} & \color{green}{0} & \color{fuchsia}{0} & \color{fuchsia}{1} & \color{fuchsia}{0} \\
C3^\ast & \color{green}{0} & \color{green}{0} & \color{green}{0} & \color{fuchsia}{0} & \color{fuchsia}{0} & \color{fuchsia}{1}
\end{array}
$$

Note that the matrix can be divided into 4 parts:

- $Q$: The $\color{red}{\text{red}}$ part, is the matrix of edges going between the original, non-terminal (or transient) nodes.
- $R$: The $\color{maroon}{\text{maroon}}$ part, is the matrix of edges from the original nodes to the dummy exit (or absorbing) nodes.
- $0$: The $\color{green}{\text{green}}$ part is just a zero matrix as no nodes originating from the dummy nodes go to the original nodes. This is zero by definition, since the dummy nodes are representative of nodes outside the SCC once the path has left the SCC.
- $I$: The $\color{fuchsia}{\text{fuchsia}}$ part is just an identity matrix. This is owing to the dummy edges created in Fig 2.b going from each dummy node to itself.

The adjacency matrix,

$$
P = \begin{bmatrix}
Q & R \\
0 & I
\end{bmatrix},
$$

and,

$$
P^2 = \begin{bmatrix}
Q^2 & QR + R \\
0 & I
\end{bmatrix},
$$

$$
P^3 = \begin{bmatrix}
Q^3 & Q^2R + QR + R \\
0 & I
\end{bmatrix},
$$

and in general,

$$
Pk = \begin{bmatrix}
Q^k & R(1 + Q + Q^2 + … + Q^k) \\
0 & I
\end{bmatrix},
$$

When $k \to \infty$, $Q^k \to 0$ as $Q$ is fractional.
Hence, applying the formula for geometric progression, we get

$$
\lim_{k \to \infty} P^k = \begin{bmatrix}
0 & R(1 - Q)^{-1} \\
0 & 1
\end{bmatrix}
$$

Note that the only non-zero portions of the matrix are:

- Trivially, the bottom-right part which comprises of only the dummy edges of weight 1
- The top-right, containing the net path-weights of $X \to Y$ transitions where $X \in \\{C1, C2, C3\\}$ and $Y \in \\{C1^\ast, C2^\ast, C3^\ast\\}$.

The net weights of paths through the SCC for each entry-exit pair can then be obtained from the matrix $B = R(1 - Q)^{-1}$.

## Substituting the resolved SCCs

After obtaining the net path-weights for entry-exit pairs in the SCC, we can modify the graph to make it acyclic.

![Fig 2.c](/assets/img/posts/Lookthrough/fig_2c.svg)
_Fig 2.c: The graph in Fig 2.a made acyclic by adding dummy nodes_

In the modified graph above, the blue nodes are _virtual_ nodes depicting the exit of a path from the SCC. The red edges are aggregated equivalents of paths going through the SCC. The green edges are where paths can lead, once it has exited the SCC. Note that in this case, weights of all the green edges are 1, due to the fact that $C1$, $C2$, and $C3$ only go to one node each outside the SCC. If there were multiple outgoing edges, the weights of the green edges would be in proportion to the weights of the original outgoing edges.

## Finding look-through paths

After resolving cycles, the graph is now acyclic, and a simple DFS can be done to find all look-through paths.
