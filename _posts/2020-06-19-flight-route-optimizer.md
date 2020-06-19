---
title: A route optimizer for the Breitling 100/24 race
layout: post
use_fontawesome: true
use_code: true
use_math: true
use_toc: true
permalink: projects/:title
---


<br>

## Introduction
---

<p class="centered">
    <img src="{{ site.baseurl }}/images/flight-route-optimizer/breitling.jpg" style="width: 50%">
</p>

<div class="justified">
    <p><span class="font-weight-bold">Abstract:</span> In the late 2000s, early 2010s, the watch manufacturing company Breitling organized airplane racing competitions in France every year. In order to win, a participant had to complete 100 “touch-and-go” maneuvers in distinct airdromes in a 24-hour timeframe by traveling a shorter distance than its competitors. Groups of airdromes are predefined and each group must be visited at least once to ensure some geographical diversity. The length of a stage must not exceed a certain maximum threshold distance based on the average fuel consumption of a plane and the capacity of its tank. We consider a simplified version of the problem where the time component is ignored (variant of the Traveling Salesman Problem). We consider four methods to solve the problem.</p>
    <p><span class="font-weight-bold">Keywords:</span> branch-and-bound, dynamic constraint generation, LP relaxation, MILP, total unimodularity, Traveling Salesman Problem (TSP)</p>
	<p><span class="font-weight-bold">Programming language:</span> Julia</p>
    <a href="https://github.com/antcrepin/flight-route-optimizer" class="btn btn-light">
        <i class="fab fa-github"></i> Repo
    </a>
</div>

<br>

## Notations
---

### Parameters

| Parameter     | Domain           | Description  |
|:-------------:|:-------------:| -----|
| $$n$$      | $$\mathbb{N}^*$$ | total number of airdromes, labeled $$1,\dots,n$$ |
| $$I$$      | $$[\![1,n]\!]$$      |   departure airdrome |
| $$F$$ | $$[\![1,n]\!]$$     |    arrival airdrome |
| $$k$$ | $$[\![1,n]\!]$$      |    minimum number of airdromes to be visited |
| $$m$$ | $$\mathbb{N}$$      |    total number of groups, labeled $$1,\dots,m$$ |
| $$R_i,\ i\in[\![1,n]\!]$$ |  $$[\![0,m]\!]$$ |    group of the airdrome $$i$$, equals 0 if the airdrome does not belong to any group |
| $$D_{ij},\ (i,j)\in[\![1,n]\!]^2$$ | $$\mathbb{N}$$     |    **rounded** distance between the airdromes $$i$$ and $$j$$ |
| $$\Delta$$ | $$\mathbb{N}^*$$  |    maximum distance that can be traveled without landing to refuel |

<br>

### Decision variables

| Decision variable     | Domain           | Description  | Used in model(s) |
|:-------------:|:-------------:| -----|:-------------:|
| $$x_{ij},\ (i,j)\in[\![1,n]\!]^2$$ | $$\{0,1\}$$    |    $$x_{ij}$$ equals 1 if the plane directly goes from $$i$$ to $$j$$, 0 otherwise | 1, 2, 3, 4 |
| $$u_i,\ i\in[\![1,n]\!]$$ |  $$\mathbb{R_+}$$ |    order of the airdrome $$i$$ such that the airdromes are visited with $$u$$ increasing| 1 |
| $$q_{ij},\ (i,j)\in[\![1,n]\!]^2$$ |  $$\mathbb{R_+}$$ |    flow circulating from the airdrome $$i$$ to the airdrome $$j$$ | 2 |
| $$\phi_i,\ i\in[\![1,n]\!]$$ |   $$\{0,1\}$$ |    flow consumption at the airdrome $$i$$ | 2 |
| $$z_i,\ i\in[\![1,n]\!]$$ |  $$\{0,1\}$$ |    $$z_i$$ equals 1 if the airdrome $$i$$ is part of the selected subset, 0 otherwise| 3, 4 |
| $$y_{ij},\ (i,j)\in[\![1,n]\!]^2$$ |  $$\mathbb{R_+}$$ |    $$y_{ij}$$ equals 1 if both airdromes $$i$$ and $$j$$ are both parts of the selected subset, 0 otherwise<br>$$y_{ij}$$ simulates $$z_iz_j$$| 3, 4 |
| $$t_i,\ i\in[\![1,n]\!]$$ |  $$\{0,1\}$$ |    $$t_i$$ equals 1 if the airdrome $$i$$ is visited by the plane, 0 otherwise| 4 |

<br>

## Mathematical modeling
---

### Optimization graph

We consider the directed optimization graph $$G=(V,A)$$ where $$V$$ is the set of airdromes and

$$
A = \left\{(i,j)\in[\![1,n]\!]^2\ \text{such that}\ D_{ij}\leqslant\Delta\ \text{and}\ i\neq j\right\}.
$$

> We excluded self-loops and arcs whose length exceeds the threshold distance $$\Delta$$.

<br>

### Objective

The problem aims at **minimizing the total traveled distance** which can be expressed as follows:

$$
\sum_{i,j}{D_{ij}x_{ij}}.
$$

<br>

### Constraints

- Each airdrome can be visited by the plane at most once, *i.e.*, there is at most one landing and one takeoff:  

  $$
  \sum_j{x_{ij}}+ \sum_j{x_{ji}} \leqslant 2\quad \forall i\in[\![ 1, n]\!].
  $$  

- The plane starts its route at $$I$$ and ends at $$F$$. For every other airdrome, the plane completes a "touch-and-go" maneuver so there are as many landings as takeoffs:  

  $$
  \sum_j{x_{ij}}-\sum_j{x_{ji}} = \delta_{iI}-\delta_{iF}\quad  \forall i\in[\![ 1, n]\!],
  $$  

  where $$\delta$$ is the Kronecker delta.  

- It is easy to see that an optimal solution for this problem does not contain any cycle, so the selected arcs have a **tree structure** (the number of visited airdromes exceeds the number of selected arcs by one unit). The plane must visit at least $$k$$ airdromes:  

  $$
  \sum_{i,j}{x_{ij}} \geqslant k-1.
  $$  

- Each predefined group of airdromes must be visited at least once:  

  $$
  \sum_{i,j:\ R_i=r\ \text{or}\ R_j=r}{x_{ij}} \geqslant 1\quad\forall r\in [\![ 1, m]\!].
  $$

<br>

### Connectivity constraint

At this point the connectivity of the solution is not guaranteed. The set of selected arcs can contain several connected components, one of which is a tree and all the others are cycles called **subtours**.

> There are **several possible formulations** for the subtour elimination constraint.

<br>

## Method 1: order of visit
---

It is possible to **avoid subtours by labeling the airdromes with respect to their order of visit**. Using the decision variables $$u$$, we can force the plane to visit the airdromes with $$u$$ increasing:

$$
u_j - u_i + n(1-x_{ij}) \geqslant 1\quad \forall (i,j)\in A.
$$

Thus we obtain a **first formulation** of the problem with $$\mathcal{O}(n^2)$$ decision variables and $$\mathcal{O}(n^2)$$ constraints.

$$
\tag{$F1$}
\label{F1}
\left\{
\begin{aligned}
    \text{minimize} && \sum_{i,j}{D_{ij}x_{ij}} \\
    \text{subject to} && \sum_{i,j}{x_{ij}} &\geqslant& k-1 \\
    && \sum_{i,j:\ R_i=r\ \text{or}\ R_j=r}{x_{ij}} &\geqslant& 1 && \forall r\in [\![ 1, m]\!]\\
    && \sum_j{x_{ij}}+ \sum_j{x_{ji}} &\leqslant& 2 && \forall i\in[\![ 1, n]\!] \\
    && \sum_j{x_{ij}}-\sum_j{x_{ji}} &=& \delta_{iI}-\delta_{iF} && \forall i\in[\![ 1, n]\!] \\
    && u_j - u_i + n(1-x_{ij}) &\geqslant& 1 && \forall (i,j)\in A \\
    && u_i &\geqslant& 0 && \forall i\in[\![ 1, n]\!]\\
    && x_{ij} &\in& \{0,1\} && \forall (i,j)\in A
\end{aligned}
\right.
$$

\eqref{F1} is solved thanks to a solver like CPLEX Optimizer or [CBC](https://projects.coin-or.org/Cbc). These solvers use the **branch-and-bound algorithm whose performance is highly dependent on the ability to find good-quality bounds** (lower bounds here) by solving LP relaxations at each visited node of the search tree.

For the smallest instances, the following table compares the optimal value to the lower bound corresponding to the LP relaxation at the root of the search tree.

| Instance     | Optimal value  | Root node lower bound |
|:-------------:|:-------------:|:-----:|
| $$\texttt{n6}$$ | 12 | 7.00 |
| $$\texttt{n20_1}$$ | 91 | 34.40 |
| $$\texttt{n20_2}$$ | 79 | 56.00 |
| $$\texttt{n20_3}$$ | 90 | 56.00 |
| $$\texttt{n20_4}$$ | 96 | 38.20 |
| $$\texttt{n30}$$ | 130 | 54.60 |
| $$\texttt{n40}$$ | 114 | 77.25 |
| $$\texttt{n50}$$ | 168 | 93.24 |
| $$\texttt{n70}$$ | 235 | 97.81 |

> The lower bounds we obtain from this formulation are not very sharp.

<br>

## Method 2: single flow
---

[Gavish and Graves (1978)](https://dspace.mit.edu/handle/1721.1/5363) gave a formulation for the subtour elimination constraints that significantly improves the quality of the relaxation while keeping a polynomial number of constraints. The methods consists in using **artificial flow variables** ($$q$$ and $$\phi$$ here).

We obtain a **second formulation** of the problem with $$\mathcal{O}(n^2)$$ decision variables and $$\mathcal{O}(n^2)$$ constraints.

$$
\tag{$F2$}
\label{F2}
\left\{
\begin{aligned}
    \text{minimize} && \sum_{i,j}{D_{ij}x_{ij}} \\
    \text{subject to} && \sum_{i,j}{x_{ij}} &\geqslant& k-1 \\
    && \sum_{i,j:\ R_i=r\ \text{or}\ R_j=r}{x_{ij}} &\geqslant& 1 && \forall r\in [\![ 1, m]\!]\\
    && \sum_j{x_{ij}}+ \sum_j{x_{ji}} &\leqslant& 2 && \forall i\in[\![ 1, n]\!] \\
    && \sum_j{x_{ij}}-\sum_j{x_{ji}} &=& \delta_{iI}-\delta_{iF} && \forall i\in[\![ 1, n]\!] \\
    && (n-1)x_{ij} - q_{ij} &\geqslant& 0 && \forall (i,j)\in A \\
	&& \sum_i{q_{Ii}} - \sum_i{\phi_i} &=& 0 \\
	&& \sum_j{q_{ij}}-\sum_j{q_{ji}} - \phi_i &=& 0 && \forall i\in[\![ 1, n]\!] \\
	&& \sum_j{x_{ji}} - \phi_i &=& 0 && \forall i\in[\![ 1, n]\!] \\
	&& q_{ij} &\geqslant& 0 && \forall (i,j)\in A\\
    && x_{ij} &\in& \{0,1\} && \forall (i,j)\in A\\
	&& \phi_i &\in& \{0,1\} && \forall i\in[\![ 1, n]\!]
\end{aligned}
\right.
$$

<br>

### Benefit

> Intuitively we can expect the lower bounds provided by \eqref{F2} to be tighter that the ones from \eqref{F1}.

In \eqref{F2}, even if the variables $$x_{ij}$$ are not integer, we will have $$\sum_j{x_{ji}} = \phi_i > 0$$ as soon as the airdrome $$i$$ is in the selected route. Then,

$$
\sum_j{q_{ji}} = \sum_j{q_{ij}}-\phi_i < \sum_j{q_{ij}}.
$$

Hence **cycles are also excluded from the solution of the LP relaxation**. That was not necessarily the case in \eqref{F1}.


For the first instances, we compare the optimal value to the lower bound obtained from the LP relaxation of \eqref{F2}.

| Instance     | Optimal value  | Root node lower bound |
|:-------------:|:-------------:|:-----:|
| $$\texttt{n6}$$ | 12 | 7.96 |
| $$\texttt{n20_1}$$ | 91 | 44.23 |
| $$\texttt{n20_2}$$ | 79 | 57.03 |
| $$\texttt{n20_3}$$ | 90 | 57.03 |
| $$\texttt{n20_4}$$ | 96 | 49.39 |
| $$\texttt{n30}$$ | 130 | 65.90 |
| $$\texttt{n40}$$ | 114 | 82.17 |
| $$\texttt{n50}$$ | 168 | 108.10 |
| $$\texttt{n70}$$ | 235 | 114.74 |

As expected, the lower bounds we obtain from this formulation are sharper than the ones from \eqref{F1}.

> The branch-and-bound algorithm should perform better with \eqref{F2}.

<br>

## Method 3: standard constraint generation
---

The absence of subtours can also be expressed by the fact that **each subset of airdromes must not contain any subtour**:

$$
\sum_{(i,j)\in S^2}{x_{ij}}\leqslant |S|-1\quad \forall S\subset[\![1,n]\!],\ |S|\geqslant2.
$$

We obtain a **third formulation** of the problem with $$\mathcal{O}(n^2)$$ decision variables and $$\mathcal{O}(2^n)$$ constraints.

$$
\tag{$F3$}
\label{F3}
\left\{
\begin{aligned}
    \text{minimize} && \sum_{i,j}{D_{ij}x_{ij}} \\
    \text{subject to} && \sum_{i,j}{x_{ij}} &\geqslant& k-1 \\
    && \sum_{i,j:\ R_i=r\ \text{or}\ R_j=r}{x_{ij}} &\geqslant& 1 && \forall r\in [\![ 1, m]\!]\\
    && \sum_j{x_{ij}}+ \sum_j{x_{ji}} &\leqslant& 2 && \forall i\in[\![ 1, n]\!] \\
    && \sum_j{x_{ij}}-\sum_j{x_{ji}} &=& \delta_{iI}-\delta_{iF} && \forall i\in[\![ 1, n]\!] \\
    && \sum_{(i,j)\in S^2}{x_{ij}} &\leqslant& |S|-1 && \forall S\subset[\![1,n]\!]\\
    && x_{ij} &\in& \{0,1\} && \forall (i,j)\in A
\end{aligned}
\right.
$$

Unlike the previous formulations, \eqref{F3} contains an exponential number of constraints, therefore solving it as it stands is not an option. For example the case $$n=20$$ would lead to more than $$2^{20}\simeq 10^6$$ constraints to generate.

> To overcome this difficulty we consider another course of action. The subtour elimination constraints will be **generated dynamically**.

- We start by solving the model without any subtour elimination constraint.
- Then we detect the subtours in the solution and add the corresponding constraints to the model (a subtour is fully characterized by the set $$S$$ of airdromes that are involved) and solve it again.
- We **repeat these steps until the solution contains no subtour**.

<br>

### Master problem

The current set of identified subtours is denoted by $$\mathcal{C}$$. At each iteration we solve the following master problem.

$$
\tag{$M3$}
\label{M3}
\left\{
\begin{aligned}
    \text{minimize} && \sum_{i,j}{D_{ij}x_{ij}} \\
    \text{subject to} && \sum_{i,j}{x_{ij}} &\geqslant& k-1 \\
    && \sum_{i,j:\ R_i=r\ \text{or}\ R_j=r}{x_{ij}} &\geqslant& 1 && \forall r\in [\![ 1, m]\!]\\
    && \sum_j{x_{ij}}+ \sum_j{x_{ji}} &\leqslant& 2 && \forall i\in[\![ 1, n]\!] \\
    && \sum_j{x_{ij}}-\sum_j{x_{ji}} &=& \delta_{iI}-\delta_{iF} && \forall i\in[\![ 1, n]\!] \\
    && \sum_{(i,j)\in S^2}{x_{ij}} &\leqslant& |S|-1 && \forall S\in\mathcal{C}\\
    && x_{ij} &\in& \{0,1\} && \forall (i,j)\in A
\end{aligned}
\right.
$$

<br>

### Subtour detection subproblem

The current solution of \eqref{M3} is denoted by $$(\tilde{x}_{ij})_{i,j}$$. We want to identify the subset of airdromes $$\bar{S}$$ that maximizes the quantity

$$
\sum_{(i,j)\in S^2}{\tilde{x}_{ij}}-|S|+1
$$

for $$S\subset[\![1,n]\!]$$.

It can be done by using the binary decision variables $$y$$ and $$z$$ and considering the following MILP.

$$
\tag{$S3$}
\label{S3}
\left\{
\begin{aligned}
    \text{maximize} && \sum_{i,j}{\tilde{x}_{ij}y_{ij}}-\sum_i{z_i}+1 \\
    \text{subject to} && y_{ij}-z_i &\leqslant& 0 && \forall (i,j)\in A\\
    && y_{ij}-z_j &\leqslant& 0 && \forall (i,j)\in A\\
    && y_{ij} &\geqslant& 0 && \forall (i,j)\in A\\
    && z_i &\in& \{0,1\} && \forall i\in [\![1,n]\!]
\end{aligned}
\right.
$$

The optimal solution of \eqref{S3} is denoted by $$\left((y_{ij}^*)_{i,j},(z_i^*)_i\right)$$.

- If the optimal value of the objective is positive, it means that the subtour constraint is violated for  
  
  $$
  \bar{S}=\{i\in[\![1,n]\!]\ \text{such that}\ z_i^*=1\},
  $$  
  
  and we can **add a new constraint** to the master problem: $$\mathcal{C}\leftarrow\mathcal{C}\cup\bar{S}$$.

- Otherwise $$(\tilde{x}_{ij})_{i,j}$$ has a **tree structure** and is the optimal solution of the initial problem.

<br>

## Method 4: advanced constraint generation
---

Being able to **detect subtours by solving a LP instead of a MILP** would enhance the efficiency of the previous solution method. [Wolsey (1998)](https://books.google.fr/books/about/Integer_Programming.html?id=x7RvQgAACAAJ) proposed a way to do this with a reformulation of the problem.

We slightly modify the problem and now search for a **single cyclical component**. The use of the arc from the arrival airdrome $$F$$ to the departure airdrome $$I$$ will be imposed, even if its length exceeds $$\Delta$$.
> The optimization graph becomes $$G'=(V,A')$$ with $$A'=A\cup\{(F,I)\}$$.

- We use the binary variables $$t$$ to express that there are as many landings as takeoffs in every airdrome:  

  $$
  x_{F,I}=1, \quad\sum_j{x_{ij}}+ \sum_j{x_{ji}} = 2t_i.
  $$  

- The connectivity constraint must be reformulated as well since the structure of the solutions has changed. We now want to avoid the subtours that do not contain the departure airdrome $$I$$. The formulation we choose is **more demanding** than before. For each subset of airdromes $$S$$, the number of selected internal arcs must be lower than the number of selected airdromes is every subset of $$S$$ with $$\lvert S\rvert-1$$ elements:  

  $$
  \sum_{(i,j)\in S^2}{x_{ij}}\leqslant\sum_{i\in S\setminus\{p\}}{t_i}\quad\forall S\subset[\![1,n]\!]\setminus\{I\},\ \forall p\in S.
  $$

The **fourth and final formulation** of the problem contains $$\mathcal{O}(n^2)$$ decision variables and $$\mathcal{O}(n2^n)$$ constraints.

$$
\tag{$F4$}
\left\{
\begin{aligned}
    \text{minimize} && \sum_{i,j}{D_{ij}x_{ij}} \\
    \text{subject to} && x_{F,I} &=& 1 \\
    && \sum_j{x_{ij}}+ \sum_j{x_{ji}} - 2t_i &=& 0 && \forall i\in[\![ 1, n]\!]\\
    && \sum_{i,j}{x_{ij}} &\geqslant& k \\
    && \sum_{i,j:\ R_i=r\ \text{or}\ R_j=r}{x_{ij}} &\geqslant& 1 && \forall r\in [\![ 1, m]\!]\\
    && \sum_j{x_{ij}}-\sum_j{x_{ji}} &=& 0 && \forall i\in[\![ 1, n]\!] \\
    && \sum_{(i,j)\in S^2}{x_{ij}}-\sum_{i\in S\setminus\{p\}}{t_i}&\leqslant&0 && \forall S\subset[\![1,n]\!]\setminus\{I\},\ \forall p\in S\\
    && t_i &\in& \{0,1\} && \forall i\in[\![ 1, n]\!]\\
    && x_{ij} &\in& \{0,1\} && \forall (i,j)\in A'
\end{aligned}
\right.
$$

We adopt the **same iterative approach** than before.

<br>

### Master problem

The dynamic set $$\mathcal{C}$$ now contains the couples $$(S,p)$$ (with $$p\in S$$) that have been detected over the iterations. At each iteration we solve the following master problem.

$$
\tag{$M4$}
\label{M4}
\left\{
\begin{aligned}
    \text{minimize} && \sum_{i,j}{D_{ij}x_{ij}} \\
    \text{subject to} && x_{F,I} &=& 1 \\
    && \sum_j{x_{ij}}+ \sum_j{x_{ji}} - 2t_i &=& 0 && \forall i\in[\![ 1, n]\!]\\
    && \sum_{i,j}{x_{ij}} &\geqslant& k \\
    && \sum_{i,j:\ R_i=r\ \text{or}\ R_j=r}{x_{ij}} &\geqslant& 1 && \forall r\in [\![ 1, m]\!]\\
    && \sum_j{x_{ij}}-\sum_j{x_{ji}} &=& 0 && \forall i\in[\![ 1, n]\!] \\
    && \sum_{(i,j)\in S^2:\ i\neq I,j\neq I}{x_{ij}}-\sum_{i\in S\setminus\{p,I\}}{t_i}&\leqslant&0 && \forall (S,p)\in\mathcal{C}\\
    && t_i &\in& \{0,1\} && \forall i\in[\![ 1, n]\!]\\
    && x_{ij} &\in& \{0,1\} && \forall (i,j)\in A'
\end{aligned}
\right.
$$

<br>

### Subtour detection subproblem

The current solution of \eqref{M4} is denoted by $$\left((\tilde{t}_i)_i,(\tilde{x}_{ij})_{i,j}\right)$$. We want to identify the couple $$(\bar{S},\bar{p})$$ that maximizes the quantity

$$
\sum_{(i,j)\in S^2}{\tilde{x}_{ij}}-\sum_{i\in S\setminus\{p\}}{\tilde{t}_i}
$$

for $$S\subset[\![1,n]\!]\setminus\{I\}$$ and $$p\in S$$.

> It is easier to fix $$p\neq I$$ and search for the corresponding set $$S$$.

We use the binary decision variables $$y$$ and $$z$$, and consider the $$n-1$$ following subproblems.

For all $$p\neq I$$,

$$
\tag{$S4$}
\label{S4}
\left\{
\begin{aligned}
    \text{maximize} && \sum_{i\neq I,j\neq I}{\tilde{x}_{ij}y_{ij}}-\sum_{i\notin\{p,I\}}{\tilde{t}_iz_i} \\
    \text{subject to} && z_p &=& 1 \\
    && y_{ij}-z_i &\leqslant& 0 && \forall (i,j)\in A'\\
    && y_{ij}-z_j &\leqslant& 0 && \forall (i,j)\in A'\\
    && y_{ij} &\geqslant& 0 && \forall (i,j)\in A'\\
    && z_i &\in& \{0,1\} && \forall i\in [\![1,n]\!]
\end{aligned}
\right.
$$

> For a given $$p\neq I$$, \eqref{S4} can be solved **exactly** by relaxing the variables $$z_i$$ because the constraint matrix is **totally unimodular**.

The optimal solution of \eqref{S4} is denoted by $$\left((y_{ij}^*)_{i,j},(z_i^*)_i\right)$$.

- If the optimal value of the objective is positive, it means that the subtour constraint is violated for  
  
  $$
  \bar{S}=\{i\in[\![1,n]\!]\ \text{such that}\ z_i^*=1\},
  $$
  
  and the current $$p$$. We can add a new constraint to the master problem: $$\mathcal{C}\leftarrow\mathcal{C}\cup(\bar{S},\bar{p})$$.

- If for all $$p\neq I$$ the optimal value of the subproblem is negative or null then considering $$(\tilde{x}_{ij})_{i,j}$$ **after removing the arc $$(F,I)$$** provides the optimal solution of the initial problem.

> The constraint generation is carried out in **two steps**:
> 1. first step based on the LP relaxation of \eqref{M4} and the LP relaxation of \eqref{S4}
> 2. second step based on \eqref{M4} and the LP relaxation of \eqref{S4} 

<br>

## Performance
---

Method 2 proves to be the most effective method among the ones considered here. It is based on a formulation with a polynomial number of constraints, so we only need to solve a MILP, which is a great advantage. Conversely, the method 1 cannot solve the largest instances in a reasonable amount of time.

Method 4 allows us to solve all the instances as well, but based on an iterative process (constraint generation). Constraints can be generated by solving LPs instead of MILPs, which is an improvement to method 3.

> This variant of the Traveling Salesman Problem shows the **interest of exploring different approaches** when addressing an optimization problem. Here, the least intuitive methods (2 and 4) appear to be the most efficience.

<br>

## Instances
---

| Name | $$n$$ | $$I$$ | $$F$$ | $$k$$ | $$m$$ | $$\Delta$$ | Optimal value | Solution |
|:----:|:-----:|:-----:|:-----:|:-----:|:-----:|:----------:|:-------------:|:--------:|
| $$\texttt{n6}$$ | 6 | 1 | 5 | 4 | 2 | 6 | 12 | ![]({{ site.baseurl }}/images/flight-route-optimizer/n6_cost=12.png) |
| $$\texttt{n20_1}$$ | 20 | 19 | 10 | 8 | 3 | 50 | 91 | ![]({{ site.baseurl }}/images/flight-route-optimizer/n20_1_cost=91.png) |
| $$\texttt{n20_2}$$ | 20 | 1 | 10 | 8 | 3 | 50 | 79 | ![]({{ site.baseurl }}/images/flight-route-optimizer/n20_2_cost=79.png) |
| $$\texttt{n20_3}$$ | 20 | 1 | 10 | 8 | 3 | 25 | 90 | ![]({{ site.baseurl }}/images/flight-route-optimizer/n20_3_cost=90.png) |
| $$\texttt{n20_4}$$ | 20 | 1 | 15 | 8 | 3 | 40 | 96 | ![]({{ site.baseurl }}/images/flight-route-optimizer/n20_4_cost=96.png) |
| $$\texttt{n30}$$ | 30 | 19 | 10 | 14 | 3 | 18 | 130 | ![]({{ site.baseurl }}/images/flight-route-optimizer/n30_cost=130.png) |
| $$\texttt{n40}$$ | 40 | 27 | 8 | 14 | 2 | 26 | 114 | ![]({{ site.baseurl }}/images/flight-route-optimizer/n40_cost=114.png) |
| $$\texttt{n50}$$ | 50 | 49 | 34 | 16 | 4 | 28 | 168 | ![]({{ site.baseurl }}/images/flight-route-optimizer/n50_cost=168.png) |
| $$\texttt{n70}$$ | 70 | 47 | 23 | 22 | 16 | 35 | 235 | ![]({{ site.baseurl }}/images/flight-route-optimizer/n70_cost=235.png) |
| $$\texttt{n80}$$ | 80 | 46 | 32 | 35 | 9 | 31 | 407 | ![]({{ site.baseurl }}/images/flight-route-optimizer/n80_cost=407.png) |
| $$\texttt{n90}$$ | 90 | 1 | 90 | 45 | 7 | 50 | 616 | ![]({{ site.baseurl }}/images/flight-route-optimizer/n90_cost=616.png) |
| $$\texttt{n100_1}$$ | 100 | 1 | 100 | 50 | 11 | 40 | 665 | ![]({{ site.baseurl }}/images/flight-route-optimizer/n100_1_cost=665.png) |
| $$\texttt{n100_2}$$ | 100 | 1 | 100 | 50 | 11 | 40 | 559 | ![]({{ site.baseurl }}/images/flight-route-optimizer/n100_2_cost=559.png) |
| $$\texttt{n120}$$ | 120 | 1 | 120 | 40 | 11 | 30 | 447 | ![]({{ site.baseurl }}/images/flight-route-optimizer/n120_cost=447.png) |
| $$\texttt{n150}$$ | 150 | 1 | 150 | 50 | 11 | 25 | 449 | ![]({{ site.baseurl }}/images/flight-route-optimizer/n150_cost=449.png) |