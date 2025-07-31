---
layout: post
title: "Effects-based scheduling for the OCaml compiler - w04"
date: 2025-07-25 09:00:00 +0100
category: "ocaml-effects-scheduling"
---

Now that I have a working prototype of a linear self-scheduling OCaml compiler, the next step was to dispatch compilation tasks in parallel. My idea was to have some sort of process (domain?) pool to submit compilation tasks to, so I got that done fairly quickly:

```ocaml
type !'a promise
(** Type of a promise, representing an asynchronous return value that will
    eventually be available. *)

val await : 'a promise -> 'a
(** [await p] blocks the calling domain until the promise [p] is resolved,
    returning the value if it was resolved, or re-raising the wrapped exception
    if it was rejected. *)

module Pool : sig
  type t
  (** Type of a thread pool. *)

  val create : int -> t
  (** [create n] creates a thread pool with [n] new domains. *)

  val submit : t -> (unit -> 'a) -> 'a promise
  (** [submit pool task] submits a task to be executed by the thread pool. *)

  val join_and_shutdown : t -> unit
  (** [join_and_shutdown pool] blocks the calling thread until all tasks are
      finished, then closes the thread pool. *)
end
```

Internally, this is done with an array of `Domain.t`, and a thread-safe task queue `(unit -> unit) TSQueue.t`, which was nothing more than a wrapper around `'a Queue.t` from stdlib. I have identical worker loops that sit on each domain, checking the queue for tasks when one completes.

Slight caveat: when a compilation task in the pool is waiting on another dependency to finish compiling, we certainly don't want to block the entire domain that the task sits on. I needed some way to yield control back to the pool, allow other tasks to run on our domain, then *continue* the task at the point the task was *suspended*. (sounds familiar?) This was done with a list of continuations, each paired with a `promise` that signals the dependency's completion. To suspend a task, I simply have it raise an effect.

Back to actual compiler work: [David Allsopp](https://github.com/dra27) (my supervisor!) suggested that for a first prototype of my parallel scheduler, I should start with `Unix.create_process` instead of jumping straight into domains, just to cut down on the mutable compiler global state that I would have to initially deal with. The idea was to only have the main process compile `.ml` files, and have it spawn child processes in parallel to compile missing `.cmi` interfaces; if those missing `.cmi` interfaces have their set of missing dependencies, they are compiled linearly[^1], i.e.: we block until its children are ready. The best way to explain this is with an example:

```ocaml
(* A.ml *)
let foo = 42
let () =
  Printf.printf "foo: %d, bar: %s, sum(baz): %d\n" 
    foo (B.bar) (List.fold_left (+) 0 C.baz)

(* B.ml *)
let bar = "Hello, world!"

(* C.ml *)
let baz = [1; 2; 3; 4; 5]
```

(insert `{A,B,C}.mli` files as appropriate!)

When we invoke our custom `ocamlc` to compile `{A,B,C}.ml` (in this order), what then should happen chronologically is:

{% figure [caption:"Image 1: Effects-based parallel scheduling between compilation of three modules"] %}
![Image 1]({{ "/public/images/effects_scheduling_1.jpeg" | relative_url }})
{% endfigure %}

1. `A.ml` starts compiling. One of its dependencies `C.mli` is missing, which is discovered by our effect handler after an effect is raised somewhere to locate `C.mli` in the load path. We launch a child process to compile `C.mli`, then move on immediately.
2. `B.ml` starts compiling. Its only dependency `B.mli` is missing, so that gets compiled in parallel.
3. `C.ml` starts compiling. Its only dependency `C.mli` is missing, but we already launched a child process to compile it (represented as a dotted line), so we attach the suspended compilation to `C.cmi` and resume it only when it is ready.
4. Suppose `C.cmi` is now ready. We can now resume the compilation of `C.ml` and it should complete successfully, since that was our only dependency.
5. `A.ml` was also waiting on `C.cmi`, so it can also be resumed. It now hits a second missing dependency `B.mli`, which we again compile in parallel.

Steps 6 to 10 follow the same logic, as shown in the diagram above. We fold on the list of implementations `{A,B,C}.ml` until all of them compile successfully. I had most of the code down for this by the end of week.

Finally, I took a couple of hours out of my weekend to make this website! I used [Jekyll](https://jekyllrb.com/), a static site generator, which was surprisingly pleasant to set up and easy to work with. The source code is publicly available on [GitHub](https://github.com/lucasma8795/lucasma8795.github.io).

[^1]: This is actually non-trivial, since the main process wants to launch child processes in parallel, but the child processes want to be linear. I did this by temporarily maintaining two branches of the compiler, one for the main process itself (with all this new fancy parallelism) and one that the main process launches (with our linear compiler from the start of week). I take my existing compiler and install it to some directory, but instead of using its executables directly, I create a new entry point to replace `driver/main.ml` and link against the `.cma` files in the installation to create the parallel compiler. This also doubles as a hack for me to use the `Unix` module in the compiler, since that originally depended on `ocamlc` to be built, which in turn depends on `ocamlcommon.cma`, which likely contains whatever that I need to modify, and I can't have those depend on `Unix`.
