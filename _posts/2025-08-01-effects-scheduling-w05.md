---
layout: post
title: "Effects-based scheduling for the OCaml compiler - w05"
date: 2025-08-01 09:00:00 +0100
category: "ocaml-effects-scheduling"
---

I started the week off by fixing my parallel scheduler that I've started writing end of last week. There was this one bug that simply refused to budge, no matter how many things I've thrown at it (you can find the setup from [last week's notes here]({{ "2025/07/25/effects-scheduling-w04.html" | relative_url}})):

```plaintext
>> Fatal error: Cannot find address for: C.baz
Fatal error: exception Misc.Fatal_error
Raised at Custom_ocamlc.handle.(fun) in file "custom_ocamlc.ml", line 365, characters 45-55
Called from Custom_ocamlc in file "custom_ocamlc.ml", line 685, characters 2-12
```

This happened after step 10 of the diagram from last week, during compilation of `A.ml`.

Continuations capture everything on the call stack, but what they don't capture is the *global state* of the compiler. Thankfully, some [people](https://github.com/ocaml/ocaml/pull/9963) over at [Merlin](https://github.com/ocaml/merlin) have already added a module ([Local_store](https://ocaml.org/manual/5.2/api/compilerlibref/Local_store.html)) to the compiler, for them to "snapshot" the global state of the type-checker to move back and forth to type different files. They do this by explicitly registering all global state with `s_ref: 'a -> 'a ref` in place of `ref`, which then registers the reference in a list of global bindings. Before we start any compilation, we call `fresh: unit -> store` once, which *snapshots* the current global state as the "initial state" and returns an opaque `store` type capable of storing a set of global states, initialized to the fresh state. This is then used in `with_store : store -> (unit -> 'a) -> 'a` to restore the global state to the state of the `store` during the run of the function, and saving any changes to the `store`. Subsequent calls to `fresh` will return a fresh `store` with values obtained from the snapshot taken at the first instance of `fresh ()`.

This is huge news, because all the missing dependencies would have already been discovered by the time the file has finished type-checking, so most if not all of the global state has already been registered for us. This is what my scheduler looked like, stripping away all unnecessary details:

```ocaml
let suspended_tasks = Queue.create ()
type _ Effect.t += Load_path : string -> string Effect.t

fresh () |> ignore (* snapshot the initial global state *)

(* start compilation of all .ml files *)
List.iter (fun ml_file ->
  let store = fresh ();
  match with_store store (fun () -> compile ml_file) with
  | () -> () (* file compiled successfully *)
  | effect (Load_path dep), cont -> (* dep will be a .cmi file *)
      begin try
        continue cont (resolve_full_filename dep)
      with Not_found ->
        (* we hit a missing dependency, suspend the task *)
        let full_mli_file = find_interface_source dep in
        let dep = (remove_suffix mli_file ".mli") ^ ".cmi" in
        let pid = compile_process_parallel full_mli_file in
        Queue.add (pid, cont, dep, store) suspended_tasks
      end
) files_to_compile

(* fold on suspended tasks until we are done *)
while not (Queue.is_empty suspended_tasks) do
  let (pid, cont, dep, store) = Queue.take suspended_tasks in
  if process_finished pid then
    (* dependency has finished compiling, we can resume the task *)
    add_to_load_path dep;
    with_store store (fun () -> continue cont dep)
  else
    (* re-add the task to the queue *)
    Queue.add (pid, cont, dep, store) suspended_tasks
done
```

I'm sure this was necessary anyway, but this somehow did not fix the issue! I then spent the good part of two whole days adding print statements all over the type-checker and staring at ridiculously long call stacks, until I came across a fairly innocuous piece of code, in `typing/env.ml`:

```ocaml
let find_same_module id tbl =
  match IdTbl.find_same id tbl with
  | x -> x
  | exception Not_found
    when Ident.persistent id && not (Current_unit.Name.is_ident id) ->
      Mod_persistent
```

At this point I had realized that `B` was being opened successfully in `A`, going through the `Mod_persistent` code path above, but somehow `C` kept on raising `Not_found` here no matter what I did, and this was quite suspicious as their behaviour should be virtually identical. The first predicate in line 5 couldn't have been the issue, so it must have been the second that was failing. `Current_unit.Name` sounds like some mutable global state, and surely something as simple as that that must have been captured by `Local_store`.

It wasn't! So when we resumed compilation of `A` (in step 10), the compiler thinks it's in `C`, and it makes sense that it couldn't find `C`, because it thinks we are already in the module `C`. The fix was:

```diff
- let current_unit : Unit_info.t option ref = ref None
+ let current_unit : Unit_info.t option ref = s_ref None
```

It took me two days to add two characters to the compiler! ([David](https://github.com/dra27) told me that he once took 5 days to fix a GC bug that changed only a couple of characters, so I guess this was bound to happen at some point...)

At this point, the entry point of the compiler was turning into a 800-line monster, so I decided to spend the rest of the week doing refactoring and logging improvements, in preparation of using domains as the next step.
