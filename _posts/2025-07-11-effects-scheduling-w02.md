---
layout: post
title: "Effects-based scheduling for the OCaml compiler - w02"
date: 2025-07-11 09:00:00 +0100
category: "ocaml-effects-scheduling"
---

### Auto-compilation of missing dependencies

Hours of refactoring and bug-fixing later, I was able to get the OCaml compiler to invoke itself in another process to compile a missing dependency, then resume the compilation process as usual.

More specifically, consider the two `.ml` files below (and their corresponding `.mli` interface files, omitted):
```ocaml
(* foo.ml *)
let bar = 42

(* program.ml *)
let () = Printf.printf "%d" Foo.bar
```

If we invoke the compiler on `program.ml` without first compiling `foo.ml`, clearly it doesn't work: we are missing a dependency `foo.cmi`. However, if we catch the exception that would've normally been raised by the compiler, in our effect handler:

```ocaml
effc = fun (type c) (eff: c Effect.t) ->
  match eff with
  (* filename -> filename *)
  | Load_path.Find_path fn ->
    Some (fun (k: (c, _) continuation) ->
      try
        Effect.Deep.continue k (find_path fn)
      with Not_found -> begin
        (* missing dependency, we need to compile it
           imitate what find_path would normally return *)
        try
          Effect.Deep.continue k (compile_dependency fn)
        (* source file not found, give up *)
        with Not_found ->
          Effect.Deep.discontinue k Not_found
      end
    )
    
  | ...
```

Invoking `./ocamlrun ./ocamlc -c program.ml -I ./stdlib`, we find a missing dependency, and `compile_dependency: filename -> filename` generates the following (hopefully, portable?) command to compile our dependency `foo.ml` (we inherit the load path from the calling parent):

```
'runtime/ocamlrun' './ocamlc' '-c' 'foo.ml' '-I' './stdlib' '-I' ''
```

...and we then resume compilation for `program.ml` with the `continue` primitive.

Linking the object files together, we then get

```
➜ ocamlrun ocamlc foo.cmo program.cmo -I stdlib -o program
➜ ocamlrun ./program
42
```

as expected.

Using the above, I was then able to trace through the Makefile and build `ocamlcommon.cma` and `ocamlbytecomp.cma`, first by building the required `.cmo` files (in no particular order, missing `.cmi` dependencies are auto-discovered and compiled), then linking the objects in dependency order (which is something I'd hope to be able to relax in the future). With this done, it is only two commands away to produce `ocamlc`, the OCaml bytecode compiler:

```
ocamlrun ocamlc -c driver/main.ml <compiler flags> <load path>
ocamlrun ocamlc ocamlcommon.cma ocamlbytecomp.cma driver/main.cmo -o ocamlc <compiler flags> <load path>
```

A slightly concerning issue that I can see coming: the [original](https://ocaml.org/manual/5.2/api/compilerlibref/Load_path.html) `Load_path` module makes the assumption that the contents of the load path don't change throughout the lifetime of the compiler process, and indeed the relevant pieces of code has been designed around this: file system calls are much much slower than simply reading from memory, and so the compiler reads in the filenames and caches them in memory. However we want newly compiled dependencies to be present in the load path as well to avoid compiling dependencies twice, and so the load path state now needs to be mutable!

The next step would be to work on building the rest of the targets that `make install` requires, more to come on this...