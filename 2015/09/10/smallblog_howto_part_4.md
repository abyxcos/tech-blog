# Smallblog.pl, or how not to create a static blog in shell (part 4)

Continued from [part 3](http://mnetic.ch/blog/2015/02/08/smallblog_howto_part_3.md.html).

Much like the third part of this series, this coincides with the release of smallblog.sh 0.3 and smallblog.pl 0.5. Both of these releases attempt to address problems encountered with smallblog on [ZFS](http://open-zfs.org/). On ZFS, `open(2)` and `stat(2)` seem to be heavy operations. This would likely be mitigated on a dedicated server by the various caches, but on a [virtual server](https://www.joyent.com/), these become rather heavy compared to Linux's filesystem. Smallblog.sh 0.3 attempted to work around this by moving the heaviest operations in terms of lookups and reads into plugins, so execution of this codepath wouldn't be required, and it would be easy to replicate the ill-performant code in another language. Smallblog.pl addresses this with a rewrite and re-architect of the code to avoid so many lookups and reads. This approach isn't incompatible with shell, but it does not fit the architecture of smallblog.sh, which relies on piping data, rather than ever storing it to reference later. Smallblog.sh could have been rewritten, but I see no advantage of shell when used as an applications language. I also wanted to leverage a templating system to remove the HTML generation from the middle of the script, and perl offers several nice choices in that area. Again, this approach isn't incompatible with shell, and a template would work the same in both applications. What finally caused me to abandon shell was the lack of compatibility for non-POSIX features and extensions across shells. As I don't use BASH, writing BASH-specific features into my code was never an issue, but I have a fully heterogeneous set of systems, and expect smallblog to run on all of them. The disparity among shells finally became too much, so I needed to switch over to a language with a living specification, not a frozen one.

## A rewrite
Due to the *small* size of *small*blog, a rewrite was easy. I had a set of inputs and outputs to test against, and a rather small specification to output data against (the [jekyll default theme](https://github.com/jekyll/jekyll/tree/master/lib/site_template).) So I just retraced the same steps that I used to create smallblog.sh, starting inside and layering on functionality as I moved outwards towards a complete implementation.

I began with the familiar

    my $text = read_file($path);
    $html = markdown($text);

and called it in a loop with a

    my @paths = split("\n", `ls -r */*/*/*.md`);
    foreach my $path (@paths) {
        ...
    }

Even for those not familiar with perl, this line may trigger some memories. It looks rather similar to a certain

    for post in `ls -r */*/*/*.md`; do
        ...
    done

All that perl snippet is doing is shelling out and using `ls(1)` to build an array of paths to posts. Why waste time digging through perl libraries when we can just ask the shell? Beats me. A proper perl implementation of this would still take just as many reads to the filesystem to build the list.

## Less stat more storage
As `stat(2)` and `open(2)` was what I wanted to avoid, I had to change from querying files every time I needed a single line to storing everything in a data structure and working against that. As it turns out, the access pattern for every function (except the main index page) is to read every file, process part of it, then pipe that output to disk in it's final form. By simply saving both the path and the file contents once, then acting on that, the filesystem access was reduced to a directory lookup and read for each file  (as opposed to a lookup and read on every file in each function.) The single exception, the main index page `index.html` only does this on the top few newest files (5 by default.) One loop with a maximum count, and there are now 10 less filesystem accesses.

It should be noted, this approach does incur a memory penalty, as now each post, it's path, and it's HTML version are now stored in memory. While the name *small*blog isn't meant to restrict this program's scope, I suspect most installations using this software will fit easily in RAM, even on smaller sized virtual servers. This is a trade-off, but I believe larger installations should make use of a proper database to allow more refined queries into the available data.

## Templates, now with less shell substitutions
Templating systems are great. Content is removed from code, and logic flow becomes cleaner and more simple, and anyone can edit the output without knowing how to "code". Smallblog.sh was slowly growing into templates, despite my effort to not reinvent a templating library. To allow for the dynamic titles for each page (just the post title, or the site title for the main page,) labels in the style of `%TITLE%` were sneaking into the `$blog_header` and `$blog_footer`, and then being regexed back out in `make_index()`.

With real templates, now the giant data structure I collected can be passed to the template, and the files just call out the variables they want in the form of `${site.title}`. Because the chosen templating system [Template Toolkit](https://metacpan.org/pod/Template) allows multiple formats for variable tags, both their standard `[% var %]` and shell style `${var}`, converting the existing HTML generation code involved one regex, and a handful of renamed variables.

As the templating system is modular, it supports more include options than just dropping a header and footer onto a page. The main index page is a particularly tedious point as it just duplicates most of the individual post page generation code, but can't reuse the code without rewinding the logic to remove headers and footers. With templates `post.tmpl` is now only the HTML to print the post, and the page generation code has become

    [% INCLUDE site_header.tmpl title=post.title %]
    
    [% INCLUDE post.tmpl %]
    <br />
    
    [% INCLUDE site_footer.tmpl %]

while the main index page can wrap the `[% INCLUDE post.tmpl %]` in a `foreach` loop, and still insert the "all posts" link at the bottom of the page

    [% INCLUDE site_header.tmpl title=site.title %]
    
    [% FOREACH post=posts %]
    
    [% INCLUDE post.tmpl %]
    <br />
    
    [% END %]
    
    <h3>
    <a
        class="extra"
        href="${site.prefix}/archive.html"
    >
        all posts
    </a>
    </h3>
    
    [% INCLUDE site_footer.tmpl %]

The "all posts" [`archive.html`](https://gitlab.com/abyxcos/smallblog/blob/0.5/templates/archive_page.tmpl) page has also substantially benefited from templates if anyone wants to peek.

## What next?
Smallblog.pl has now caught up to smallblog.sh in features, while being easier for me to maintain (and use.) As such, smallblog.sh is being deprecated in favor of smallblog.pl. I have purposefully skipped a version number to allow for one last smallblog.sh release should any bugs or interesting features come up.

As for smallblog.pl, it is a fairly direct translation from smallblog.sh, re-architecting aside. There are plenty of places the code and templates can be cleaned up. I would also like to further reduce the number of filesystem accesses. Right now, every file is blindly regenerated, even if the source and templates haven't changed. I would like to avoid the needless churn, as in most cases, only the main index page and archives need updating.

There likely won't be any new features introduced to smallblog.pl for a while, as it is currently feature-complete for my usage. If there are any features or changes that would interest you, feel free to email me, or simply file an issue.

tags: smallblog shell bash perl
