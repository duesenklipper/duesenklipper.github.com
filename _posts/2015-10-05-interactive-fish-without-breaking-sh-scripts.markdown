---
layout: post
title:  "Using fish interactively as 'login' shell without breaking sh-assuming scripts"
categories: linux shell bash fish login ssh howto
---
At some point last year I fell in love with the [`fish` shell](http://fishshell.com/). It features almost zsh-like power, very nice editing and searching, for example autocompletion like in a browser. I've been using it as my primary command line ever since.

I've also switched to using `fish` for scripting, rather than bash. This has a great advantage, and also a disadvantage: The bad thing is that its syntax is not `sh`-compatible. The good thing is that its syntax is not `sh`-compatible.

`sh` syntax is of course time-tested and much beloved by many users and admins. But if we're really honest, it's a bit of a mess. `fish` improves on this a lot. Have a look at [its tutorial](http://fishshell.com/docs/current/tutorial.html) to see things like in-terminal syntax highlighting, sane arrays, sane functions and plenty more.

However, with `sh` being a standard and `bash` being nearly everywhere, there is a lot of stuff out there that blindly assumes it can just dump `sh`-style commands into e.g. an `ssh` connection and have things work on the other hand. The culprit that annoys me the most is `ssh-copy-id` &mdash; when I changed my login shell to `fish` on a server, it broke down, because `fish` didn't understand what that script wanted.

There's an easy fix for that bug in the script: It could simply make `ssh` launch an `sh`-compatible shell before pouring in the commands. This fix is in the works in upstream, but at least in Ubuntu 15.04 it's not available yet.

That said, I wanted a general fix anyway, because there will certainly be other scripts that try to do this. `fish` will never be fully `sh`-compatible, and I don't mind that. I like `fish`-style scripting. So I want to make the _server_ smart enough to figure out what is going on. It turns out that that is pretty simple.

This is what I want to achieve:

* When I log in to the server via `ssh` I want to end up in a nice interactive `fish`.
* This `fish` should act as a login shell so things like the automatic `byobu` launcher in Ubuntu still 
  work.
* When I exit this `fish`, the connection should terminate instead of dropping me into an outer shell.
* When a script like `ssh-copy-id` connects, `fish` should not be started. Instead, something `sh`-capable should run, for example `bash`.
* I should still be able to run `bash` interactively, by running `bash` from `fish`.

Here's how I did it:

1. I changed my login shell back to `bash`:

        $ chsh -s /bin/bash

2. I added the following at the end of `~/.bash_login`:
        
        case "$-" in
            *i*) fish -il; exit ;;
              *)  ;;
        esac
        
What does this little monstrosity do? 

Since this is in `~/.bash_login`, we know we are a login shell. "Manually" launched `bash` instances will not run this script and so will behave normally. 

`bash` stores some of its runtime flags in `$-`. If it's an interactive shell, that string will contain `i`. Given that here we are in a login shell and combining that with the `i` flag 
being present, we know there is a human who just logged in and who has a keyboard to do interactive
things. Let's give him a `fish`.

The `-i` argument makes sure `fish` will be interactive too, and `-l` tells it to behave like a
login shell.

Finally, once `fish` exits, we call `exit` so `bash` will exit too, so the user will not even 
notice that there is a `bash` involved in all this.

In the other case, when `bash` is not interactive, we now know that there is a script connecting to
our server. We do nothing and let `bash` start up normally, so even bad scripts like `ssh-copy-id`
will find their familiar environment and just work.