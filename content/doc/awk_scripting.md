+++
title = 'scripting with awk'
date = 2021-07-09
+++

The awk scripting language is an oft-overlooked tool in the Unix toolkit.  One ([harmful][harmful]) consequence of this state of affairs may be the invention of Bash, as awk is the natural fit for that scripting niche between sh and a more generalized programming language.

> DISCLAIMER: We refer to "POSIX" sh within this document.  The term is inexact, but about as close to the reality of what we mean as it seems possible to get without inventing new words or inflating the size of this document tenfold.  What we really mean involves a portable subset of any common POSIXish shell identified as `/bin/sh` on any remotely sane Unixy system.  This is, indeed, a very imprecise definition.  Learn to live with it.

You might be forgiven for thinking awk is actually unsuitable for portable scripting, though, and thus coming to the conclusion that you might as well just write a Bash script.  After all, even if Bash is not readily available everywhere (thus destroying some level of portability), at least you can use a shebang line for Bash scripts that is portable on any system that has Bash installed:

    #!/usr/bin/env bash

Doing the same with awk does not work.  A working awk shebang line depends on where awk is installed on the system.  Compare two common examples:

    #!/bin/awk -f

    #!/usr/bin/awk -f

The problem here is hopefully obvious: without env, you need different shebang lines on systems that store the awk executable in different locations, which means the script itself must be edited to port it between systems with different "standards" for placement of tools such as awk within the filesystem.  Obviously, the One True Way to organize the filesystem hierarchy would result in the `#!/usr/bin/awk -f` shebang line, but some less-Unixy systems clearly violate such standards of good taste.

Most Unix-like systems do not handle options well enough to include the `-f` option in an `env` based shebang line, so perhaps you're stuck.  It seems you can't have a portable awk script executed simply by filename in `$PATH`.

Actually, you can.  "Think outside the box."  If you never had a teacher tell you that, you may never have had a real teacher.

As [harvested from StackExchange][gilles] (yes, occasionally there's something useful in that cesspool), you may find this beauty of indirection useful:

    #!/bin/sh
    "exec" "awk" "-f" "$0" "$@" && 0 {}

That's right: you can get around this problem by using a standard shell shebang line and executing awk from within the script.  (See that StackExchange page for a note about truly ancient Unices, then ignore it if you don't absolutely have to port it to something obsolete.)

The line immediately following the shebang line might need a little more explanation.  Assume a file with the above shebang line trickery with the name `example.awk`.

First, note that both lines are valid POSIX standard shell and valid POSIX standard awk at the same time.  When executed, the `sh` executable gets invoked to interpret the contents of the file, which leads to invoking the shell's `exec` builtin, which in turn replaces the shell process with the awk process created by its own following argument, `awk`, with `-f` as its first parameter; that option takes a filename argument.  In this case, the argument `$0` specifies the `example.awk` file, and `$@` passes along any positional parameters following the user's initial command line input of the filename.  Thus, assume the user entered this:

    $ example.awk foo bar

In that case, `$0` gets replaced by `example.awk`, and `$@` gets replaced by `foo bar`.

In short, the entire above process becomes the following stepwise process:

1. Execute `/bin/sh` with arguments `foo bar` and feed it the contents of this file to interpret and execute.

2. Replace variables with their values, so that the line following the shebang becomes `"exec" "awk" "-f" "example.awk" "foo" "bar" && 0 {}`

3. Execute the `exec` command to replace the `sh` process with a new `awk` process, with arguments `-f example.awk foo bar`, specifying `example.awk` (as the script file `awk` should read) and passing `foo bar` along as paramters for the script.

The following explanation of how the `exec` line works is essentially a brief introduction to the awk language.  Buckle your seatbelts.

The syntax of awk is basically a series of two constructs: comments, and pattern/action pairs.  A pattern is evaluated for truth or falsehood.  If true, awk executes the following action; if false, it does not.  A missing pattern is considered true.  A missing action, or an empty action, is treated as a print action: `{ print }`.  If a pattern is always true regardless of input, awk will apply the associated action to every subsequent line of input, in addition to any other operations performed by later pattern/action pairs (such as printing lines of input, resulting in printing multiples of each line).

In the above shebang portability hack, the awk parser discards the shebang line itself as a comment (because it starts with `#`).  The next line contains a series of strings, which could constitute an awk pattern on its own that produces a true value.  The series of strings is part of a larger pattern, however, and not a stand-alone pattern in this case.  The larger pattern is two sub-patterns joined by a test condition (the logical "and" operator: `&&`).

If the sub-pattern preceding `&&` was false, awk would not evaluate the right-hand sub-pattern but, because it is true in this case, it does.  The right-hand side, `0`, evaluates as false, and this is the final value of the whole pattern.  The false value ensures the specified action never executes, avoiding the inconvenience of the script portability hack implicitly instructing awk to print all input.

Including an empty action does not appear necessary for POSIX correctness because awk should treat a missing action as a print action anyway.  To make the empty action even less relevant, the false value of the preceding pattern is zero, so the `{}` action should never execute, and an efficient parser should ignore it altogether.  Despite this, some awk implementations will insert `{}` or `{ print }` behind your back if it's missing, and some implementations may behave poorly without a specified action here.  It seems best to keep the empty action in your file for maximum portability assurance, even if the end (and desired) result is that awk never executes the action.  Besides, it looks more like proper awk syntax with an explicit action.

## Awkward Portability

A nicer approach to solving the awk shebang line portability problem would involve broad implementation of a POSIX standard version of either awk or env that behaves appropriately, perhaps by allowing contextual script execution without `-f`, or better argument handling by env.  Until the day that solution arrives -- with both POSIX standard support and sufficiently broad implementation support -- various other forms of trickery may come close.  One of us (the authors: in this case apotheon) cobbled togethr the following kludgey solution for improved portability, before the other stumbled across the POSIX shell approach above.  The following example of this alternative shebang line trickery employs an exceedingly simple awk wrapper script.  Create an executable file in your `$PATH` with the following contents:

    #!/bin/sh
    awk -f "$@"

Call this file "awkward", in the same sense that "starward" means "at or toward the stars".  With that in place, this shebang line works well:

    #!/usr/bin/env awkward

You're welcome.  Please don't use it if the sh shebang line solution works for your needs.

## Blame

All errors are apotheon's fault; it was apotheon who wrote the final draft.

<p class="subtitle signature">by <strong>apotheon</strong> and <strong>zenema</strong></p>

[gilles]: https://unix.stackexchange.com/questions/361794/why-am-i-able-to-pass-arguments-to-usr-bin-env-in-this-case#answer-361796

[harmful]: https://blogstrapping.com/2013.271.13.19.30/
