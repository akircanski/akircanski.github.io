---
title: On DAGs and passage of time
---
Computer security people often use the following terms to describe events relevant in a computer breach: "vulnerability chaining", "horizontal/lateral movement", "escalation", etc. If there's some satisfaction in using those terms, that's because they refer to the processes' _internal structure_. This internal structure of the process attackers follow is very intuitive, even if it isn't noted down explicitly. 

We look at what this underlying structure looks like. In general, there are two axioms that shape process leading to a computer breach: (1) the process is _cumulative_ in nature and (2) the attacker's capability undergoes _sudden changes_. 

What mathematical description best describes this process? Think before reading, how would you represent cumulativeness and sudden changes over time. 

### 1. Attackers don't play Mahjong 

In the [Mahjong Solitaire](https://en.wikipedia.org/wiki/Mahjong_solitaire) game, the goal is to identify duplicate pieces on the board, after which they're removed. The process is repeated until there are no pieces on the board. Mahjong can be seen as trimming a DAG. A leaf node of a DAG is removed which "frees up" other nodes, allowing their removal in one of the next steps. 

One could consider that a privilege escalation process encountered in computer system hacking follows this pattern. After all, it _is_ true that the process is tedious and requires lots attention, so that basic thing does check out. One may also consider the hacking process to involve acquiring "gadgets", or "capabilities", similar to piece in the game. 

However, Mohjang Solitaire follows a certain _steady pace_; unlocking Mahjong pieces does not include rapid state changes. This is where the Mahjong Solitaire analogy with the computer hacking fails. 

As mentioned, the privilege escalation process follows a specific dynamic, not followed by some other processes out there. That dynamic has at least the following two properties:

* It's usually _cumulative_ in terms of attacker's capability 
* it includes _sudden, dramatic changes_

### 2. Unexpected state changes

The first thing that comes to mind when representing a process with rapid changes is a transformation, over some state. The mapping is monotone, as in bits only activate, they do not deactivate. Consider the following state trail:

```
time         state
0     -------------------- 
1     x-------------------
2     x-------------------
3     xx------------------
4     xx------------------
5     xx------------------
6     xx------------------
7     xx--------x---------
8     xxxxxxxxxxx---------
9     xxxxxxxxxxx--------x
10    xxxxxxxxxxxxxxxxxxxx
   ...

```
The "avalanche" effect is the rule that a flip in bit k results in bit flips in all bits j < k. Avalanche is present in steps 8 and 10. A computer hack involves _capabilities_; for example the capability to authenticate as a system user or the capability to read an arbitrary file on disk. For a given system, we're looking at a set of capabilities:

```
p1, p2, p3, ... p_n
```
Bit position k corresponds to `p_k` and `-` means that the actor does not yet posess the capability. An attack on say a Windows network progresses with the attacker's capability of running commands in a Docker container (step 1 on the picture), then expanding the capability to running code on a host in the network (2), access to internal services keys repository (step 7) and finally step 10 establishing control at the domain controller of the network, which allows access to all capabilities. 

### 3. Inside the Boolean hypercube

The problem with the previous section is that it requires a `p_i => p_i-1` for all `p_i`. This does not correspond to reality; there may be multiple independent paths to the full capability or privilege set. To represent this, we use a "power set" (a set of all subsets) of the set of capabilities. 

<p align="center"> <img src="other-pics/DAGs/power-set.png" alt="The power set or Boolean cube graph"/></p>
The power set forms a graph in which movement from the bottom towards the top of the graph signifies privilege escalation. For example, a file read on the local system may provide the attacker the ability to read a configuration file and authenticate to related database; this is represented as movement from `a` to `{a,b}`. 

A general note about graphs: each movement the power-set graph is "coordinate-wise" in the sense that from every node it's possible to "choose" a capability and add it to the set. This is is why [Boolean hyper-cubes](https://en.wikipedia.org/wiki/Hypercube_graph) are a good alternative way of representing power sets. Capabilities are identified with positions in an n-tuple and from each capability set. It's possible to independently "unlock" or "lock" any capability from any node in the graph.

### 3. Deleting impossible edges

It should be noted that the possibility of privilege escalation is probabilstic in nature. For example, an XSS payload may or may not execute. A phishing attempt succeeds with a certian probability. An ASLR bypass such as a sprayed heap may yield to a desired outcome, or may not. An LLM may or may not identify an exploit that allows escalation. 

That also means that some privilege escalations are known to be impossible, or simply don't make any sense in reality. A pair of capabilities may refer to completely independent system parts. In practice, these routes will simply be uninhabited over different attack attempts.

Such edges can be removed from the picture. You could say this _simplifies_ the graph, but you could also say deleted edges make the graph more complicated. The latter is probably true, as the graph is not the Boolean hypercube anymore and dependencies are introduced. The corresponding monotone Boolean function is more unpredictable. 

### 4. Quotient of the power set graph

Our previous partitive set graph fails to capture the "avalanche" property. A full (or large) set of capabilities is often times gained suddenly, in one passage. Paradoxically, in order to gain a single capability (such as, read from a database), often times, the attacker gains the full set of capabilities. 

For that, we add some back-propagations to the system. A capability "back-propagates" a number of other capabilities. For example, running code on the host system implies access to all application users, administrators as well as access to secrets in configuraiton files (three capabilities). This is represented as `a => b, a => c, a => d` in the picture above. 

In the partitive set model, back-propagation dictates that nodes should be merged (see the notion of Strongly Connectected Component (SCC) [quotient graph](https://en.wikipedia.org/wiki/Strongly_connected_component)). Since `a => c, d`, each node with just `a` and any subset of `{c, d}` is merged into one node. Another merge that follows from this implication is a node that contains both `a, b` and any subset of `{c, d}`. 

<p align="center"> <img src="other-pics/DAGs/merged.png" alt="Merging nodes to get a more accurate graph"/></p>

If the implication is of the form `X, a => B`, where `X` and  `B` are capability subsets, each subset of X will correpsond to a node with all `a U B` elements. 

### 5. Capabilities are not limited to network vantage points

The good thing about this approach is that it's versatile in the following sense. Traditionally, computer security dealt with notions such as "user access to host X", "foot hold in internal network Y". These notions are intuitive as network-wise proximity maps to physical (metric) proximity when the network is represented as a graph on paper. 

However, notions of capability also express the following:

* The ability to leak i-th bit from j-th round of a cryptographic primitive (or j-th round in a protocol run) vs. the ability to extract a bit of a cryptographic key
* Ability to run code in a browser sandbox on a mobile device vs. ability to run code in userland or kernel

### 6. Conclusion

In computer security breaches, the main currency we're dealing is privilege escalation. Everything else is a non-event. The narration of a computer security breach is a sequence of privilege escalations. 

* (Connected) DAGs have "orientation" and moving from the bottom to the top corresponds to passage of time
* A trail from the bottom to top corresponds to a choice of events that may have taken place
* The more complicated the DAG is, the more subtle event inter-dependencies are 
* Power-sets or Boolean hyper-cubes are heavily restricted DAGs; they allow "freely choosing" events at each step

**The type of real-world progression  we looked at corresponds to a "morphed" partitive set graph (Boolean hypercube or a partial ordering relation)?** [Order theory](https://en.wikipedia.org/wiki/Order_theory) is an area which looks at partial orders systematically; DAGs are wider than that in scope. Either way, order theory and DAGs do not burden themselves with identifying real-world models, they take right off into deriving abstract properties of these structures. 

Thank you for reading!

Caveats: TBD

Acknowledgments: TBD


