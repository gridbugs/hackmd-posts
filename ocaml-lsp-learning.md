# Getting the OCaml Language Server Working in a Non-Dune Project

Skip to the end if you just want instructions on how to do this!

This post will describe the process of how I got
[OCaml-LSP](https://github.com/ocaml/ocaml-lsp) working in my non-Dune project.
I used Neovim with the
[LanguageClient-neovim](https://github.com/autozimu/LanguageClient-neovim)
plugin as my editor, but see the instructions at the end of this post for the
equivalent VSCode configuration.

My goal was to get OCaml-LSP - the
[Language Server Protocol](https://microsoft.github.io/language-server-protocol/)
implementation for OCaml - running on a project that builds in an unconventional way.
OCaml-LSP is a tool that allows text editors to navigate through source code
and annotate symbols with their documentation, as well as performing many other tasks that help
developers write code.

My 'unconventional project' was an experiment to see if Dune files could be generated
at build time. I had a directory with some OCaml source files (but no `dune`
file) and a single top-level metadata file with things like the name of the
package and a list of dependencies (kind of like a combination of `dune` and
`dune-project` files but with different syntax). I wrote a script which took
this non-Dune project and generated a temporary `_dune` directory  containing a
copy of all the OCaml source code and the generated Dune files (`dune-project`,
`dune`, etc), then ran `dune build` inside the `_dune` directory to build the
generated Dune project. This seemed to work fine, but when I opened the original OCaml
source files in my editor all references to external libraries were highlighted
as errors.

All the docs for OCaml-LSP and the editor service
[Merlin](https://ocaml.github.io/merlin/), which it uses internally, express
pretty clearly that they work best with Dune but can still be used without
Dune with some configuration. My project did of course technically still use
Dune, but you wouldn't have known it by looking at its source code. It would be more
accurate to say that the script that generated the temporary Dune project was my
project's build system rather than Dune itself. Thus it took some manual
configuration to get OCaml-LSP and Merlin to work in my project.

I decided to explore what these configurations would look like with barely any
knowledge of how either OCaml-LSP or Merlin worked at the outset.

## First Steps

As a sanity check, the first thing I tried was running my script to generate a
Dune project, building that project, and then testing OCaml-LSP by editing the copies
of the source files in the generated project. Here everything worked as I
expected, for example I could jump to the definition of a function from an
external library installed with Opam. Editing the original version of that file
(outside the generated Dune project), I could jump to the definition of local
functions and functions from the OCaml prelude but not external libraries, plus
OCaml-LSP highlighted any external modules as errors as it couldn't find the
libraries where those modules were defined. Also, back in the generated Dune
project, if I deleted the `_build` directory and attempt to edit a copy of a
source file I would get the same behaviour as if I was editing the original
(OCaml-LSP doesn't know about external libraries). That's what I expected - there was
something inside the `_build` directory that was allowing OCaml-LSP to find external
libraries.

So the question was, which files from `_build` were important for letting OCaml-LSP
find external libraries?

To get some more information about how OCaml-LSP works I enabled logging in my
language server client (LanguageClient-neovim) by adding the following to my
`~/.config/nvim/init.vim`:
```vim
let g:LanguageClient_loggingFile = expand('~/.vim/LanguageClient.log')
```

Then I watched `~/.vim/LanguageClient.log` as I tried to navigate to the definition of a symbol
that OCaml-LSP didn't know about. This error from the log caught my eye:
```
'Locate' query to merlin returned error: not in environment: ...
```

The error message suggested that Merlin was unable to find the definition of that function.

It's time to learn about Merlin - the OCaml code analyser used internally by
OCaml-LSP.

## Merlin

Merlin's [website](https://ocaml.github.io/merlin/) links to [docs for manual
configuration](https://github.com/ocaml/merlin/wiki/Project-configuration). It
assumes your project builds with Dune by default, but if you use a different
build system you can manually configure Merlin by placing a `.merlin` file at
your project root containing instructions on how to find code and libraries.

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

OCaml-LSP's README says that if I run `ocamllsp` with a flag `--fallback-read-dot-merlin`
then it will respect the `.merlin` file at the root of my project. 

I configured
my editor to start OCaml-LSP with that command:
```vim
" This configures the language server neovim will start when editing an OCaml file
let g:LanguageClient_serverCommands = {
\ 'ocaml': ['opam', 'exec', 'ocamllsp', '--', '--fallback-read-dot-merlin'],
\ }
```

Note that it ran the command `opam exec ocamllsp -- --fallback-read-dot-merlin`
rather than simply `ocamllsp --fallback-read-dot-merlin`. This is because when you
install OCaml-LSP with Opam the directory containing the `ocamllsp` executable
won't be in your shell's `$PATH` variable unless you run `eval $(opam env)`
before launching your editor, and I _always_ forget to do this. Running the
command with `opam exec` works since `opam` is always in your
`$PATH` and knows how to find executables that were installed with Opam.
Don't forget the `--` between `ocamllsp` and `--fallback-read-dot-merlin` or
you'll be passing that flag to `opam` rather than to `ocamllsp`.

Now I needed a way to generate a `.merlin` file for my project. Since this already
worked for the generated Dune project as long as its `_build` directory is
present, that's the first place I checked.

A cursory glance revealed a file
`_build/default/bin/my-project/.merlin-conf/exe-main` which didn't look
human-readable but did contain strings that seem to correspond to the locations
of external libraries.

```
$ cat _build/default/bin/my-project/.merlin-conf/exe-main
DUNE-merlin-confv4:C    "/home/s/src/my-project/_opam/lib/ocaml
$/home/s/src/my-project/_opam/lib/ISO8601@A "/home/s/src/my-project/_opam/lib/csexp@
&/home/s/src/my-project/_opam/lib/fileutils@ &/home/s/src/my-project/_opam/lib/ocaml/str@
...
```

I could see references to `fileutils` and a bunch of other external libraries
I was using in my project.

But when digging through the Dune source code, it looked like this was a custom format
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

So that wasn't going to work for me. Even if I could parse that file it was
just a serialisation of an internal Dune data structure and not some general
format that Merlin would understand.

The [Dune
docs](https://dune.readthedocs.io/en/stable/usage.html#querying-merlin-configuration)
describe some merlin-related commands. Running `dune ocaml dump-dot-merlin`
in a directory with some `.ml` files will print out a config suitable for a
`.merlin` file:
```
$ dune ocaml dump-dot-merlin
EXCLUDE_QUERY_DIR
STDLIB /home/s/src/my-project/_opam/lib/ocaml
B /home/s/src/my-project/_opam/lib/ISO8601
B /home/s/src/my-project/_opam/lib/base
B /home/s/src/my-project/_opam/lib/base/base_internalhash_types
...
S /home/s/src/my-project/_opam/lib/ISO8601
S /home/s/src/my-project/_opam/lib/base
S /home/s/src/my-project/_opam/lib/base/base_internalhash_types
...
```

So I put the output of that command in a `.merlin` file at the root directory of
my project and tried editing an original OCaml source and...

It still didn't work. It behaved the same as if the `.merlin` file wasn't
there.

Maybe I missed something in configuring OCaml-LSP? A look at its CLI
docs revealed:
```
$ ocamllsp --help
...
  --fallback-read-dot-merlin read Merlin config from .merlin files.
    The `dot-merlin-reader` package must be installed
```

`dot-merlin-reader`? There was no mention of that in the README (at the time of writing).

That was probably my issue. I installed it:
```
$ opam install dot-merlin-reader
```

And now...

It still didn't work. Same problem as before.

## Debugging with `strace`

Is OCaml-LSP even trying to read the `.merlin` file? To answer this question I
made this shell script to run `ocamllsp` with `strace` (this only works on
Linux):
```bash
#!/bin/sh
OCAMLLSP=$(opam exec which -- ocamllsp)
strace $OCAMLLSP --fallback-read-dot-merlin 2> /tmp/ocamllsp.strace
```
This will run OCaml-LSP logging every system call to `/tmp/ocamllsp.strace`. I
saved this file to `strace-ocamllsp.sh` and put it in somewhere in my `$PATH`.
Then I updated my editor (Neovim) config to run the script when it edited an OCaml file:
```vim
" This configures the language server neovim will start when editing an OCaml file
let g:LanguageClient_serverCommands = {
\ 'ocaml': ['strace-ocamllsp.sh'],
\ }
```

With this configuration I opened an OCaml file in Neovim and then checked
`/tmp/ocamllsp.strace` for any mention of `.merlin` but that string didn't
appear in the trace. However this section of the trace caught my attention:
```strace
newfstatat(AT_FDCWD, "/home/s/src/my-project/src/dune-project", 0x7ffe4d08ec30, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/s/src/my-project/src/dune-workspace", 0x7ffe4d08ec30, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/s/src/my-project/src/dune", 0x7ffe4d08ec50, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/s/src/my-project/src/dune-file", 0x7ffe4d08ec50, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/s/src/my-project/dune-project", 0x7ffe4d08ec30, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/s/src/my-project/dune-workspace", 0x7ffe4d08ec30, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/s/src/my-project/dune", 0x7ffe4d08ec50, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/s/src/my-project/dune-file", 0x7ffe4d08ec50, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/s/src/dune-project", 0x7ffe4d08ec30, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/s/src/dune-workspace", 0x7ffe4d08ec30, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/s/src/dune", {st_mode=S_IFDIR|0755, st_size=4096, ...}, 0) = 0
newfstatat(AT_FDCWD, "/home/s/src/dune-file", 0x7ffe4d08ec50, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/s/dune-project", 0x7ffe4d08ec30, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/s/dune-workspace", 0x7ffe4d08ec30, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/s/dune", 0x7ffe4d08ec50, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/s/dune-file", 0x7ffe4d08ec50, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/dune-project", 0x7ffe4d08ec30, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/dune-workspace", 0x7ffe4d08ec30, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/dune", 0x7ffe4d08ec50, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/dune-file", 0x7ffe4d08ec50, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/dune-project", 0x7ffe4d08ec30, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/dune-workspace", 0x7ffe4d08ec30, 0) = -1 ENOENT (No such file or directory)
```
It looked like OCaml-LSP was trying to read files named `dune-project` or
`dune-workspace` starting next to the source file I opened and repeatedly
searching in parent directories going up to the file system root. In a Dune
project these files are usually present in the project's root directory which
also happens to be where the `.merlin` file is meant to go.

My first guess was that OCaml-LSP was using the `dune-project` or
`dune-workspace` file to identify the project's root directory before reading
the `.merlin` file from there. A little strange since
`--fallback-read-dot-merlin` is meant for use in non-Dune projects, but it was
easy to test:
```
$ touch dune-workspace   # creates an empty file named "dune-workspace"
```

And now it worked!

Just for fun I checked the trace:
```strace
newfstatat(AT_FDCWD, "/home/s/src/my-project/src/.merlin", 0x7ffc65ebc430, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/s/src/my-project/src/dune-project", 0x7ffc65ebc430, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/s/src/my-project/src/dune-workspace", 0x7ffc65ebc430, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/s/src/my-project/src/dune", 0x7ffc65ebc430, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/s/src/my-project/src/dune-file", 0x7ffc65ebc430, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/s/src/my-project/.merlin", {st_mode=S_IFREG|0644, st_size=5589, ...}, 0) = 0
```

So it was definitely looking for a `.merlin` file, and on the last line of that
snippet it found one!

Reading through OCaml-LSP's source code confirmed that `dune-project` and
`dune-workspace` were used to identify the project's root directory. Clearly this
was a bug in OCaml-LSP as the main use case for the `--fallback-read-dot-merlin`
flag is for when you're not using Dune.

It turned out this was a known issue: [.merlin based projects still require a
blank dune-project file](https://github.com/ocaml/ocaml-lsp/issues/859). That
issue had been open for almost a year and in an amazing coincidence the day
before I attempted this experiment a
[PR](https://github.com/ocaml/ocaml-lsp/pull/1173) was raised which fixed it.

## How to Use OCaml-LSP for Non-Dune Projects

To wrap up this post, below are instructions on how to configure your editor and
project to use OCaml-LSP without Dune.

### Run `ocamllsp` with the `--fallback-read-dot-merlin` Flag

Since `ocamllsp` is started by your text editor you'll need to configure your
editor to pass the extra flag.

#### Visual Studio Code

In VSCode using the [OCaml Platform Visual Studio Code
extension](https://marketplace.visualstudio.com/items?itemName=ocamllabs.ocaml-platform)
put this in your `~/.config/Code/User/settings.json`:
```json
{
    "ocaml.server.args": [
        "--fallback-read-dot-merlin"
    ]
}
```

#### Neovim

In Neovim with the
[LanguageClient-neovim](https://github.com/autozimu/LanguageClient-neovim)
plugin put this in your `~/.config/nvim/init.vim`:
```vim
let g:LanguageClient_serverCommands = {
\ 'ocaml': ['opam', 'exec', 'ocamllsp', '--', '--fallback-read-dot-merlin'],
\ }
```

### Install `dot-merlin-reader`

Run:
```
opam install dot-merlin-reader
```

### Install `ocaml-lsp-server` with Version `1.17.0` or Above

This version will include the bug fix that allows OCaml-LSP to find `.merlin`
without a `dune-workspace` or `dune-project` to be present.

At the time of writing version `1.17.0` hasn't been released but when it's
released you can install it with:
```
opam update
opam install ocaml-lsp-server
```

If it's not released yet or you just want the bleeding edge version of
OCaml-LSP, run this instead:
```
opam pin ocaml-lsp-server.git https://github.com/ocaml/ocaml-lsp.git
```

Note that the `git` in `ocaml-lsp-server.git` is the version number that will
be associated with the installation of the `ocaml-lsp-server` package. I chose
`git` to distinguish this version from other versions of that package known by Opam. You can replace `git` with whatever you like.

Check your version by running:
```
opam exec ocamllsp -- --version
```

### Make a `.merlin` File in Your Project Root

Read the [Merlin docs for manual
configuration](https://github.com/ocaml/merlin/wiki/Project-configuration). You
can generate `.merlin` with Dune in a temporary Dune project (as I've done in
this post) by running `dune ocaml dump-dot-merlin` in the directory containing
your code, or alternatively you can write a `.merlin` file by hand.

### Try Editing a File!

Hopefully OCaml-LSP will work correctly in your editor. This required more
configuration than I would have liked. I would rather OCaml-LSP respect
`.merlin` files by default and include `dot-merlin-reader` among its
dependencies so the user doesn't have to install it manually.

## Until Next Time

Hopefully this helps you enjoy the benefits of OCaml-LSP in your non-Dune
projects and gives a general example of how to debug situations where tools
aren't working the way you expect. As I said, the whole process involved more
configuration than I think should be necessary, but in the end it was
technically possible to use OCaml-LSP and Merlin in a non-Dune project.
