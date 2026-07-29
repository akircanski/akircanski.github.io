---
title: What SHA-x collision search heuristics look like
---

There was an excursion from the cryptographic hash function research into the area of search heuristics, similar to the [DPLL procedure](https://en.wikipedia.org/wiki/DPLL_algorithm). The interesting thing about this excursion is that the SAT solver applied directly does not work well, whereas the fine-tuned search does work and is useful. Note: it is unlikely that this line of research will break full SHA-2. When we say "works well", we mean it works much better than direct search against round-reduced SHA-2 variants.

In this blog post, we share the flavor of what hash collision search algorithms look like. We intentionally ignore many of the details to make the exposition more straightforward. 

### The (simplified) SHA-2 compression function 

The SHA-2 hash function is a [Merkle-Damgard](https://en.wikipedia.org/wiki/Merkle%E2%80%93Damg%C3%A5rd_construction) construction built off of a [Davis-Meyer](https://en.wikipedia.org/wiki/One-way_compression_function) compression function, which itself is built out of a simple recurrence relation. It's a bit of a matryoshka, but as mentioned, we're leaving many details out, so we'll just leave the "heart" of transformation: the recurrence relation inside the Davis-Meyer construction. 

There's a set of words, which constitute the hash function state. For example, 8 `2^64` words, `x_0`, .. `x_7`) and the input message `m_0`, .. `m_15`. The form of the recurrence (for a 3-word state example): `k: x_k = F(x_k-1, x_k-2, x_k-3, m_k)`. 

```
 ????????     
 ???????? ...
 ????????----+
 ????????----|----m_k
 ????????----|     
 ????????<---F    
 ????????
     ...      
 ????????
 ????????
```
Each row in the diagram represents a round or a step of the compression function (one application of `F`). The `?` sign denotes a bit position that can be either `0` or `1`. If all `?`s are replaced with `0`s and `1`s, then we get a hash function execution trace over a number of rounds (e.g. 80 rounds). `F` consists of operations such as modular addition, XOR, bit rotation as well as logical function such as AND. 

In this case, the output would be last set words (e.g. 8 of them). As for the initial state (`x_0`, .. `x_7`), they can be assumed to be constant (as per the simplifications that are allowed in this blog post - Merkle-Damgard uses the previous block's output as input, but that's not important here). The only user-controlled input is the message `m`. We also omit the fact that the Davis-Meyer construction adds the input to the output of the function, without which we'll survive. 

More details on the message that's being hashed `m`:  the `m_k` value are derived through "message expension": the message `m_0, .. m_15` is expanded to 128 values and dissemenated across the rounds through the `F` function:

`(m_0, .. m_15) -> (m_0, .. m_79)` 

assuming there's 80 compression function steps. These are the values that fed into `F` in each step. 

Note: The diagram above is a very general one. We have a stream of bits flowing according to some rule, which is deterministic in this case. Optionally, the flow can be influence from the outside (by message bits). It is hard to imagine which process does _not_ follow this pattern and in that sense this research clearly generalizes out of hash functions.

### Why finding collisions is hard

The most obvious reason is: the computation follows the "commit early, compute later" pattern. **You commit to the message you control, but then the process follows with a bunch of rounds and you end up with an unexpected result, loosing control over what happens.**

In order to compute the function from the beginning through the end, you're forced to choose  `m_0, .. m_15` and then loose control over the remaining steps. It is similar if one starts from the middle, or from the end.  As mentioned, the caller has to commit early, but only then the majority of computation takes place, on which the caller has no direct influence. Exmaple:
```
 00000000----+                 00000000---+               
 00000000----|<--00000000      00000000---|<--00000001
 00000000----|                 00000000---|
 01001001<---F                 01101101---F 
 ????????                      ????????
     ...                           ...
 ????????                      ????????
 ????????                      ????????
```
Recall that we mentioned that the initial recurrence relation fields are fixed, say (as an example) `x_0=x_1=x_2=0`. The LSB of the first message word that goes into `F` is *active*: in the run on the left, the LSB of `m_0` is `0` and on the right it is set to `1`.
The active bit gets dissemenated throughout the trace (not shown on the picture). The active bit positions will soon take over the state and measure a half of the real estate inside the function. 

In order to find a collision, the attacker needs to *cancel out* differences that are quickly dispersing throughout the state, using their control of the message `m`. This is trivial to do in the first 16 rounds, because the attacker has full liberty to choose pairs of `m_0, .. m_15` as they wish. However, after round 16, the _reuse_ of message material the solver already commited to takes place.

The original famous work by Wang et al on MD4, MD5 and SHA-1 exploited **certain synchronicity between the state expansion (the `x_i` registers) and the message expansion (the `m_i` words)**. It was possible to confine activated bits in the state to a single, repating by canceling out "unwanted" differences using the in the message expansion. The cancellation throughout all rounds was possible, as the two patterns would repeat and cancel each other (one in the state expansion and the other one in the message expansion). This is exactly what one does _not_ want from the recurrence relation and the message expansion. 


### Reasoning about differential trails

Let's omit even more details and replace the compression function with just modular addition with 8-bit words, `z = x + y mod 2^8`. Choose a random values x and y and derive z. Now flip a bit in x and consider how the difference propagates in z:

```
   ----x-- x
   ------- y
   ---xx-- z = x + y
```
This picture determines all specific triplets `(x,y,z)` and `(x', y', z')`, such that `x XOR x' = 0000100 = 4 `and `z XOR z' = 00001100 = 2^4 + 2^8`. It is a correct differential trail over modular addition; the difference from one bit in `x` to two bits in `z`, because of the carry bit at that position. 

An example of an impossible differential trail would be the following (no solutions):

```
   ----x-- x
   ------- y
   --x-x-- z
```
Our grammar for specifying differential trails includes only `-` and `x` symbols, but we can conveniently add the `?` sign as well, which specifies that there's no known relation between bits on that specific position. We can now derive conclusions about differential trails, such as:

```
   ----x--- x                             ----x--- x
   -------- y     ->  propagation ->      -------- y
   --x??--- z                             --xxx--- z
```
The two bit positions are unspecified, however, they can be deduced via _difference propagation rules_: the two symbols have to be `x`, as that's the only way bit 5 in `z` can be active (be an `x`).

**Importantly, the notation introduced in this section roughly corresponds to what any human would naturally do when studying difference propagations.** The human would think in terms of "there's a difference in position 5, to which other positions is it going to propagate in the hash function trace?". Or, "under what conditions will there be a difference in bit X if a difference was injected in past rounds in position 5".

### Local collision search

Here's an example of a differential trail search problem from an [important paper](https://csrc.nist.rip/groups/ST/hash/documents/RECHB_FindingSHA1Characteristics_NIST.pdf) on the topic. 

<p align="center"> <img src="other-pics/quotient/single-bit.png" alt="First Image" width="70%"> </p>

In the light of previous sections, we can understand what's going on in the picture:

* We're looking at the first 19 steps of SHA1, the first column (`i`) denotes the step number. The problem is localized in the first 19 steps, the rest of the steps are not shown on the picture
* In the second column, the recurrent relation state is left unspecified (the question marks). Past step 12, a small number of active bits is set
* The right-hand side of the picture has the message expansion differences, there's no question marks; this is because the message expansion is optimized to minimize the number of active bits in _later steps_, thus prolonging non-colliding words as far into the rounds as possible (this is a reduced-round attack and the goal is to extend it as far into the rounds as possible)
* The primary goal of this search instance is to end up with the differential trail as specified in the picture  in steps 12-19
* The gadget is prepared to result in positive differential trail outcome past step 19, but it is unclear whether the trail can be satisfied or not
* If the search algorithm finds the solution, will be used as a "gadget" to be connected with other parts of the attack. 

Note: the modular addition carry state is left out. The carry states are tracked by an additional diagram equally important to the shown one. There should be an additional diagram to this one, however, hash function cryptanalysis papers usually omit the carry propagation diagrams. Possible carry states for each bit position at each step are represented as graphs. We won't go into details on that. 

The search runs as follows, see this [paper](https://csrc.nist.rip/groups/ST/hash/documents/RECHB_FindingSHA1Characteristics_NIST.pdf) on the topic (many papers followed): 

* Choose a random question mark in the state
* Replace it with `-`, or, with less probability, with `x`
* Propagate the results and prune the carry graphs 
* Repeat; if a contradiction is reached, backtrack

The search procedure reminds of a DPLL SAT solver. However, it does not work on 0 and 1 truth values. The propagation rules that involve symbols such as `-` and `x` are pre-computed in huge tables stored in computer memory.  Other differences with DPLL:

* The search is geared towards finding "sparse" differential paths; it makes `-` guesses more often than `x`
* It is unsophisticated in that it does not contain any smart or optimized backtracking, results caching, etc

### Differences and similarities with DPLL

**The main finding of research  is that the differential trail search algorithm performs better than the "direct" SAT methodology.**
Compare the previous algorithm with the following "direct" DPLL method for finding local collisions:

* Encode the two hash function executions as a logical formula
* Add the constraints from the diagram above 
* Run a state of the art SAT solver to find satisfying inputs and outputs 

Finally: the differential trail search heuristic can be seen as a "divide and conquer" approach, as the problem image is first "blurred" - abstration is used to remove details from the picture. Once the "blurred" view of the problem is solved, an actual solution is derived from the "blurred" solution using e.g. a SAT solver. 




