---
layout: post
title: "Effects-based scheduling for the OCaml compiler - w03"
date: 2025-07-18 09:00:00 +0100
category: "ocaml-effects-scheduling"
---

This week was an extension of last week's work, where I took my success of building `ocamlc` with the modified compiler and started to build the rest of the executables that made up the OCaml installation. Ideally, I want to replicate the behaviour of `make world && make install`, which builds everything necessary for a complete OCaml installation, including the compiler, the standard library, and the tools that come with it (e.g.: `ocamlc`, `ocamlopt`, `ocamldep`, etc.), and installs it in some directory. To make the entire build process reproducible, I made a shell script that does all the above. Since I have the compiler find all the dependencies of the `.ml` files, I can drop all the `.mli` files in the recipe and have it find them on-the-fly. Having to pull out all the relevant parts from the `Makefile` was quite the tedious process, but the end of the week I had it all up and working, and a quick `diff` between a clean OCaml installation and an installation from my script verifies this:

```bash
➜ diff ./Documents/cambridge/urop/ocaml/install ./Github/ocaml/install -qr | grep "Only in"
Only in ./Documents/cambridge/urop/ocaml/install/lib/ocaml/compiler-libs: handler_common.cmi
Only in ./Documents/cambridge/urop/ocaml/install/lib/ocaml/compiler-libs: handler_common.cmt
Only in ./Documents/cambridge/urop/ocaml/install/lib/ocaml/compiler-libs: handler_common.cmti
Only in ./Documents/cambridge/urop/ocaml/install/lib/ocaml/compiler-libs: handler_common.mli
```

`handler_common.ml` is the only new file that I have added to the compiler so far, which installs the effect handler to the entry point of the compiler, so it makes sense that it appears in the diff.
