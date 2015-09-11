# How to create a static blog in shell (part 3)

Continued from [part 2](http://mnetic.ch/blog/2013/08/18/smallblog_howto_part_2.md.html).

This third section comes in straight off the first release of smallblog. This release cleans up a lot of ugly areas that were added just to get the blog up and running. It also pulls out most of the configuration into site specific files, removing the need for the user to ever touch the main executable. These specific changes were set off by the ugliness of the `stat(1)` solution. Fixing the use of `stat(1)` ended up cascading through quite a few areas, and I cleaned up the code as I went, removing most of the ugly parts. There are still a few specific areas that need a better solution found, but the code has been returned to a presentable state.

## Configuration variables vs configuration files
The first revision of smallblog had no configuration in it. The main loop was a single line that appended any markup file to the main page. As I layered on bits, I threw a few variables at the top of the script to collect logically configurable parts like the title in the header and the contact details in the footer. There was little need to change this when I was only generating a single site, but when I wanted to generate a second site, making a copy of the script for each site configuration was too much. Moving the configuration to a file turned out to be really easy. As my configuration section was already simple a shell script, I just pulled all of the related lines into a new file, and sourced that file, as such:

    $ cat smallblog
    . ./smallblog.conf

and

    $ cat smallblog.conf
    max_posts=3

The `.` (single dot) built-in is a more portable version of the `source` built-in, and will just execute the contents of the file. This is most commonly seen while setting up shell and other dotfile configurations, but it turns out to be a rather clean solution here. This line is executing the smallblog.conf file in the current working directory. Because smallblog is run in the root of the site, this makes it trivial to also drop the configuration file in the site root, and then pick it up automatically when smallblog is run. Now only one executable per system is required to serve multiple sites.

## RSS feeds
The RSS generation function was probably one of the worst sections of code I've written for smallblog. I'm a frequent user of RSS, but I've never had to write an RSS feed before. I know they're XML, but beyond that, nothing else. So the first thing I did was go to [wikipedia](https://en.wikipedia.org/wiki/RSS) and implement the format in the example section. The commit is [here](https://github.com/abyxcos/smallblog/commit/df944ab4d9518cc956748dd99a42155d3b6d3153#diff-65be323c984cb0bef3372e71fc23b2f2R120) if you wish to view the initial function I used. The structure of the function should be familiar by now. I `echo`ed a header, and then jumped straight into the `for` loop to generate an entry for each post. As I had no understanding of how RSS worked, or what it requires, I wrote a sequence of very strange code that was driven by trial and error. Each section was created by forcibly acquiring and molding the information I wanted into the RSS feed. This led to lines like the following

    echo "<pubDate>"
    grep '<p class="meta">Date:' "${post}.html" |
        sed 's/^\s*<p class="meta">Date: //' |
        sed 's/<\/p>$//'
    echo "</pubDate>"

which I used to pull the date from the html version of the post. The proper solution in this case would have been to just re-`stat(1)` the markdown file to grab the date, rather than attempting to use regexes on HTML, and hard-coding the format in more locations. As I was cleaning up the code for the [v0.2](http://mnetic.ch/blog/2015/02/08/smallblog_v0.2.md.html) release, this whole function got rewritten, rendering that previous line into something much simpler:

    echo "<pubDate>"
    echo "${post%/*}"
    echo "</pubDate>"

The full commit can be found [here](https://github.com/abyxcos/smallblog/commit/7f62cb0c43879a8c0ccb237ed8aa28a1dd53b5d4#diff-65be323c984cb0bef3372e71fc23b2f2R129). This new line simply uses shell [parameter/variable substitution](http://www.tldp.org/LDP/abs/html/parameter-substitution.html) to take the post path, in the form of `year/month/date/post_title.md` and chop off the rightmost section matching `/*`. This leads into the **Portability** section.

## Portability
As I mentioned in [part 2](http://mnetic.ch/blog/2013/08/18/smallblog_howto_part_2.md.html), I use (and have used) many different systems. While this software was originally developed on a [Linux VPS](http://prgmr.com/xen/), this is my last remaining Linux system. Currently, my two primary systems are an [OpenBSD](http://www.openbsd.org/) desktop, and a [Solaris VPS](https://www.joyent.com/). I knew going into this project that I would be deprecating my Linux VPS in favor of a BSD or Solaris VPS, so smallblog had to be portable.

Ignoring my RSS feed generation function, the use of `stat(1)` as a crutch to grab post times was really ugly, and not something I wanted to deal with. The appropriate solution in this case is to write a small C program to call `stat(2)` [(OpenBSD)](http://www.openbsd.org/cgi-bin/man.cgi/OpenBSD-current/man2/S_ISBLK.2?query=stat&sec=2) [(Linux)](http://linux.die.net/man/2/stat) and return the mtime of the file in the format I want. The downside of this solution is that I now add an extra dependency or step into the use of the script.

In a stroke of genius, I remembered that I am already storing every post in a series of folders that gives the date I need. All I need to do is remove the filename from the `year/month/date/post_title.md`, and I have the same date string. The downside of this is that I lose the `%H:%M:%S %z` time and timezone from the posted line. Not perfect, but the solution reduces the complexity enough that I am willing to sacrifice the time display for it. I was able to rewrite my use of `stat(1)`

    TIME_CMD="stat -c %y %FILE% |
        sed 's/\..* / /'"
    #TIME_CMD="stat -f
        \"%Y-%m-%d %H:%M:%S %z\" %FILE%"

    _time=`eval ${TIME_CMD/\%FILE\%/$1}`

to just

    date=${1%/*}
    echo ${date//\//-}

I mentioned shell [parameter/variable substitution](http://www.tldp.org/LDP/abs/html/parameter-substitution.html) in the RSS section, but I think this example shows the strength of it better than my inappropriate use of regexes on HTML. To break this down so you don't have to read the whole manual, the first line takes the variable `${1}` I am passing to `make_post()`, which is the file path in the `year/month/date/post_title.md` format, and uses a non-greedy right anchor to knock off the last matching `/*`, which is the filename and the trailing slash. With variable substitution, `#` and `%` act similar to the perl anchors `^` and `$`, however they can be swapped to greedy easily with `##` and `%%`. Once the filename has been chopped off, and I now have the date in `year/month/date` format, I use the second substitution, a normal regex format to replace the slashes with dashes. This is the same type of substitution I was already using with `stat(1)`  to replace the `%FILE%` tag with the filename from `$1`.

The catch with variable substitutions is that they are not POSIX compliant. As I was already using them, there is no change in functionality, at a great increase in portability and cleaner code, so I have no problem with this trade-off. The problem that arises is I don't know the extent of the support of this feature. I have tested this in KSH and BASH on several platforms, both running as a login shell and in `/bin/sh` (POSIX compatible) mode, so while it "works for me", I am relying on user reports to determine compatibility. If you pick it up, and find a shell it doesn't work in, file an [issue](https://github.com/abyxcos/smallblog/issues) so I know. In the same vein, I have left the `#!` line at the top of the script as `#!/bin/sh`. Even though smallblog is no longer POSIX compatible, this choice was deliberate. All of my systems have different shells, so locking it to `#!/bin/bash` or `#!/bin/ksh` would be pointless, and a hassle, and it works in POSIX mode on top of both of those shells. This will causes issues on Debian though, where the system shell is provided by [dash](https://en.wikipedia.org/wiki/Almquist_shell), a strict POSX ash variant.

## Where to next?
With the release of [v0.2](http://mnetic.ch/blog/2015/02/08/smallblog_v0.2.md.html) I am happy with the state of smallblog. There are no remaining feature related bugs, but a bit of polish is still required in some functions. Pagination is also something I want to consider, as it is a handy feature, but generally doesn't lend itself to static generation. [Part 4](http://mnetic.ch/blog/2015/09/10/smallblog_howto_part_4.md.html) of this series will likely coincide with one of the next versions I release, and will likely discuss more code clean-ups that I've managed, along with any new features that make their way in.

tags: smallblog shell bash
