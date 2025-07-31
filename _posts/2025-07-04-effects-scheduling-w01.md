---
layout: post
title: "Effects-based scheduling for the OCaml compiler - w01"
date: 2025-07-04 09:00:00 +0100
category: "ocaml-effects-scheduling"
---

This is a series of blog posts documenting my progress for an internship at the University of Cambridge. This project explores the potential of using OCaml's [effect handlers](https://ocaml.org/manual/5.3/effects.html) and [domains](https://ocaml.org/manual/5.3/parallelism.html) in place of the current separate build system (dune, make) to self-schedule compilation of missing dependencies on-the-fly.

My knowledge with functional programming, at this point, basically only came from the [CST 1A Foundations course](https://www.cl.cam.ac.uk/teaching/2425/FoundsCS/). To catch up, much of the first few days were spent studying the [OCaml effect handler](https://ocaml.org/manual/5.3/effects.html), and the rest were spent poking around in the OCaml compiler. Here is what I've picked up so far:

### Continuations

A [first-class](https://en.wikipedia.org/wiki/First-class_citizen) continuation `k`, informally, is a callable that represents "the rest of the computation", held at a given point in execution. In other words, it is a snapshot of the control flow at a given moment. This is made explicit in the [continuation-passing style (CPS)](https://en.wikipedia.org/wiki/Continuation-passing_style) of a program, where control is passed explicitly in the form of continuations `k : 'a -> unit`, where `'a` is the type of an intermediate result:

```ocaml
let eq x y k = k (x = y)
let sub x y k = k (x - y)
let mul x y k = k (x * y)

let rec factorial n k =
  eq n 0 (fun b ->
    if b then
      k 1
    else
      sub n 1 (fun m ->
        factorial m (fun x ->
          mul n x k)))

(* 120 should appear in stdout *)
factorial 5 (fun ret -> Printf.printf "%d\n" ret)
```

This is somewhat analogous to `setjmp`/`longjmp` in C.

(side note: notice that in CPS, all calls must be tail-calls!)

### OCaml algebraic effect handlers

_Delimited continuations_ generalize continuations, in the sense that we now capture the context only up to a delimiter (read: slice of a stack frame). Naturally, unlike continuations, _delimited_ continuations can meaningfully return values, and not just `unit`.

OCaml (algebraic) effect handlers generalize [exception handlers](https://ocaml.org/docs/error-handling), in the sense that the handler is provided with the delimited continuation of the call site, whereas exceptions do not have access to a "continuation mechanism". Here is a nice example, courtesy of [this tutorial](https://github.com/ocaml-multicore/ocaml-effects-tutorial):

```ocaml
type _ Effect.t += Conversion_failure : string -> int Effect.t

let int_of_string l =
  try int_of_string l with
  | Failure _ -> perform (Conversion_failure l)

let rec sum_up acc =
    let l = input_line stdin in
    acc := !acc + int_of_string l;
    sum_up acc

let () =
  let acc = ref 0 in
  match_with sum_up acc
  {
    effc = (fun (type c) (eff: c Effect.t) ->
      match eff with
      | Conversion_failure s ->
        Some (
          fun (k: (c,_) continuation) -> continue k 0
        )
      | _ -> None
    );
    exnc = (function
      | End_of_file -> Printf.printf "Sum is %d\n" !acc
      | e -> raise e
    );
    retc = fun _ -> failwith "impossible"
  }
```

Here, `match_with f v h` runs the computation `f v` in the given handler `h`, and handles the effect `Conversion_failure` when it is invoked (c.f. `try`/`with`).

Effects are performed (invoked) with the `perform : 'a Effect.t -> 'a` primitive (c.f. `raise : exn -> 'a`), which hands over control flow to the corresponding delimiting effect handler, and the continuation `k` is resumed with the `continue : ('a, 'b) continuation -> 'a -> 'b` primitive. (The type `('a, 'b) continuation` can be mentally processed as `'a -> 'b` but used exclusively for effects, as far as I can tell.)

#### ...how is this type-checked?

Effects are declared by adding constructors to an [extensible variant type](https://ocaml.org/manual/5.3/extensiblevariants.html) defined in the `Effect` module. In short, extensible variant types are [variant types](https://dev.realworldocaml.org/variants.html) which can be extended with new variant constructors at runtime , with the `+=` operator. As an aside, this is also how one could extend the built-in exception type `exn`:

```ocaml
type exn += Invalid_argument of string
type exn += Out_of_memory
```

(there is of course the `exception` keyword that one should probably use instead!)

Effects are strongly typed, but the effect handler needs to be able to match against multiple effects at once, and since constructors can be added at runtime, the handler must be generic over every possible effect type (and so we must match against the wildcard `_`). A `None` return value means to exhibit transparent behaviour (ignore the effect), and allow it to be captured by an effect handler lower down the call stack. (OCaml effects are unchecked, i.e.: it is a runtime error if an effect is ultimately not handled.)

The syntax `fun (type c) (eff: c Effect.t) -> ...` makes use of [locally abstract types](https://ocaml.org/manual/5.3/locallyabstract.html). This is required for type inference here, when different branches of the pattern-matching have possibly different `c` (the type of `c` is "locally collapsed" inside a branch when we have a match). It follows that the scope of `c` cannot escape a branch.

While reading on this, I [came](https://stackoverflow.com/questions/69144536/what-is-the-difference-between-a-and-type-a-and-when-to-use-each) [across](https://discuss.ocaml.org/t/locally-abstract-type-polymorphism-and-function-signature/4523) another interesting construct: explicit polymorphism. Turns out, if we write the following in a module interface:

```ocaml
(* foo.mli *)
val foo : 'a * 'b -> 'a
```

This would mean what one would think it means: for all types `'a` and `'b`, `foo` must be able to take in a 2-tuple of type `'a * 'b` and return a result of type `'a`. However, if we instead write the following in a module implementation:

```ocaml
(* bar.ml *)
let bar : 'a * 'b -> 'a = fun (x,y) -> x + y
```

`bar` would have the type signature `int * int -> int`, i.e.: `'a` and `'b` are both refined into `int`. This is because in a module implementation, instead of having implicit universal quantifiers in the type signature as we would normally expect, the type checker interprets this as "there exists types `'a` and `'b` that satisfies the definition".

To force it to take a polymorphic type signature, we declare the polymorphism explicitly, with:

```ocaml
(* bar.ml *)
let bar : 'a 'b. 'a * 'b -> 'a = fun (x,y) -> x + y
(* read: forall types 'a and 'b, ... *)
```

which now fails to compile, as expected.

#### ...surely this has (significant) overhead?

No. (I hope so!)

OCaml delimited continuations are implemented on top of _fibers_: small runtime-managed, heap-allocated, dynamically resized call stacks. If we install two effect handlers (corresponding to the two arrows), just before doing a `perform` in `foo`, we have the following execution stack:

```text
+-----+   +-----+   +-----+
|     |   |     |   |     |
| baz |<--| bar |<--| foo |
|     |   |     |   |     |
+-----+   +-----+   +-----+ <- stack_pointer
```

Suppose that then the effect is performed and being handled in `baz`. We then have the following stack:

```text
+-----+                   +-----+   +-----+
|     |                   |     |   |     |   +-+
| baz |                   | bar |<--| foo |<--|k|
|     |                   |     |   |     |   +-+
+-----+ <- stack_pointer  +-----+   +-----+
```

The delimited continuation `k` here is an object on the heap that corresponds to the suspended computation. When the continuation is resumed, the stack is restored to the previous state. (we can safely do this since continuations are _one-shot_ -- they can only be resumed at most once). Notice that it was not necessary to copy any stack frames in the capture and resumption of a continuation; my guess is that they probably have around the same cost as a normal function call?

### So what is it that I'm doing?

The original project proposal [can be found here](https://anil.recoil.org/ideas/effects-scheduling-ocaml-compiler).

Currently, the compiler is built with an external build system [Make](https://en.wikipedia.org/wiki/Make_(software)). Compilation units naturally form a directed acyclic graph of (immediate) dependencies, and this is generated and saved in a text file `.depend`. In the Makefile, one can add dependencies to build rules, and thus the build system knows to launch a compiler instance for every compilation unit in dependency order.

The project aims to explore the potential of taking that ability away from the build system, and instead get the OCaml compiler to effectively "discover" the dependency order itself, via launching a copy of itself when it discovers that a dependency is missing.

### Progress so far

I have [hoisted](https://github.com/lucasma8795/ocaml/commit/708d64a9b5b650b9208c8da85e5ffdd95e8b7bab) all the logic in `driver/Load_path.ml` up to `main.ml` via effects (performing effects in `Load_path.ml` and installing an effect handler at `main.ml`. The point of this is to get the relevant path resolution logic from being buried deep inside the compiler, to just below surface level.

I have also successfully performed a [bootstrap cycle](https://en.wikipedia.org/wiki/Bootstrapping), where one builds a compiler with a previously stable version of itself.

The logical next step would be to experiment with code that launches a copy of the compiler whenever a dependency has not been compiled, and eventually merge that with my existing code...
