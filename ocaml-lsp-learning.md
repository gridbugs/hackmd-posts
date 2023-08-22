# Getting the OCaml Language Server Working in a Strange Non-Dune Project

My goal is to get [OCaml-LSP](https://github.com/ocaml/ocaml-lsp) - the
[Language Server Protocol](https://microsoft.github.io/language-server-protocol/)
implementation for OCaml - running on a project that builds in an unconventional way.
OCaml-LSP is the tool than allows text editors to navigate through source code
and annotate symbols with their documentation, and many other tasks to help
developers write code.

My unconventional project is an experiment to see if Dune files can be generated
at build time. I have a directory with some OCaml source files (but no `dune`
file and a single top-level metadata file with things like the name of the
package and a list of dependencies (kind of like a combination of `dune` and
`dune-project` files but with different syntax). I wrote a script which takes
this non-Dune project and generates a temporary directory `_dune` containing a
copy of all the OCaml source code and generates the Dune files (`dune-project`,
`dune`, etc), then runs `dune build` inside the `_dune` directory to build the
generated dune project. This works fine but when I open the original OCaml
source files in my editor all references to external libraries are highlighted
as errors.

All the docs for OCaml-LSP and the editor service
[merlin](https://ocaml.github.io/merlin/) which it uses internally express
pretty clearly that they work best with Dune but they can still be used without
Dune with some configuration. My project does of course technically still use
Dune but you wouldn't know it by looking at its source code. It would be more
accurate to say that the script that generates the temporary dune project is my
project's build system rather than Dune. That is to say that OCaml-LSP and
merlin are going to take some configuration in order to work in my project.

This post will describe the process of getting OCaml-LSP working in my non-Dune
project. I'll be using neovim with the
[LanguageClient-neovim](https://github.com/autozimu/LanguageClient-neovim)
plugin as my editor.

## First Steps

As as sanity check the first thing I tried was running my script to generate a
Dune project, building that project, and testing OCaml-LSP by editing the copies
of the source files in the generated project. Here everything worked as I
expected, for example I could jump to the definition of a function from an
external library installed with Opam. Editing the original version of that file
(outside the generated Dune project) I can jump to the definition of local
functions and functions from the OCaml prelude but not external libraries, plus
OCaml-LSP highlights any external modules as errors as it can't find the
libraries where they are defined. Also, back in the generated Dune project, if I
delete the `_build` directory and attempt to edit a copy of a source file I get
the same behaviour as if I was editing the original (OCaml-LSP doesn't know
about external libraries). That's what I expected - something inside the
`_build` directory was allowing OCaml-LSP to find external libraries.

So the question is which files from `_build` are important for letting OCaml-LSP
find external libraries?

To get some more information about how OCaml-LSP works I enabled logging in my
language server client. In LanguageClient-neovim I added the following to my
`~/.config/nvim/init.vim`:
```vim
let g:LanguageClient_loggingFile = expand('~/.vim/LanguageClient.log')
```

Then I watched `~/.vim/LanguageClient.log` as I tried to navigate to the definition of a symbol
that OCaml-LSP didn't know about. This error from the log caught my eye:
```
'Locate' query to merlin returned error: not in environment: Toml.Types.Table.Key.to_string
```
(`Toml.Types.Table.Key.to_string` was the function whose definition I was trying
to jump to.)

It's time to learn about `merlin` - the OCaml code analyzer used internally by
OCaml-LSP.

## Merlin

Merlin's [website](https://ocaml.github.io/merlin/) links to [docs for manual
configuration](https://github.com/ocaml/merlin/wiki/Project-configuration). It
assumes your project builds with Dune by default but if you use a different
build system  you can place a `.merlin` file at your project root with
instructions on how to find code and libraries.

Its syntax looks like:
```
# Source directories
S src
S src/foo
S src/bar

# Build artifact directories
B _build/src
B _build/src/foo
B _build/src/bar
```

OCaml-LSP's docs say that if I start it with a flag `--fallback-read-dot-merlin`
then it will respect the `.merlin` file at the root of my project. I configured
my editor to start OCaml-LSP with that command:
```vim
let g:LanguageClient_serverCommands = {
\ 'ocaml': ['opam', 'exec', 'ocamllsp', '--', '--fallback-read-dot-merlin'],
\ }
```

Now I need a way to generate a `.merlin` file for my project. Since this already
works for the generate Dune project as long as its `_build` directory is
present, that's the first place I checked.

A cursory glance revealed a file
`_build/default/bin/my-project/.merlin-conf/exe-main` which doesn't look
human-readable but dose contain strings that seem to correspond to the locations
of external libraries.

```
$ cat _build/default/bin/my-project/.merlin-conf/exe-main
DUNE-merlin-confv4:C    "/Users/s/src/my-project/_opam/lib/ocaml
$/Users/s/src/my-project/_opam/lib/ISO8601@A "/Users/s/src/my-project/_opam/lib/csexp@
&/Users/s/src/my-project/_opam/lib/fileutils@ &/Users/s/src/my-project/_opam/lib/ocaml/str@
...
```

I see references to `fileutils` and a bunch of other external libraries
I'm using in my project.

But digging through the dune source code it looks like this is a custom format
Dune uses to store internal state to disk so it doesn't have to recompute it on
subsequent runs:
```
(** Persistent values *)

(** This module allows to store values on disk so that they persist after Dune
    has exited and can be re-used by the next run of Dune.

    Values are simply marshaled using the [Marshal] module and manually
    versioned. As such, it is important to remember to increase the version
    number when the type of persistent values changes.

    In the future, we hope to introduce a better mechanism so that persistent
    values are automatically versioned. *)

```

So this is isn't going to work for me. Even if I could parse that file I would
be exposed to the risk of its format changing without warning and breaking my
code, since it's not intended to be stable across Dune releases.

The [Dune
docs](https://dune.readthedocs.io/en/stable/usage.html#querying-merlin-configuration)
also describe some merlin-related commands. Of note, running `dune ocaml dump-dot-merlin`
in a directory with some .ml files will print out a merlin config file.

```
$ dune ocaml dump-dot-merlin
EXCLUDE_QUERY_DIR
STDLIB /Users/s/src/my-project/_opam/lib/ocaml
B /Users/s/src/my-project/_opam/lib/ISO8601
B /Users/s/src/my-project/_opam/lib/base
B /Users/s/src/my-project/_opam/lib/base/base_internalhash_types
...
S /Users/s/src/my-project/_opam/lib/ISO8601
S /Users/s/src/my-project/_opam/lib/base
S /Users/s/src/my-project/_opam/lib/base/base_internalhash_types
...
```

So I put the output of that command in a `.merlin` file at the root directory of
my project and tried editing an original OCaml source and...

It still didn't work. It behaved the same as if the `.merlin` file wasn't
there.

Maybe I missed something in configuring OCaml-LSP. Taking a look at its CLI
docs:
```
$ ocamllsp --help
...
  --fallback-read-dot-merlin read Merlin config from .merlin files.
    The `dot-merlin-reader` package must be installed
```

`dot-merlin-reader` eh? There was no mention of that on their online
documentation.

That was probably my issue. I installed it:
```
$ opam install dot-merlin-reader
```

And now...

It still doesn't work. Same problem as before.

## Going Deeper


