+++
title = 'Scripting With awk'
date = 2021-07-09
+++

The awk scripting language is an oft-overlooked tool in the Unix toolkit.  One (harmful) consequence of this state of affairs is the invention of Bash, as awk is the natural fit for that niche between sh and a more generalized programming language.

One might be forgiven for thinking awk is actually unsuitable for portable scripting, though, and thus coming to the conclusion that one might as well just write a Bash script.  After all, even if Bash is not readily available everywhere (thus destroying some level of portability), at least you can use a shebang line for Bash scripts that is portable on any system that has Bash installed:

    #!/usr/bin/env bash

Doing the same with awk does not work.  A working awk shebang line depends on where awk is installed on the system.  Compare two common examples:

    #!/bin/awk -f

    #!/usr/bin/awk -f

The problem here is obvious: without env, you need a different shebang line, which means the script itself must be edited to port it between systems with different "standards" for filesystem placement of tools such as awk.  Obviously, the One True Way to organize the filesystem hierarchy would result in the `#!/usr/bin/awk -f` shebang line, but some less Unix-like systems clearly violate such standards of good taste.

Most Unix-like systems do not handle options well enough to include the `-f` option in an `env` based shebang line, so we're stuck.  We can't have a portable awk script executed simply by filename in `$PATH`.

Actually, we can.  "Think outside the box."  If you never had a teacher tell you that, you never had a real teacher.

[Harvested from StackExchange][gilles] (yes, occasionally there's something useful in that cesspool), we find this beauty of indirection:

    #!/bin/sh
    "exec" "awk" "-f" "$0" "$@" && 0 {}

That's right: we can get around this problem by using a standard shell shebang line and executing awk from within the script.  (See that page for a note about truly ancient Unices.)

The line immediately following the shebang line might need a little more explanation.  Assume a file with the above shebang line trickery with the name `example.awk`.

First of all, both lines are valid POSIX standard shell and valid POSIX standard awk at the same time.  When executed, the `sh` executable gets invoked to interpret the contents of the file, which leads to invoking the shell's `exec` builtin, which in turn replaces the shell process with the awk process created by its own following argument, `awk`, with `-f` as its first parameter; that option takes a filename argument.  In this case, the argument `$0` specifies the the `example.awk` file, and `$@` passes along any positional parameters following the user's initial command line input of the filename.  Thus, assume the user entered this:

    $ example.awk foo bar

In that case, `$0` gets replaced by `example.awk`, and `$@` gets replaced by `foo bar`.

In short, the entire above process becomes the following stepwise process.

1. Execute `/bin/sh` with arguments `foo bar` and feed it the contents of this file to interpret and execute.
2. Replace variables with their values, so that the line following the shebang becomes `"exec" "awk" "-f" "example.awk" "foo" "bar" && 0 {}`
2. Execute the `exec` command to replace the `sh` process with a new `awk` process, with arguments `-f example.awk foo bar`, specifying `example.awk` as the script file `awk` should read and passing `foo bar` along as a paramter for the script.

In awk syntax, the shebang line itself is discarded as a comment.  The following line contains a series of strings, which means that line would result in producing a nonzero value, which would result in default behavior of printing current input an extra time during execution (explaining this is beyond the scope of this article: learn awk for more detail).  By attaching `&& 0 {}` at the end, we change the final value of the line, preventing that duplication of input in awk script output.

<p class="subtitle signature">by <strong>apotheon</strong> and <strong>zenema</strong</p>

[gilles]: https://unix.stackexchange.com/questions/361794/why-am-i-able-to-pass-arguments-to-usr-bin-env-in-this-case#answer-361796
