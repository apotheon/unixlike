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

Harvested from StackOverflow (yes, occasionally there's something useful in that cesspool), we find this beauty of indirection:

    #!/bin/sh
    "exec" "awk" "-f" "$0" "$@" && 0 {}

That's right: we can get around this problem by using a standard shell shebang line and executing awk from within the script.

TO BE CONTINUED: This article is not yet finished.
