% Stuff I Can Never Remember

    [BLOpts]
    profile    = witt3rd
    postid     = 69
    tags       = [awesome, stuff, blogging]
    categories = [Writing, Stuff]
    
Inspired by [What I Wish I Knew When Learning Haskell](http://dev.stephendiehl.com/hask/), I am collecting a series of small articles on various topics that I frequently need/use but just can't ever seem to remember the details of.

## BlogLiterately
This very blog is edited on a Mac (using [Mou](http://25.io/mou/)), stored on [my GitHub](https://github.com/witt3rd/blog), and uploaded to [this site](witt3rd.com) using Brent Yorgey's [BlogLiterately](https://byorgey.wordpress.com/blogliterately/) tool:

    /.cabal-sandbox/bin/BlogLiterately ./2015-09/sicnr/main.md

This assumes several things have already been configured.  BlogLiterately is installed by creating a cabal sandbox and installing BlogLiterately:

    cabal sandbox init
    cabal install BlogLiterately

Everything is self-contained at this point.

I use a "profile" by hosting a "dotfile" on my Dropbox then symbolically linking it to my home directory, as in:

    ln -s ~/Dropbox/.BlogLiterately ~/.BlogLiterately

(Remember: linking is always source then destination.)

In this configuration file (`witt3rd.cfg`) I have a set of options, such as the blog URL, user name and password, and some options for generating the blog.

In each blog post, at the top, I use the Pandox title followed by a set of blog options, as in:

    % My Title
    
    [BLOptions]
    profile    = witt3rd
    postid     = 200
    tags       = [awesome, stuff, blogging]
    categories = [Life, Haskell]
    
This keeps everything self-contained and self-describing (i.e., no extra information being passed in).

## Homebrew
How did we exist before [Homebrew](http://brew.sh)??

Install it now:

    ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"

Always keep things up-to-date:

    brew update
    brew upgrade
    
### Synchronizing Brews
I use several different computers and I like to keep things synchronized.  For Homebrew, there is a simple little script (brew-sync.sh)[https://gist.github.com/jpawlowski/5248465].  It will take your current brew configuration and merge it with one on your Dropbox (`~/Dropbox/Apps/Homebrew`).  So, if you already have a bunch of cruft on the box you are trying to sync, it is best to clear out the old brew first:

    ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/uninstall)"
    
And reinstall before syncing.  I placed the `brew-sync.sh` script in my path and run it on a fresh system or anytime I add something.  Removing things is not so nice, though.  Also, it doesn't keep track of custom build settings, like `emacs --with-cocoa`.  Oh well, nothing is perfect.

## Hack Font
I like the [Hack font](https://github.com/chrissimpkins/Hack) and use it my terminal and editors.

## Cabal
Of course you have already `brew install ghc cabal-install`.  From $HOME, I periodically `cabal update`, since this is useful for system-wide stuff.  But, for the most part, I user sandboxes everywhere.

### New Project
In a new directory:

    cabal init
    cabal configure
    cabal sandbox init
    cabal install -j4 --only-dependencies

## Emacs
After `brew install emacs --with-cocoa` it's all about configuration.  Again, I use a symlinked `.emacs.d` to my Dropbox to keep things synchronized across my boxes.

### Haskell

## iTerm 2

## ZSH