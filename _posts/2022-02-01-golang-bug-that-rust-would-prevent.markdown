---
layout: post
title:  "A Golang security bug that Rust would have prevented"
date:   2022-02-01 14:21:19 -0400
categories: blockchain
excerpt: hello
---

*Note: This blog originally appeared at NCC Group Crypto Services [blog](https://research.nccgroup.com/2022/07/02/a-deeper-dive-into-cve-2021-39137-a-golang-security-bug-that-rust-would-have-prevented/).* 

This blog post discusses two erroneous computation patterns in Golang. By erroneous computation
we mean simply that given certain input, a computer program with certain state returns incorrect
output or enters an incorrect state. While clearly there are no limits on how erroneous computations
can happen in general, there are language usage patterns which make erroneous computation more
likely. In blockchain, erroneous computation is a problem as the ledger can end up in an unexpected
state or the blockchain may get wedged at a certain corrupt endpoint. In addition, if erroneous computation
happens in only a subset of nodes on the network, a netsplit occurs, which may result in double-spend attacks. 

The first erroneous computation example is CVE-2021-39137 which is an interesting `go-ethereum` bug
identified by Guido Vranken and from which the title of this blog post is derived. The bug caused a
netsplit in the Ethereum network and essentially results from the ability to have a mutable and
non-mutable slice referencing the same chunk of memory. The second erroneous computation pattern
example is extracted from the Go Programming book by Donovan and Kerninghan and concerns deferred
functions' access to parent-scope variables. This pattern could happen in any language that supports
parent-scope variables in a similar way Golang does. 

This blog post was motivated by Cosmos blockchain implementation reviews. Cosmos is a blockchain
building framework which offers an out-of-the-box consensus protocol and an SDK that provides
tools backing common state transition code and, as such aims to be the "Ruby 
on Rails of blockchain". It allows connecting to other Cosmos blockchains via IBC (Inter Blockchain
Communication Protocol) and allows drawing the consensus security from parent chains (Inter-blockchain
security). The developer mainly implements the state-transition logic. It is worth noting that:

* Panics are recovered from by the Cosmos framework and treated as errors. This renders a whole class
of Golang Denial of Service bugs unimportant. This includes eg. `nil` pointer dereferences, out of bounds
array/slice reads and writes, calling methods on `nil` interfaces, etc.
* Often times, Cosmos SDK blockchain apps are fairly simple in terms of the Golang constructs they use.
They may be conservative in that they don't use Go routines and channels, known as another common
source of bugs in Golang. In addition, severity of eventual race conditions may be low due to the fact
that the condition needs to happen on a large portion of nodes at once.

Given the setting described above, other generic erroneous computation patterns in Golang are worth
discussing (unrelated to Denial of Service via panic or Golang's concurrency primitives).  Let's discuss
the first bug, for which we speculate would not happen if Rust was used. The reason is that Rust does
not allow simultaneously having a mutable and an immutable reference pointing to the same memory. 

# CVE-2021-39137: Erroneous computation in `go-ethereum` 

Given a specially crafted contract, `go-ethereum` nodes would fork off the network, as their contract
evaluation result would be different than the rest of the network. Erroneous computation in blockchain
clients is a serious issue, since it results in network forks and can lead to double-spend attacks. 

The [post-mortem](https://github.com/ethereum/go-ethereum/blob/master/docs/postmortems/2021-08-22-split-postmortem.md)
which explains this bug involves several EVM details with which the reader may not be familiar. We start with
a code snippet that removes all the EVM details, is very simple, and makes the bug obvious. It should be noted that the
code below is not present in the `go-ethereum` implementation, it just mimics the essence of the issue when all of the
EVM details are removed:

```golang
func returnAndCopy(mem []byte, n int, copyTo int) []byte {
	// boundary checks omitted

	ret := mem[0:n]

	copy(mem[copyTo:copyTo+n], mem[0:n])    // copy(dst, src)

	return ret
}
```
The `returnAndCopy` function returns the first `n` bytes of a slice. Before that, it copies those
same `n` bytes to a different location in the slice. For example:

```golang
 slice := []int{0,1,2,3,4,5,6,7,8,9}
 fmt.Println(returnAndCopy(slice, 3, 5))    // 0,1,2
```
In this case, the `returnAndCopy` function returns correct values:

```
  0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
  ----------         -----------  
   return  	        copyTo

  0 | 1 | 2 | 3 | 4 | 0 | 1 | 2 | 8 | 9
  ----------         -----------  
   return	        copyTo

  return: 0 | 1 | 2      // correct
```
In this example, the return interval and copy interval do not intersect. If the intervals have shared (common) locations in the array, but do not point to the exact same subarray, `returnAndCopy` fails to produce the correct result:

```golang
 slice := []int{0,1,2,3,4,5,6,7,8,9}
 fmt.Println(returnAndCopy(slice, 3, 1))
```
In more detail:

```
  0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
  ----------         
  return	      
      ----------
        copyTo

  0 | 0 | 1 | 2 | 4 | 5 | 6 | 7 | 8 | 9
  ----------         
  return	      
     -----------
        copyTo

  return: 0 | 0 | 1      // incorrect
```
The issue is that `ret` and `mem` are pointing to the same backing memory. A modification to `mem`
affects the `ret` slice in an unintended way. It is interesting to note that the same bug would likely
be prevented in Rust, as the compiler would prevent having a mutable and non-mutable references to
a same memory location. If that is the case, Rust's memory safety principles would have prevented
this hard fork bug from happening. 

**Mapping to EVM:** CVE-2021-39137 identifies the previously described pattern inside `go-ethereum`'s EVM.
Let's start from looking at the key line of the contract that demonstrated the vulnerability, decompiled
from the `statetest` provided at the end of
[post-mortem](https://github.com/ethereum/go-ethereum/blob/master/docs/postmortems/2021-08-22-split-postmortem.md).

```golang
contract Contract {
    function main() {
        memory[0x00:0x01] = 0x01;
        memory[0x01:0x02] = 0x02;
        memory[0x02:0x03] = 0x03;
        memory[0x03:0x04] = 0x04;
        memory[0x04:0x05] = 0x05;
        memory[0x05:0x06] = 0x06;
        var temp0;
        temp0, memory[0x02:0x08] = address(0x04).call.gas(0x7ef0367e633852132a0ebbf70eb714015dd44bc82e1e55a96ef1389c999c1bca)(memory[0x00:0x06]);     // (1) 
        // [...]
```
After filling up the EVM contract memory with an increasing sequence of bytes, a contract at address `0x04`
is called at the line denoted by (1).

The contract at address `0x04` is a special, "pre-compiled" contract and it is implemented natively in `go-ethereum`.
Pre-compiled contracts implement commonly used functionalities such as hash function computation and elliptic curve
operations. In this case, the `dataCopy` pre-compiled contract is very simple, it implements the "identity" function,
which just returns its one argument:

```golang
func (c *dataCopy) Run(in []byte) ([]byte, error) {
        return in, nil
}
```
Let's next look what happens inside `opCall`,  right after  `dataCopy` is called:

```golang
func opCall(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {

  // ..SNIP..

        ret, returnGas, err := interpreter.evm.Call(scope.Contract, toAddr, args, gas, bigVal)  // (2)

	if err != nil {
                temp.Clear()
        } else {
                temp.SetOne()
        }
        stack.push(&temp)
        if err == nil || err == ErrExecutionReverted {
                //  ret = common.CopyBytes(ret)       //  (3) (FIX)
                scope.Memory.Set(retOffset.Uint64(), retSize.Uint64(), ret)    // (4) overwrite
        }
        scope.Contract.Gas += returnGas

	interpreter.returnData = ret    // (5) returnData set
        return ret, nil
}
```
Relevant to CVE-2021-39137 are the following aspects: 

* In the line denoted with (2), the `ret` value is a slice that points to the first 6 bytes of the contract
memory. That portion of the memory was chosen by the caller in the offending contract (`memory[0x00:0x06]`,
see line (1)). The `dataCopy` contract does not create a new copy of memory before returning. 
* The call to `scope.Memory.Set` copies `ret` to another location, see line (4). The copy target location is
chosen by the caller and in the case of the decompiled contract above, it is `memory[0x02:0x08]`, see line (1).
* Close to the end of the `opCall` implementation at line (5), `interpreter.returnData` is set to be
equal to `ret`, that is, `memory[0x00:0x06]`. In EVM, the interpreter keeps track of "`returnData`": this is
an internal holder opCode that calls other contracts returned. 

The `scope.Memory.Set` line corrupts the memory that `returnData` will point to and `returnData` ends up
being incorrect (`0|1|0|1|2|3` as opposed to `0|1|2|3|4|5`). The fix is to add line (3), which preserves
the original subslice that will be returned before it is overwritten. 

**Note:** A related issue in Golang concerns memory reallocation which happens in the context of the `append` function.
This type of [edge-case behavior](https://www.jjinux.com/2015/05/go-surprising-edge-case-concerning.html) can be more
likely to cause issues as it may evade detection in testing. 

# Parent scope variables in deferred execution

This bug is similar to a general bug pattern where global variables change underneath code
in an unforeseen way. Recall that functions are "first-class values" (pg. 135, sect 5.6 in the Golang book):

```golang
func main() {

	var f func(k int) int

	f = func(n int) int {
		return n*n
	}

        fmt.Println(f(2))      // 4 

}
```
This means that functions can be assigned to variables, passed to and returned from functions.
Functions cannot be compared, and their zero value is `nil`. It is interesting to note
that functions have state. See the following example:

```golang
func createIncrease() func() int {

	x := 0
	increase := func() int {
                x = x + 1
                return x
        }

	return increase
}  // x goes out of scope

func main() {

	f := createIncrease()

        fmt.Println(f())  // 1
        fmt.Println(f())  // 2
}
```
Even though `x` went out of scope (as a local variable inside `createIncrease` function),
`x` still exists as a part of the `f` function's state. We don't see `x` anymore, but it's there.
There is also the question what happens if multiple functions pick up on the same parent scope
variable:

```golang
func createIncreaseAndSquare() (func() int, func() int) {
        x := 2
        increase := func() int {
                x = x + 1
                return x
        }

        square := func() int {
                x = x*x
                return x
        }

	return increase, square
}

func main() {

	// x is created inside createIncrease and goes out of scope
	f, g := createIncreaseAndSquare()

	// x lives on
	fmt.Println(g())  // 4
        fmt.Println(g())  // 16

	// f shares the same x as g
        fmt.Println(f())  // 17
}
```
As can be seen, the parent-scope variable is shared between the two functions. This is similar
to the usage of global variables by which a global variable may change underneath the deferred
function from multiple locations in the code.

```golang
func main() {

        keys := []string{"first", "second", "third"}
        m := make(map[string]bool)
	var cleanupFs []func()

        for _, k := range keys {

		m[k] = true

		// k := k       <-- FIX

		cleanupFs = append(cleanupFs, func() {
                        fmt.Println("deleting ", k)
		        delete(m, k)
	        })
	}

	// ..do something with m..

	// clean up
        for _, cleanup := range cleanupFs {
                cleanup() 
	}
}
```

The map does not properly get cleaned up. The "clean up" for loop only deletes the "third" key from
the map and does so 3 times:

```golang
	// deleting  third
	// deleting  third
	// deleting  third
```

As the loop unwinds, the cleanup functions are picking up the parent scope's `k` variable.
By the end of the for loop, the delayed functions and the parent function share the same `k`,
which is equal to "third".

# Conclusion

Both bug types described in this blog post can roughly be described as variable content unexpectedly
changing underneath code. Whether a particular logical issue of this type is labeled as a security
issue or not is less important. Audits of correctness-critical Golang code such as Cosmos blockchain
code should include checking for these types of issues. 

