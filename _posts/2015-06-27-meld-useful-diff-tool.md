---
layout: post
title:  "Meld: a useful diff tool"
date:   2015-06-27 12:36:02
categories: tools
thumbnail: /images/meld-my-favourite-diff-tool/thumbnail.png
---
<h2>Introduction</h2>

<img src="/images/meld-my-favourite-diff-tool/meld-96.png" alt="Meld logo" class="float-left" />

[Meld](http://meldmerge.org) is a cool and simple tool that you can use to compare either files or even whole directories in a fashionable graphical way.

<h2 style="clear: both;">Installation</h2>

Meld can be installed on most of the operating systems:

  * Windows - using straightforward .msi installer available on the [Meld homepage](http://meldmerge.org)
  * Linux - using package manager on most Linux distributions, e.g. for Ubuntu/Debian you can use this command:
  ``` sudo apt-get install meld ```
  * Mac OSX - using [.dmg](https://github.com/yousseb/meld/releases) packaged by [yousseb](https://github.com/yousseb)

<h2>Basic usage</h2>

This is how the main window looks like:

<img src="/images/meld-my-favourite-diff-tool/basic-window.png" alt="Basic Meld window" />

You can either compare single files, compare directories or use Meld as a version control merge tool.

<h2>Directory comparison</h2>

This is how example directory comparison looks:

<img src="/images/meld-my-favourite-diff-tool/directory-compare.png" alt="Example of directory comparison" />

<h2>File comparison</h2>

From directory comparison you can dive directly into file comparison:

<img src="/images/meld-my-favourite-diff-tool/file-comparison.png" alt="Example of file comparison" />

Meld provides many useful shortcuts:

<img src="/images/meld-my-favourite-diff-tool/file-comparison-options.png" alt="File comparison options" />

<h2>Directories comparison workflow</h2>

To merge changes from one directory to the other you can select the source directory and click "Copy right" (alt + &rarr; on Mac):

<img src="/images/meld-my-favourite-diff-tool/directory-compare-after-copy-right.png" alt="Directories after copy right" />

Then you will have to only delete files and directories missing in the source directory.

<h2>Final thoughts</h2>

Meld is a very nice diff tool. If you cannot use your IDE and VCS to merge files/directories Meld is an invaluable option.
