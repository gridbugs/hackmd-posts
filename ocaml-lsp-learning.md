# Getting the OCaml Language Server Running in a Weird Place

My goal is to get [OCaml-LSP](https://github.com/ocaml/ocaml-lsp) - the
[Language Server Protocol](https://microsoft.github.io/language-server-protocol/)
implementation for OCaml - running on a project that builds in a weird way. For
my project I want to have some OCaml files in a directory with no `dune` file
and then run a script that copies them into a transient, generated Dune project
before building _that_ project with dune. OCaml-LSP works out of the box in dune
projects (as long as you've run `dune build` at least once) but in my weird
experimental setup it does not.

This post will describe the process of getting OCaml-LSP working in my weird
project. I'll be using neovim with the
[LanguageClient-neovim](https://github.com/autozimu/LanguageClient-neovim)
plugin as my editor.

## Out of the Box

From an Opam [switch](https://opam.ocaml.org/doc/Manual.html#Switches)
with the `ocaml-lsp-server` package installed I can jump to
the definition of terms from the prelude but not from any libraries
installed with Opam, and I get `Unbound module` errors whenever I try referring
to code from a library. When I run the script that generates a Dune project out
of my OCaml source files I can edit the copies of the source files in the Dune
project and OCaml-LSP works as expected. _But_ if I remove the `_build`
directory from that dune project then OCaml-LSP behaves exactly the same as in
my non-dune project - it understands code from the prelude but can't see any
external libraries.

So the question is which files from `_build` are important for letting OCaml-LSP
find external libraries?

To get some more information about how OCaml-LSP works I enabled logging in my
language server client. In LanguageClient-neovim I added the following to my
`~/.config/nvim/init.vim`:
```
let g:LanguageClient_loggingFile = expand('~/.vim/LanguageClient.log')
```

Then I watched that file as I tried to navigate to the definition of a symbol
that OCaml-LSP didn't know about. This error from the log caught my eye:
```
'Locate' query to merlin returned error: not in environment: Toml.Types.Table.Key.to_string
```
(`Toml.Types.Table.Key.to_string` was the function whose definition I was trying
to jump to.)

I don't know much about `merlin`. It's the OCaml code analyzer used internally
by OCaml-LSP.

## Merlin

Merlin's [website](https://ocaml.github.io/merlin/) links to [docs for manual
configuration](https://github.com/ocaml/merlin/wiki/Project-configuration). It
assumes dune by default but you can write a config file if you're using a
different build system. Reading the docs and following their instructions didn't
seem to have any effect on OCaml-LSP's ability to find external libraries.

However I did happen to notice that the `_build` directory created by Dune
contains a file `_build/default/bin/my-project/.merlin-conf/exe-main` which
contains binary data but also strings that seem to correspond to the locations
of external libraries.

```
$ cat _build/default/bin/my-projct/.merlin-conf/exe-main
DUNE-merlin-confv4:C    "/Users/s/src/my-projct/_opam/lib/ocaml@    $/Users/s/src/my-projct/_opam/lib/ISO8601@A "/Users/s/src/my-projct/_opam/lib/csexp@      /Users/s/src/my-projct/_opam/lib/dyn@AB    &/Users/s/src/my-projct/_opam/lib/fileutils@ &/Users/s/src/my-projct/_opam/lib/ocaml/str@        '/Users/s/src/my-projct/_opam/lib/ocaml/unix@AB      %/Users/s/src/my-projct/_opam/lib/ordering@CD?/Users/s/src/my-projct/_opam/lib/pp@       /Users/s/src/my-projct/_opam/lib/seq@A      )/Users/s/src/my-projct/_opam/lib/stdlib-shims@     #/Users/s/src/my-projct/_opam/lib/stdune@A   4/Users/s/src/my-projct/_opam/lib/stdune/filesystem_stubs@  !/Users/s/src/my-projct/_opam/lib/toml@A    "default/bin/my-projct/.main.eobjs/byte@BCDE@*@A(@&@AB$@"@ @AB@CD@@A@@A@@)bin/my-projct@ABCDE"-w -@1..3@5..28@30..39@43@46..47@49..57@61..62-400-strict-sequence/-strict-formats,-short-paths*-keep-locs"-g@@@@6default/bin/my-projct/main@ࠠ$Main@9default/bin/my-projct/main.ml%ocaml@#.ml@@@@.workspace_root@B@@@+ocamlformat@@@&--impl@@@@*input_file@b@@@,.ocamlformat3.ocamlformat-ignore3.ocamlformat-enable@A$.mli@@@@#@"@@@!@@@&--intf@@@@ @@@@:default/bin/my-projct/main.mliB/dune__exe__Main@@B@@A@@
```

I see references to `stdune` and `fileutils` which are two of the external
libraries I'm using in my project.

The [Dune
docs](https://dune.readthedocs.io/en/stable/usage.html#querying-merlin-configuration)
also describe some merlin-related commands. Of note, running `dune ocaml dump-dot-merlin`
in a directory with some .ml files will print out a config file like the one in
merlin's custom build system docs, but so far I haven't managed to make that
type of config file actually do anything to OCaml-LSP.
