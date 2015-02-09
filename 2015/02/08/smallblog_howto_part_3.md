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

