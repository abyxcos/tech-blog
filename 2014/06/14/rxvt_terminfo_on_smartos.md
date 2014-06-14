# How to make your server happy while using rxvt
This one is for all of you [rxvt](http://en.wikipedia.org/wiki/Rxvt-unicode) fans out there. I'm sure many a time you have logged into some strange or new machine and been greeted by

    $ tmux
    open terminal failed: missing or unsuitable terminal: rxvt-unicode-256color

And I'm sure just as many of you have solved it with a quick

    $ export TERM=xterm

and been on your way. I finally got tired of this and decided to fix it. This post is targeted at [SmartOS](http://smartos.org/), as it's use of a segregated [pkgsrc](http://www.pkgsrc.org/) userland is fairly unique, although shared by [Mac OSX](http://brew.sh/). These missing terminal definitions also occur on Debian and [NetBSD](http://mail-index.netbsd.org/netbsd-users/2014/05/31/msg014665.html).

This problem occurs when you install a new terminal emulator to your local computer, now your servers don't know what capabilities it has. Servers have a much smaller database of popular terminals that generally includes xterm, but not rxvt and friends. If your server already has X installed, you can simply install your favorite terminal there and benefit from pre-packaged terminfo files. Most servers don't have X installed so we need to inform our server about our new terminal.

Unfortunately, terminfo entries are stored as binary, so we need to decompile them before passing them over to the server. On your host machine, [`infocmp(1)`](http://linux.die.net/man/1/infocmp) will handle this. If, like me, you only care about a single entry, `rxvt-unicode-256color`, then 

    $ infocmp -I rxvt-unicode-256color > rxvt-unicode-256color.terminfo

and [`scp(1)`](http://linux.die.net/man/1/scp)ing the resulting file to your server will do the trick. If you want more, just continue feeding [`infocmp(1)`](http://linux.die.net/man/1/infocmp) more names. If you're not sure of the exact names, the terminfo files are likely located under `/usr/share/terminfo/*/*` alphabetically on a Linux machine.

Now we need to compile and install the file on our server. On Linux and BSD, this is simple and handled by [`tic(1)`](http://linux.die.net/man/1/tic). To install for all users,

    $ sudo tic -s rxvt-unicode-256color.terminfo

or for just the local user,

    $ tic -s -o ~/.terminfo rxvt-unicode-256color.terminfo

If your terminfo files still aren't being found, you may need to set `$TERMINFO` to point to the installed path,

    $ export TERMINFO=~/.terminfo

SmartOS is a bit more complicated, due to the separate userlands. There are two `tic(1m)`'s, the system `tic(1m)` at `/usr/bin/tic`, and the pkgsrc `tic(1m)` at `/opt/local/bin/tic`. I was not able to locate the man pages online, but you may note the difference in

    $ MANPATH=/usr/man man -s 1m tic # system manpages
    $ MANPATH=/opt/local/man man -s 1 tic # pkgsrc manpages

If you [`ldd(1)`](http://www.illumos.org/man/1/ldd) the pkgsrc tmux,

    $ ldd /opt/local/bin/tmux

and look for curses, you will see that it's linked against `libcurses.so.1`, which is living in `/lib`. This means that `tmux` will default to looking in the system directory, `/usr/share/lib/terminfo`, which on SmartOS is read only. In this case, you *will* need to set `$TERMINFO` to ensure that your path is being found, as the system `tic(1m)` doesn't include `~/.terminfo` in it's default path like linux does. For a system install,

    $ sudo /opt/local/bin/tic -s rxvt-unicode-256color.terminfo
    $ export TERMINFO=/opt/local/share/lib/terminfo

or a local install,

    $ tic -s -o ~/.terminfo rxvt-unicode-256color.terminfo
    $ export TERMINFO=~/.terminfo

and now `tmux` should work.

tags: rxvt smartos solaris bsd linux tmux
