# How to create a static blog in shell (part 1)
This is a short description of the process I went about to create [smallblog](https://github.com/abyxcos/smallblog). If you're reading this, you've most likely heard of [jekyll](http://jekyllrb.com/). If you haven't heard of jekyll, it's a small ruby script to create a static set of pages out of markdown files, and link them all together with a simple index. No server-side language, so it's both [nginx](http://nginx.org/) friendly, and [Hacker News](https://news.ycombinator.com/) friendly. No more worries about caching plugins, or if your database is optimized for traffic. And no chance of updates breaking your site. Of course the downside is you must regenerate your whole blog every time you update a post.

As you're still here, you probably weren't interested in adding ruby to your setup for jekyll, and none of the python counterparts interested you. Or you found an engine you liked, but the theme was downright depressing. And now you've decided to make your own, but you're not sure where to start. Well, let me help you out.

## Step 1
Find a theme you like. No really. If you go down the path of trying to create one (again,) wire-framing things up, you'll probably give up (again.) I picked the [jekyll theme](https://github.com/mojombo/jekyll/blob/master/lib/site_template/css/main.css), which is the one currently in use here. Contrary to how they dressed up their site, the theme is nice and clean, and very simple. It's also easy to modify, and the kind folks over there released it under an MIT license. Now that you've picked a theme you like, it's trivial to modify the output of your script to line up the div classes with your new theme.

## Wait, what script?
See, picking a theme is so easy that you already forgot you didn't have a script. So, let's make a script.

    $ cat smallblog
    markdown */*/*/*.md > index.html

Seems easy enough. Now let's expand it a bit. We can store markdown files for each post in `year/month/day/`, and iterate over them. But it would be really nice if we had some way to visually separate posts, instead of just scanning the page for h1s. Let's spit out a header before each post so we can have a title, and even a date string.

    $ cat smallblog
    for post in `ls -r */*/*/*.md`; do
        echo "<p><div class=\"post\">" >> index.html
        markdown "${post}" >> index.html
        echo "</div></p>" >> index.html
    done

Hardly the most pretty script, but it's starting to look quite like a blog. The div class `post` plugs directly into jekyll's theme. If you're using a different theme, alter the html slightly so it all lines up. Now let's let our programmer sensibilities take over, and pretty this up into a real program. While we're at it, we can make it a bit more robust. This is left as an exercise to the reader. [Here](https://github.com/abyxcos/smallblog/blob/0ceabeaa5cf5a04247a03d930b481c257ce658e3/smallblog) is the initial code I created to print out the blog. This shows evidence of an unfortunate habit I picked up from another language that doesn't support regexes. The next step is to pull out code that actually generates the post and it's headers into a function so it can be called both individually (to create stand-alone post that can be linked to,) and in the main loop to populate the front page. There is some extra logic to limit the number of posts that appear on the front page, as it may get rather crowded once it's in use. [Here](https://github.com/abyxcos/smallblog/blob/5f4c320f1d96226239271db5092e038948a4c18b/smallblog) is the final code from today that removes the for loops, and replaces them with the wildcard regex from this article.

## What next?
Congratulations, we now have a serviceable blog. As we let the dust settle, now we realize that we're lacking any means of navigation beyond scrolling through the latest set of articles on the main page. So we need some kind of index of all the posts in the blog. As seen, we can easily generate a list of files with `ls`, and we can exploit our datefull directory structure to easily split up posts by month, or even year.

Also, a few of you may have noticed, this script isn't terribly portable. In particular, the way we use `stat(1)` to include the date in the posts only works on linux. Another problem that commonly crops up is the reliance on `/bin/bash` to create portable shell scripts. Smallblog was developed under pdksh, and executes under `/bin/sh`, so many of the common bash-isms that can creep into shell scripts have been avoided (you may notice the use of `expr` rather than `let` for the inline math.) We'll discuss this more in [part 2](http://mnetic.ch/blog/2013/08/18/smallblog_howto_part_2.md.html).
