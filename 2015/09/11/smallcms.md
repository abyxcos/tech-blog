# Introducing smallcms
Smallcms is a simple content management system (CMS) for allowing sections of your front page to become dynamically editable. Smallcms is a perl CGI app that you can drop into most sites.

Smallcms will iterate over any tag with a class that ends in `-editable` and present it as a text box, making it ideal for quick news tickers and small boxes that need to be updated frequently, but don't warrant adding a database to your site. The smallcms code is shorter than a page, and easy to understand. It currently does not offer any features except for `<br>` to `\n` conversions as appropriate. Smallcms does not care about it's name; it is suggested that the binary is named something more appropriate, such as `edit_news.pl` when installed.

Smallcms may be found on [gitlab](https://gitlab.com/abyxcos/smallcms).

tags: smallcms perl
