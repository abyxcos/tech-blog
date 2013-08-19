# How to create a static blog in shell (part 2)
Continued from [part 1](http://mnetic.ch/blog/2013/08/17/smallblog_howto.md.html).

In this second section, we'll talk about creating a way to list previous posts, portability fixes, and cleaning up the code. Yup, our current code is hardly pretty. And I assume that most of the programmers reading this have been a bit bothered. While our very first iteration was elegant in it's simplicity (being a one-lined pipe,) it quickly got hackish once we needed to add loops. And if you looked through the commits I linked, you'll notice that it gets worse and worse with escapes, and pipes everywhere. Something must be able to be optimized here.

Well, this was on purpose. Recently, on [Hacker News](https://news.ycombinator.com/) there has been a short but intense burst of articles about "getting things done" and "just doing it." I've been trying to create a personal website for way too many years now. It's never been a technical problem that stopped me, rather the small things. PHP came around, and it was cool to throw up small personal websites, but I never had anything to write about. Then frames and tables were going out of style in favor of convoluted CSS layouts, and PHP was going to die any day now. So I jumped on the python bandwagon, but now I had nothing to write about again. Then another year of wanting to write, but not having a site, followed by a few more years of having a wordpress blog, but hating the software and theme, and being worried that it would get hacked, taking down the rest of the projects on my server. In the last month I finally got external validation that people were actually interested in reading some of my posts, after getting a lot of interest in the hoops I jumped through to get a working root [ZFS](http://zfsonlinux.org/) filesystem on Debian Linux. Now that I had things to write about, I needed a place to write them. I decided to write about creating a blog as I created the blog, finally solving everyone's favorite circular dependency.

As you can see from the previous [post](http://mnetic.ch/blog/2013/08/17/smallblog_howto.md.html) and my commits, this code very much embodies the [MVP](http://en.wikipedia.org/wiki/Minimum_viable_product) principle. The core feature of this software is a single line pipe, surrounded by a few `echo`s, and it grew outwards from that single line. This has the benefit over the top-down approach in that your program always works, and you're only adding features to it. Taking the top-down approach can easily lead to trying to make everything work at once, and attempting to add features to a core that still isn't functioning. So, let's add some features to our core.

# Shell functions
Yup, shell has functions. And they'll let us stave off some of this uglyness we're getting from piping every line to index.html. So let's take our previous example, and pull out the core of that for loop into a function. Previously, we had this:

    $ cat smallblog
    for post in `ls -r */*/*/*.md`; do
        echo "<p><div class=\"post\">" >> index.html
        markdown "${post}" >> index.html
        echo "</div></p>" >> index.html
    done

The inside of this loop is the important part, and we want to pull out all those pipes to index.html. So, let's go with something like this:

    $ cat smallblog
    prepare_post(){
        echo "<p><div class=\"post\">"
        markdown "$1"
        echo "</div></p>"
    }

    for post in `ls -r */*/*/*.md`; do
        prepare_post "${post}" >> index.html
    done

Much cleaner now, right? Clean enough that we could even re-use this to generate individual .html pages we could link to for each post. Let's rewrite a bit that for loop, and set a variable up top to cap the maximum number of posts that can be on the front page at once.

    $ tail smallblog
    max_posts=5

    for post in `ls -r */*/*/*.md`; do
        prepare_post "${post}" > "${post}.html"
        if [ ${max_posts} -gt 0 ]; then
            prepare_post "${post}" >> index.html
            max_posts'`expr ${max_posts} -1`
        fi
    done 

Hardly as clean as a `if(max_posts--)`, but not every language can be C. With our new loop, we now have a bunch of abandoned files buried in our date folders, but no one else knows this. So let's link to them.

    $ head smallblog
    prepare_post(){
        echo "<h2 class=\"title\"><a href='/blog/$1.html'>
            `head -n1 $1 | sed 's/^#* //'`
            </a></h2>
            <p class=\"meta\">Date: $2</p>
            <div class=\"post\">"
        markdown "$1"
        echo "</div>"
    }

    for post in `ls -r */*/*/*.md`; do
        time=`stat -c %y ${post} | sed 's/\..* / /'`
        prepare_post "${post}" "${time}" >> index.html
    done

I'll explain those two regexes for everyone. The first one in the header, `head -n1 $1 | sed 's/^#* //'` grabs the first line of your markdown post, and strips the pound signs off of it, leaving you with the plain title for your link text. The second one, `stat -c %y ${post} | sed 's/\..* / /'` grabs the last modified timestamp from [`stat(1)`](http://linux.die.net/man/1/stat) and strips off the miliseconds from the time, giving us a nice time and dateto put on the post. For those of you who aren't interested in including the timezone on the post, you can pipe it through an additional regex to strip that bit off, such as ` | sed 's/ [+-].*$//'`.

A few of you that have been following along are now scratching your heads; that last example is giving me an error. Why did this happen, there haven't been any typos in the post so far? A few of you may have even checked the [code](https://github.com/abyxcos/smallblog/blob/5f4c320f1d96226239271db5092e038948a4c18b/smallblog#L57). Congratulations, you are most likely the owner of a Macbook. If you're more lucky, you may even be running [OpenBSD](http://www.openbsd.org/). But the most likely cause of this is that you are running a BSD userland. One of the better things Apple has done for the world is include a BSD userland with their operating system (based on FreeBSD, but with a GNU toolchain.) This has caused a large amount of pain to many Linux developers, as they've now realized that their portable unix software isn't really that portable at all. In fact, most of it's highly Linux specific. This isn't entirely their fault, the GNU foundation has been pushing the GNU is not Unix motto for a bit, and many GNU and Linux tools have diverged a bit from the standards, for better or worse. This leads to such things as the wildly different `stat(1)` options you are now seeing. The man pages listing the options of each `stat(1)` as follows:

* [Linux](http://linux.die.net/man/1/stat)
* [Mac OSX](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man1/stat.1.html)
* [OpenBSD](http://www.openbsd.org/cgi-bin/man.cgi?query=stat&sektion=1)

As you can see, OSX uses the BSD version of `stat(1)`, both of which differ from the current Linux implementation. I am currently developing on a Linux computer, so I didn't notice this right away (I'm not a heavy user of `stat(1)`,) but I felt it made for a good example. Another highly divergent utility is `ls(1)`. But now, for all of you BSD folks, the `stat(1)` line you're looking for is `stat -f "%Y-%m-%d %H:%M:%S %z" "${post}"`. This is included in the latest version of [smallblog](https://github.com/abyxcos/smallblog) so you can switch out the Linux line for the BSD line as required. I'm currently looking for a more portable method of finding the file modification time. If you know a good one, send over a pull request.
