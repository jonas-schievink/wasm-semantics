---
title: 'Verifying Wasm Functions: `$i64.reverse_bytes`'
author:
-   Rikard Hjort
-   Stephen Skeirik
date: \today
header-includes:
-   \usepackage{amssymb}
---

This blog post is the second in a series of posts
written by [Runtime Verification, Inc.](https://www.runtimeverification.com)
and sponsored by [dlab](https://medium.com/dlabvc)
on [web assembly (Wasm)](https://en.wikipedia.org/wiki/WebAssembly),
and more specifically KWasm, a tool for doing
[formal verification](https://en.wikipedia.org/wiki/Formal_verification) of Wasm programs.
If you missed the
[first post](https://medium.com/dlabvc/kwasm-a-new-executable-semantics-for-the-blockchain-14e1bca8a360),
feel free to take a look!

Today, we will be exploring how to verify a fragment of the
[WRC20 contract](https://gist.github.com/axic/16158c5c88fbc7b1d09dfa8c658bc363),
i.e., a Wasm implementation of the [Ethereum](https://en.wikipedia.org/wiki/Ethereum)
smart contract [ERC20](https://eips.ethereum.org/EIPS/eip-20), which provides
an API for token exchange that is usable by other kinds of smart contracts (e.g., digital wallets).
This is relevant because [the Ethereum virtual machine (EVM) is migrating to Wasm implementation
dubbed EWasm](https://www.coindesk.com/open-heart-surgery-inside-ethereums-crucial-replacement-of-the-evm).

In other words, we will show how we can formally verify a fragment of a
program (i.e., the WRC20 smart contract) which is written in web assembly (Wasm).
Our desire is that you will not only enjoy this technical adventure but also walk
away knowing just a bit more about Wasm and how the formal verification process
proceeds in practice.

# A Whirlwind Tour of Formal Verification

Formal verification is a process by which a system (typically, a software system)
is proved to be consistent with a set of formalized requirements.

![Formal Verification Process Overview](media/img/formal-verification-process-diagram.png)

The figure provides an overview of how requirements are turned into working code.
In conventional software development, a developer (and typically also compiler)
together turn system requirements into executable code.
A (possibly different) developer will then write tests that hook into the executable
code, ensuring that certain code paths return desired results.
Here is the million dollar question: can testing ensure a perfect correspondence
between the requirements and the code?
The answer is *simply* no; we know from experience that testing can only demonstrate
the presence of bugs; not their absence.
The beauty of formal verification is that we can *provably* demonstrate an equivalence
between our requirements and the source code via a *semantics*.
You can think of a semantics as a compiler that maps code into its mathematical meaning.

Let's examine the phases of formally verifying a program:

1. Specify Language Semantics - must be done *once* for each programming language that consider (e.g., C, Javascript...) and can be used for all programs in that language;
2. Formalize Program Requirements - convert natural language requirements document into an equivalent mathematical description;
3. Prove Correspondence - use a theorem prover to prove the desired equivalence between the *formalized requirements* and *formalized system specification*.

We are going to illustrate this process concretely, using our tool, the [K Framework](https://github.com/kframework/k).
The K Framework (which we abbreviate as K) can be used to specify language semantics as well as verify programs written in a specified language.

## Specifying Language Semantics

Recall that a semantics is a way of mapping *terms* in a language into their *mathematical meaning*.
In the case of a programming language, the terms of the language are *programs*.
What is the mathematical meaning of a program?
We can understand a program as a function that over *system states*, e.g.
the state of the memory, the hard disk, network, etc...

A K language semantics consists of three parts:

1. a *syntax* for the language (i.e., a parser for programs in the language);
2. a hierarchical XML-like syntax that defines the *system states*, which include the language syntax plus more stuff;
3. and a set of *transition rules* that map systems states to new system states (typically by executing language expressions).

The bulk of the semantics are the transition rules which describe how the state of a system executing a program gradually changes.
For a typical language semantics, an initial system state for a program $P$ is just:

1. default initialized values (e.g., the object `Object` in Java);
2. the text of program $P$ stored in the `<k>` cell (a cell is just a fragment of the system state).

When the program terminates, the program $P$ will have been totally consumed by the transition rules,
and the rest of the state will have changed in some meaningful way.

Here's an example of a simple rule in KWasm.
The Wasm `drop` instruction simply drops the top element on the stack.

```
rule <k> drop => . ... </k>
      <valstack> _ : VALUESTACK => VALUESTACK </valstack>
```

Here's how you read it:

- An XML pair `<name>` and `</name>` introduces a *cell*, i.e., a fragment of the system state;
- Anywhere a double arrow `=>` appears in a cell, it describes how the system state changes when
  the rule is applied---the current state is described by the pattern on the left-hand side---the
  successor state is described by the pattern on the right-hand side;
- The cell `<k>` is distinguished; it contains the program code;
- In this rule, the first element of the `<k>` cell (the next instruction to be executed) is `drop`---the
  ellipses (`...`) refers to the (possibly empty) list of the instructions following `drop`;
- The `<valstack>` cell contains a list of values where `:` is list concatenation;
- In the successor state after the rule is applied, the `drop` instruction is deleted from the program code
  to be executed and the top value in the `<valstack>` cell is also deleted.

## Formalizing Program Requirements

<!--
As has been shown, a $mathbb{K}$ semantics can be read and understood as a computational transition system specifying an interpreter.
But it can also be understood as a logic theory under which we can prove properties about programs.

The set of rewrite rules in KWasm, $\Sigma$, are the axioms of the theory of KWasm transitions, $T$, where $\Sigma \subseteq T$.

The full theory, $T$, consists of all theorems which are provable from the axioms.

To use the $mathbb{K}$ framework for deductive program verification, one writes proof obligations as regular rewrite rules, which the $mathbb{K}$ prover tries to prove or disprove belong to $T$.

An implication (rewrite) is a theorem of $T$ iff all terminating paths starting on the left-hand side eventually reach a state that unifies with the right-hand side.
Take, for example, the following implication:
--->

In K, the task of formalizing program requirements is *also* done via rules of the form `S => S'`
such that:

- `S`  is a pattern that describes a (potentially infinite) set of *initial system states*;
- `S'` is a pattern that describes a (potentially infinite) set of *acceptable final states*.

We want to ensure that *all* initial states `S` end up in *some* acceptable final state `S'`.
Let us see a more concrete example rule:

```
rule <k> foo X Y => Z ... </k>
  requires Z >Int X +Int Y
```

This rule *symbolically* specifies a (possibly infinite) set of program states where the `<k>` cell
contains the program `foo X Y` where `X`, `Y`, and `Z` are integer variables (and all other cells are left unspecified).
The requirement will be satisfied whenever all executions of program `foo X Y` starting in all possible system states either:
1. do not terminate, or;
2. terminate with integer `Z` on top of the `<k>` cell, with everything else that initially followed `foo X Y` (indicated by `...`) left unchanged, such that `Z` is larger sum of `X` and `Y`.

For example, if our programming language semantics contained the following rules, the requirement would be satisfiable:

```
rule [a] <k> foo X Y => foo Y X ... </k>
rule [b] <k> foo X Y => X +Int Y +Int 1 ... </k>
```

Clearly, any path that applies rule `b` would immediately satisfy the program requirement.
Of course, if we apply rule `a` forever, the program will never terminate (so we can ignore such executions).

Interestingly, writing down requirements precisely is often the most challenging part of the verification process.
This is typically because we have not fully considered all possible edge cases (e.g., what happens when $x \leq 0$?)
or because we have assumptions about our system that have often never been written down or even verbalized.

## Proving Correspondence

Here we use the K Framework to *formally prove* the correspondence between our *program meaning* and our *program requirements*.
This last step is surprisingly quite mechanical.
In some sense, you just have to "follow your nose," so to speak, in the sense that:

1. (as we saw above) the formalized program requirements provides:

  * a (possibly infinite) set of *initial program states* and;
  * a (possibly infinite) set of *acceptable final program states*.

2. the semantic definition of the programming language *completely determines* how initial states can evolve into final states.

But, that being the case, how can this formal proof process be challenging?
There are two main ways that things can go wrong:

1. the proof process may take too much time and/or memory;
2. the proof may get stuck by something that could not be proved.

This leads us to primary proof search process that we call here the *verification cycle*:

![Verification Cycle Overview](media/img/verification-cycle-diagram.png)

What this means is: if the proof is taking too long *or* if we get stuck, we will:

1. Examine the proof output term to understand *why we are stuck*
2. Extract any necessary lemmas that will help the stuck proof make progress *or*
   strengthen the proof precondition in order to prune useless paths.
3. Repeat the proof with the simplified claims.

### A Very Simple Proof

As mentioned above, proof obligations in $mathbb{K}$ are specified exactly like regular semantic rules.

Just like in a semantic rules, values referenced by rules mentioned may be symbolic.

A set of these proof obligations is called a *spec*.

Below is a simple spec which asserts that copying the value of a local variable to the stack with `local.get` and then writing that value back to the same variable with `local.set`

1. terminates, as expressed by the whole program rewriting to `.`, and
2. produces no other changes in the state, since there are no other rewrites.

```k
module LOCALS-SPEC
    imports WASM-TEXT
    imports KWASM-LEMMAS

    rule <k> (local.get X:Int) (local.set X:Int) => . ... </k>
         <locals>
           X |-> < ITYPE > VAL
         </locals>

endmodule
```

The program in the `<k>` cell is simple to verify, because during the course of its evaluation, only one semantic rule ever applies at a time.

The prover will first need to apply the statement heating rule followed by parenthesis unpacking

```k
rule <k> (S:Stmt SS) => S ~> SS ... </k>
  requires SS =/=K .Stmts
rule <k> ( PI:PlainInstr ):FoldedInstr => PI ... </k>
```

Now the `<k>` cell of the spec contains

```k
local.get X ~> (local.set X)
```

Next, the rule for `local.get`[^1] applies

```k
rule <k> local.get I:Int => . ... </k>
      <valstack> VALSTACK => VALUE : VALSTACK </valstack>
      <locals> ... I |-> VALUE ... </locals>
```

giving the new configuration (after `.` is removed and parenthesis unpacking)

```k
<k> local.set X:Int ... </k>
<valstack> < ITYPE > VAL : VALSTACK </valstack>
<locals>
  X |-> < ITYPE > VAL
</locals>
```

where `VALSTACK` is whatever the stack contained before.

Lastly, the rule for `local.set`[^2] applies

```k
rule <k> local.set I:Int => . ... </k>
     <valstack> VALUE : VALSTACK => VALSTACK </valstack>
     <locals> ... I |-> (_ => VALUE) ... </locals>
```

giving

```
<k> . ... </k>
<valstack> VALSTACK </valstack>
<locals>
  X |-> < ITYPE > VAL
</locals>
```

which unifies with the right-hand side of the  configuration.

In this simple case we were able to simply state how a program would terminate and leave the state unchanged, and the prover could infer it for us.
Indeed, in making this example, the specification above was written and proved on the first try.
The proving process is often not so straight-forward, however, and may require some tweaking and ingenuity.

### A trickier example

Some proofs require that we further specify our intended semantics and encode the invariants of the transition system.

As an example, we take the exact analogue of our previous proof.
Only this time, instead of modifying local variables we are modifying heap storage.

`#inUnsignedRange` captures the invariants that all integer values, once passed through an `ITYPE.const`, will be represented by their corresponding unsigned value, regardless of signed representation.
I.e., any int value in the state, except at some points in the `<k>` cell, is represented is considered to be in $\mathbb{Z}_{\text{ITYPE}}$, and the representative is chosen to be the value of the class in the range 0 to $2^{|\text{ITYPE}|-1}$.

An invariant the semantics have been designed to maintain (but that we have yet to prove it maintains) is that of `#isByteMap`.

```k
module MEMORY-SPEC
    imports WASM-TEXT
    imports KWASM-LEMMAS

    rule <k> (i32:IValType.store (i32.const ADDR) (i32.load (i32.const ADDR)):Instr):Instr => . ... </k>
         <curModIdx> CUR </curModIdx>
         <moduleInst>
           <modIdx> CUR </modIdx>
           <memAddrs> 0 |-> MEMADDR </memAddrs>
           ...
         </moduleInst>
         <memInst>
           <mAddr> MEMADDR </mAddr>
           <msize> SIZE </msize>
           <mdata> BM   </mdata>
           ...
         </memInst>
       requires #chop(<i32> ADDR) ==K <i32> EA
        andBool EA +Int #numBytes(i32) <=Int SIZE *Int #pageSize()
        andBool #isByteMap(BM)

endmodule
```

However, the verification for this spec will fail.

The reason is that storing to and reading from memory is more complicated than storing local values.

When a value is stored to memory it is getting spliced into bytes.
When a value is read from memory, the bytes are assembled into an integer value.

Conceptually, the load will put on the stack the following:

$$
val = bm[addr] + (bm[addr + 1] + (bm[addr + 2] + bm[addr + 3] * 256) * 256) * 256
$$

The store operation takes the value off the stack, and conceptually stores the following sequence of bytes:

----------------- ---------------- -------
  bm[addr]     := val              mod 256
  bm[addr + 1] := (val / 256)      mod 256
  bm[addr + 2] := (val / $256^2$)  mod 256
  bm[addr + 3] := (val / $256^3$)  mod 256
----------------- ---------------- -------

If we plug $val$ into the above equation it becomes clear that the modulus and division operators will cancel out exactly so all we are doing is writing the values in each address back.

This type of reasoning presents a challenge for the $mathbb{K}$ prover using the current semantics.
The semantics uses pure helper functions, `#setRange` and `#getRange` for writing to and reading from the byte map.
These functions expand to a series of `#set` and `#get`, that do the obvious[^3].

However, Z3 can not reason about these functions in the way we would like without giving full definitions in Z3 of the functions themselves.
Since the getting and setting happens at the $mathbb{K}$ level while the arithmetic reasoning happens at the Z3 level, we are stuck.
We can remedy this by either extending Z3's reasoning capabilities, or $mathbb{K}$'s.

In this case, we chose to extend $mathbb{K}$.
This simple case could be handled by just adding a high-level trusted lemma:

```k
rule #setRange(BM, EA, #getRange(BM, EA, WIDTH), WIDTH) => BM
```

Indeed, we will resort to this later, when dealing with a symbolic type rather than `i32` or `i64`.

We add the lemmas the following lemmas, which should obviously hold in integer and modular arithmetic[^4].

```k
rule (X *Int N +Int Y) modInt N => Y modInt N
rule (Y +Int X *Int N) modInt N => Y modInt N

rule 0 +Int X => X
rule X +Int 0 => X

rule (Y +Int X *Int N) /Int N => (Y /Int N) +Int X
```

Together, they help eliminate the expressions for assignment to

----------------  ------------  --------  -------------------------------  -------
     bm[addr] :=  bm[addr]                mod 256
 bm[addr + 1] :=  bm[addr]      /256      mod 256 + bm[addr + 1]           mod 256
 bm[addr + 2] :=  bm[addr]      /$256^2$  mod 256 + bm[addr + 1] /256      mod 256
               +  bm[addr + 2]            mod 256
 bm[addr + 3] :=  bm[addr]      /$256^3$  mod 256 + bm[addr + 1] /$256^2$  mod 256
               +  bm[addr + 2]  /256      mod 256 + bm[addr + 3]           mod 256
----------------  ------------  --------  -------------------------------  --------

We can now make use of the invariant that we claim to maintain for the byte map.
We add the following two lemmas:

```k
rule #get(BMAP, IDX) modInt 256 => #get(BMAP, IDX)
  requires #isByteMap(BMAP)
rule #get(BMAP, IDX) /Int   256 => 0
  requires #isByteMap(BMAP)
```

They state that as long as a byte maintains its intended invariant -- that all values are integers from 0 to 255, inclusive -- we may discard the modulus on the values, and the division amount to zeroing them.
The lemma in itself is self-evident since it assumes the byte map maintains the invariant.
The claim that our semantics maintain this invariant is, at present, a conjecture.

With all these lemmas added to the theory, the proof goes through.

#### Using a Symbolic Type

If we want to make a proof that uses a symbolic type, rather than `i32` or `i64`, matters become more complicated.
Without knowing the type, `#setRange` and `#getRange` will receive a symbolic `WIDTH` argument, and not be able to expand.

To make a proof like that go through, we introduce the higher level idempotence lemma from before:

```k
rule #setRange(BM, EA, #getRange(BM, EA, WIDTH), WIDTH) => BM
```

But rather than including it in the set of manual axioms for all our verification, we can apply it locally, only where it is needed.

```k
require "kwasm-lemmas.k"

module MEMORY-SYMBOLIC-TYPE-LEMMAS
  imports KWASM-LEMMAS

  rule #setRange(BM, EA, #getRange(BM, EA, WIDTH), WIDTH) => BM

endmodule

module MEMORY-SYMBOLIC-TYPE-SPEC
    imports WASM-TEXT
    imports MEMORY-SYMBOLIC-TYPE-LEMMAS

    rule <k> (ITYPE:IValType.store (i32.const ADDR) (ITYPE.load (i32.const ADDR)):Instr):Instr => . ... </k>

...
```

By invoking the $mathbb{K}$ prover with the option `--def-module MEMORY-SYMBOLIC-TYPE-LEMMAS` instead of the usual `--def-module KWASM-LEMMAS`, the prover will accept this new lemma in to its axioms, and the proof will go through.

[^1]: The rule is paraphrased here, it actually is slightly more complex to deal with identifiers.
[^2]: Again, paraphrased.
[^3]: Actually, there is one non-obvious part of each function: when the stored value is 0, that is represented by no entry.
      The two functions respect that by erasing 0-valued entries and interpreting an empty entry as 0, respectively.
[^4]: N.B. that we cannot use the distributive property of division, as that holds over the rational numbers and its supersets, not over the integers.

# WRC20 Verification: first step

WRC20 is a Wasm version of an ERC20.
The WRC20 program can be found [here](https://gist.github.com/poemm/823566e89def007daa5d97ac5bd14419).
It's simpler than an ERC20: it only has a function for transferring (the caller's) funds to another address, and a function for querying the balance of any address.
Keep in mind also that Ewasm is part of Ethereum 2.0, phase 2.
It's still a work in progress, so exactly what Ewasm will look like is unclear.
This is based on an early work-in-progress specification of Ewasm.

In the end, we want to verify the behavior of the two external functions, `transfer` and `getBalance`.
To do so we will need to introduce K rules about the Ethereum state.
That's a topic for the next blog post.
This time, we will focus on a helper function: `$i64.reverse_bytes`.
This function does a simple thing: it takes an `i64` as a parameter and returns the `i64` you get by reversing the order of the bytes.

```
x_0 x_1 x_2 x_3 x_4 x_5 x_6 x_7
```

becomes

```
x_7 x_6 x_5 x_4 x_3 x_2 x_1 x_0
```

What's the point of this?
Wasm stores numbers to memory as *little endian*, meaning the least significant byte goes in the lowest memory slot.
So for example, storing the `i64` value `1` to memory address 0 means that the memory looks like this

```
Address: 0        1        2        3        4        5        6        7
Value:   00000001 00000000 00000000 00000000 00000000 00000000 00000000 00000000
```

Ewasm call data is by convention interpreted as *big endian*, meaning the bytes are in the opposite order.
So the contract must reverse all integer balances: in `transfer`, it must reverse the amount to be sent, and in `getBalance` it must reverse the result before returning.

## `$i64.revserse_bytes` Source Code

### Wasm Text Format

Here's how our function looks like when rendered using the Wasm text format.

```
(func $i64.reverse_bytes (param i64) (result i64) (local i64) (local i64)
  block
    loop
      local.get 1
      i64.const 8
      i64.ge_u
      br_if 1
      local.get 0
      i64.const 56
      local.get 1
      i64.const 8
      i64.mul
      i64.sub
      i64.shl
      i64.const 56
      i64.shr_u
      i64.const 56
      local.get 1
      i64.const 8
      i64.mul
      i64.sub
      i64.shl
      local.get 2
      i64.add
      local.set 2
      local.get 1
      i64.const 1
      i64.add
      local.set 1
      br 0
    end
  end
  local.get 2
)
```

Recall that Wasm instructions are for a stack machine.
The operations in the function above are manipulating a stack and a set of local variables.
The return value of the function is the last remaining value on the stack.
Like other assembly languages, because Wasm is low-level, it can be quite hard to read.
Let's try to improve the situation by rewriting this program in an easier to read format.

<!--
### Wasm Text Format (Folded)

For that reason, the Wasm text format specification provides a folded variant that allows
expressions to be written in a simplified Lisp-like syntax, where an $n$-ary stack
operation `f`---written postfix `x1 ... xn f` in stack-based languages like Wasm---can be written
as `(f x1 ... xn)`.

Let's rewrite our function using the folded instruction notation and see if things become clearer:

```
(func $i64.reverse_bytes (param i64) (result i64) (local i64) (local i64)
  block
    loop
      (br_if 1 (i64.ge_u (local.get 1) (i64.const 8)))
      (local.set 2
        (i64.add
          (i64.shl
	    (i64.shr_u
	      (i64.shl
	        (local.get 0)
	        (i64.sub (i64.const 56) (i64.mul (local.get 1) (i64.const 8))))
	      (i64.const 56))
            (i64.sub (i64.const 56) (i64.mul (local.get 1) (i64.const 8))))
          (local.get 2)))
      (local.set 1 (i64.add (local.get 1) (i64.const 1)))
      br 0
    end
  end
  local.get 2
)
```

The prefix form, when combined with nested parens, allows for the operator arguments to be easily disambiguated.
However, the disambiguated expressions still seem quite complex.
-->

### Wasm Pseudo-code

Notice that the sub-expression `(i64.const 56) (local.get 1) (i64.const 8) i64.mul i64.sub i64.shl`
(with added parentheses for disambiguation) actually appears twice.
Unfortunately, without changing the Wasm semantics or function declaration, we cannot simplify out this expression
to a local, temporary variable. However, for the purposes of understanding how this function works, we would prefer
to use a pseudo-code representation anyway. As opposed to jumping through a bunch of intermediate states as each
feature is "compiled" into a nicer syntax, we will do the entire pseudo-code compilation at once. What we will do:

1. update the function definition header to separate formal parameters from local variable declarations
2. explicitly zero-initialize our local variables (this is implicit in the Wasm semantics)
3. replace `local.get n` and `local.set n x` syntax with `local[n]` and `local[n] = x` explicitly
4. compute the repeated sub-expression only once and assign it to an (imaginary) temporary variable
5. replace the prefix notation with standard infix operation names and Python-like control flow constructions
5. remove all redundant type information (we are always working with unsigned 64-bit integers)
7. explicitly return the result using the `return` keyword

And the result is:

```
(func $i64.reverse_bytes(i64 input)
  i64 local[1] = 0
  i64 local[2] = 0
  while True:
    if local[1] >= 8:
      break
    bits = 56 - (local[1] * 8)
    local[2] = local[2] + (((input << bits) >> 56) << bits)
    local[1] = local[1] + 1
  return local[2]
)
```

This is *much* more readable.
As is usual, operations like `+` and `*` can overflow; if they do, the result is undefined.
The `>>` and `<<` bit-shifting operations operate just as in other languages,
unless the shift width is greater than or equal to the operand size, i.e., here 64, in which case, the shift width modulo 64 would be used.
In this program, we can prove that no operator can ever overflow (because the shift widths are all provably less than `64`) and because
adding any unsigned number to 0 cannot overflow.

In this notation, we see that we are copying the bytes from the `input` variable to our output `local[2]`, one byte at a time.
The bytes of `input` are successively extracted by the incantation `((input << bits) >> 56)` which zeros out all bits of `input` except
for the `local[1]`th byte (which is now shifted all the way to right).
These extracted bytes are then successively added to `local[2]` in reverse order, by first left-shifting by `bits`.

Finally, remember that pseudo-code is *just for the reader*; the formal verification process only concerns real code.

## The proof obligation

Without further ado, here is what we are going to try to prove[^5]:

```
    rule <k> #wrc20ReverseBytes // A macro that expands to the function source code shown above.
          ~> (i32.const ADDR)
             (i32.const ADDR)
             (i64.load)
             (invoke NEXTADDR) // Invoke is an internal Wasm command, similar to `call`.
             (i64.store)
          => .
             ...
        </k>
        <curModIdx> CUR </curModIdx>
        <moduleInst>
          <modIdx> CUR </modIdx>
          <memAddrs> 0 |-> MEMADDR </memAddrs>
          <types> TYPES => _ </types>                                     /* These five state changes  */
          <nextTypeIdx> NEXTTYPEIDX => NEXTTYPEIDX +Int 1 </nextTypeIdx>  /* are due to the fact that  */
          <funcIds> _ => _ </funcIds>                                     /* we declare a new function */
          <funcAddrs> _ => _ </funcAddrs>                                 /* in the first step of the  */
          <nextFuncIdx> NEXTFUNCIDX => NEXTFUNCIDX +Int 1 </nextFuncIdx>  /* specification.            */
          ...
        </moduleInst>
        <funcs> _ => _ </funcs>                                      /* So is this change. */
        <nextFuncAddr> NEXTADDR => NEXTADDR +Int 1 </nextFuncAddr>   /* And this one.      */
        <memInst>
          <mAddr> MEMADDR  </mAddr>
          <msize> SIZE     </msize>
          <mdata> BM => BM' </mdata>
          ...
        </memInst>
      requires notBool unnameFuncType(asFuncType(#wrc20ReverseBytesTypeDecls)) in values(TYPES)
       andBool #isByteMap(BM)
       andBool #inUnsignedRange(i64, X)
       andBool #inUnsignedRange(i32, ADDR)
       andBool ADDR +Int #numBytes(i64) <=Int SIZE *Int #pageSize()
      ensures  #get(BM, ADDR +Int 0) ==Int #get(BM', ADDR +Int 7 )
       andBool #get(BM, ADDR +Int 1) ==Int #get(BM', ADDR +Int 6 )
       andBool #get(BM, ADDR +Int 2) ==Int #get(BM', ADDR +Int 5 )
       andBool #get(BM, ADDR +Int 3) ==Int #get(BM', ADDR +Int 4 )
       andBool #get(BM, ADDR +Int 4) ==Int #get(BM', ADDR +Int 3 )
       andBool #get(BM, ADDR +Int 5) ==Int #get(BM', ADDR +Int 2 )
       andBool #get(BM, ADDR +Int 6) ==Int #get(BM', ADDR +Int 1 )
       andBool #get(BM, ADDR +Int 7) ==Int #get(BM', ADDR +Int 0 )
```

The interesting parts are:

- the `<k>` cell,
- the `<memInst>` cell group, and
- the pre- and postconditions, `requires` and `ensures`.

[^5]: We should note that this is slightly different from the lemma we will need in the end when verifying the `transfer` and `getBalance` functions.
  To make this spec useful, we need to make sure the starting state of the spec matches the state of the program when `transfer` and `getBalance` calls `$i64.reverse_bytes`.
  If we do that, the prover will be able to go: "Aha! This state corresponds exactly to something I've proven, I can just jump to the conclusion!"
  But the above version makes for nice presentation.

### The `<k>` cell

Here we first declare the function, which we have saved, in pre-parsed form, as a macro.
This will store the function in the state, which means several updates will happen.
A new type and a new function address pointer get added to the module instance, and a new function get added to the world of functions (`<funcs>`).
After that (remember, `~>` should be read as "and then"), we run a few Wasm instructions that load 8 bytes from memory, invokes the `i64.reverese_bytes` function, and stores the result back to the same address.

### The `<memInst>` cell

The `<memInst>` cell simply states that there is a memory with address `MEMADDR`, the same as int `<memAddrs>` in `<moduleInst>`, which makes this the memory belonging to the module we are currently executing in.
We also state that the memory gets rewritten, from `BM` to `BM'`.
Every part of the state that we do not state get rewritten will be assumed to stay the same.
So if we omitted this rewrite, the proof obligation would be stating that the memory doesn't change at all.
We also take the `SIZE` of the memory in account, given in number of pages (each page is 64 KiB).

### The pre- and postconditions

The `requires` and `ensures` sections say what we assume to be true when at the outset of the proof and what we need to prove at the end of the proof.
Note that some pre- and postconditions are expressed in the rewrite rules themselves, such as the value in `<memAddrs>` of the current module matching the `<mAddr>` of the `<memInst>` that gets changed (precondition) and that the program in the `<k>` gets consumed (postcondition).
The `requires` and `ensures` clauses are simply for stating facts that we can't express directly in the rewrite.

The first 4 requirements are really boilerplate relevant to the technicalities of the semantics.
The first states that the type of the `i64.reverse_bytes` function has not already been declared.[^6]
The second, third and fourth rules all make sure that integers, whether constants or stored in the byte map, are in the allowed range.
Without these assumptions the prover assumes the values are unbounded integers.

The final clause in the `requires` section states that our memory accesses are within bounds. This is why we need the `SIZE` of the memory.
A separate (but less interesting) proof would show that the function causes a `trap` if this precondition is violated.

The `ensures` section is straightforward.
We are simply asking the prover to ensure that the final memory has correctly flipped the bytes.

[^6]: This is a somewhat arbitrary choice.
  There is a semantic rule which declares the function type if it is not already present.
  There are some technicalities associated with declaring and looking up functions.
  By letting the prover go through those steps, it can construct the state of the `TYPES` the way the semantics specifies.
  This way, the proof becomes more robust (and readable) than if we wrote out the expected state of the types directly in the proof.

## Helping the prover along

Simply stating the above proof obligation and giving it to the prover will result in inconclusive results.
The prover will fail without having proven or disproven the claim.

One reason for this failure is that under the hood, there is a good deal of modular arithmetic going on.
This happens when we transition from the bytes in memory to integer values, and back.
K does not (yet) have much support for reasoning about modular arithmetic.
This presents an excellent opportunity to add that support.
We will extend the set of axioms K knows, triple checking each (so that we don't introduce unsoundness), directed by the places where the prover gets stuck.

### Axiom Engineering: avoiding infinite rewrites

The new axioms need to be designed with care.
Apart from making sure they are correct, we want to ensure that they are simplifications: that is, they reduce the state somehow.
This can happen by making expressions or values smaller.
The reason is that we want to ensure we cannot get explosive growth.
The prover will look for all axioms it knows and try to apply them.
Therefore, if we were to add statements like `X +Int Y => Y +Int X`, which does not reduce the state, we are in trouble -- even though this is perfectly sound.
Can you guess why?

If we were to add a rule that says `X +Int Y => Y +Int X`, then the prover will apply this rewrite wherever it can, over every addition it sees.
But once it is done, there are still just as many additions in the state, and the prover will again apply this rewrite over all additions it sees.

To avoid this we follow a few general principles when engineering our axioms:

Every rule must do at least one of the following:

1. Reduce the number of `modInt`, `>>Int` or `<<Int` constructs.
   This may increase the number of `*Int`, `/Int`, `+Int` and `-Int` (arithmetic) constructs, and increase the values of integers.
2. Make the expression strictly smaller.
3. Make some integer in the expression strictly closer to 0 without making the expression larger or making some other integer farther from 0.

With these rules we can avoid infinite rewrites.
The only way for the expressions size or integer sizes to grow is by removing a `modInt`, `<<Int` or `>>Int`, and there is no way to introduce more of these.
Therefore, the first rule can apply only a finite number of times.
This puts a hard upper limit on how large any given expression can grow through axiom application.

The second rule can also only be applied a finite number of times, since the expression sizes are bounded from above.

The third rule can also not apply infinitely often since there
can only be a finite number of integers in a finite expression, and once these all go to 0, the rule can no longer apply.

This kind of careful engineering is necessary for axioms we want to add to K's reasoning capabilities for all programs.
For verifying specific languages or programs we may get away with being a little less rigor.
However, it is good practice to try to ensure that your axioms are causing your prover to loop forever.

### Adding new axioms

When we conduct the proof without adding anything to the semantics, the prover manages will manage to run the program to termination.
The contents of the `<k>` cell are gone, rewrites have happened in all the expected places.

However, the memory array in `<mdata>`, `BM`, will have been replaced by a behemoth of a symbolic expression:

```
// TODO: Put in once KLab produces it.
```

Now, the rewrite part of our proof obligation states only that `BM` rewrites to *something*.

```
<mdata> BM => BM' </mdata>
```

The crux comes in the `ensures` section.
K cannot immediately see that the postconditions hold.
It is not immediately obvious to that, for example, `#get(BM, ADDR +Int 1) ==Int #get(BM', ADDR +Int 6)`.
K then discharges the proof obligation to Z3 and asks it to prove the postcondition.
But since Z3 does not understand our byte map functions, `#get` and `#set`, nor the bit shifts over integers, it doesn't stand a chance.

So it's time for us to start doing axiom engineering.
Our job is to ask ourselves: "What are some true things that the prover doesn't seem to get?" and expand its knowledge base while sticking to the 3 principles we set out for ourselves.

One common type of sub-expression is of the form `(Y +Int X *Int M) modInt N`.
That is, a modulus operation over an addition, but with a multiplication on the right.
Furthermore, it is usually the case that `N` is `256`, which is `2^8`, and `M` is some even larger power of 2.
In these cases we can actually get rid of `X *Int M`, because `(a + b) mod n = a` when `b` is a multiple of `n`.[^8]

```
rule (Y +Int X *Int M) modInt N => Y modInt N
  requires M >Int 0
   andBool N >Int 0
   andBool M modInt N ==Int 0
  [simplification]

```

In theory, this could be generalized further by not matching on the multiplication and instead checking if the right side of the addition is 0 when applying `modInt N`.
However, this seems to not be good enough for Z3 to solve the issue in reasonable time, so we go with this more specialized lemma for now.


<!-- TODO: Write about some other axiom (that we can generalize more cleanly). -->

[^8]: We don't include a proof of this here, but there is a formal proof included with the proof in our [lemmas file](https://github.com/kframework/wasm-semantics/blob/master/kwasm-lemmas.md)

# That's it

Hopefully we have given you some insight into the process of formal verification.
As you may have noticed, it is a lot of work to verify something seemingly simple.
This goes to show what our CEO expressed recently on Twitter:

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">1/2 &quot;Formal verification&quot; is now a buzzword in the blockchain, but it will not be done properly unless people understand that it takes *significantly* more work to formally verify a program than to write the program first place. Think 9x more for smart contracts!</p>&mdash; Grigore Rosu (@RosuGrigore) <a href="https://twitter.com/RosuGrigore/status/1134549315828035584?ref_src=twsrc%5Etfw">May 31, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Formal verification takes time.
Luckily, it doesn't have to take this much time and effort in the future.
With the new axioms we added we can do more automatic reasoning about modular arithmetic and bit shifts in general, and of Wasm memory in particular.[^7]
Note that this Wasm semantics project is still in an early phase.
As we write more proofs, the set of axioms will grow, and with it the KWasm prover's reasoning power.

To see the full set of axioms we use in KWasm, see [this file](https://github.com/kframework/wasm-semantics/blob/master/kwasm-lemmas.md) in the GitHub repository.
If you want to use K or KWasm for anything, feel free to open issues or contact us in the open [K chat](https://riot.im/app/#/room/#k:matrix.org).