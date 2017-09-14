---
layout: post
title: "Clojure On Windows"
date: 2015-10-05 13:30:00
categories: clojure clojurescript windows
comments: true
---

### Updated: 14/09/2017

Most Clojure developers seem to use Linux or OSX, and the community is heavily predisposed towards those platforms.  Nevertheless, there are plenty of us who use Windows for whatever reason.  Clojure development on Windows is surprisingly smooth, but here's a guide to get you started anyway (and help me remember how to do it).  Note that a lot of this is not specific to Clojure.

# Terminal setup

The best solution is to install the [Windows Subsystem for Linux](https://msdn.microsoft.com/en-gb/commandline/wsl/install_guide).

## Git 

Conventional wisdom is to set `core.autocrlf` to `true` on Windows, but I think this is wrong.  The idea is that because you're on Windows, you need CRLF line endings.  I've never come across a Windows tool that doesn't correctly handle LF line endings (except notepad), but many Linux and OSX tools can't cope with CRLF.  I always set

    git config core.autocrlf input
    
to make sure that I'm always using LF everywhere.  This is the only setup that doesn't give me occasional problems around line endings, and is much easier to think about - if it's not LF anywhere, on my machine or anyone else's, it's wrong.  Do this in both Bash for Windows and in cmd.

_WARNING:_ If you use SourceTree, it will break your line endings if you discard changes.  Just for kicks.

### Create ssh keys

GitHub explain [how to create ssh keys in Git Bash](https://help.github.com/articles/generating-ssh-keys/) better than I can.  Follow steps 1 and 2 of their guide - we'll sort out the ssh agent in a better way next.  Don't forget to add your new ssh key to GitHub or BitBucket or wherever if you use a centralised git repo; your new public key file is in `~/.ssh/id_rsa.pub`.

### Remember git ssh key passphrase with ssh-agent

Add `eval $(ssh-agent)` to your `.bash_profile`.

### Aliases

You've probably got your own favourite aliases for git commands.  Git supports aliases in your `.gitconfig` file, which you can set like this:

    git config --global alias.co checkout
    git config --global alias.br branch
    git config --global alias.ci commit
    git config --global alias.st status

This isn't as good as regular bash aliases, as you still have to type `git` at the start of each command.  Here are my aliases, if you like 'em stick 'em in your `.bash_profile`:
 
    alias ll='ls -la'
    alias gs='git status'
    alias gl='git log --graph --pretty=format:'\''%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset\'\'' --abbrev-commit'
    alias ga='git add .'
    alias grp='git remote prune origin'
    
## Colours

I can't be bothered to learn how ANSI escape sequences and XTERM colours and stuff, just paste this in your `.bash_profile`:

    PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w \[\033[38;5;197m\]$(__git_ps1 "(%s)")\[\033[00m\]$ '

## ConEmu

Install [ConEmu](https://conemu.github.io/).  It provides a tabbed, split environment that can host any other consoles. It's worth having a look through all the settings as it's extremely customisable, but here are a few key ones. 

### Buffer size

You want a long buffer.  You can't have a infinite one like on Linux, but you can make it long in Settings > Main > Size & Pos > Long console output.  Set it to 9999.

### One tab per group

When displaying multiple consoles in a window, I want them all also to be within a single tab.  Go to Settings > Main > Tab bar and check the "One tab per group" checkbox.

### Copy/Pasting

There are a number of settings related to copy/pasting behaviour, but there's one in particular that I find annoying.  ConEmu will pop up a warning menu if you paste more than 200 characters or an enter keypress by default, but you can turn these off in the Settings > Keys & Macro > Paste section under "Confirm <Enter> keypress" and "Confirm pasting more than X chars". 

### Set Bash for Windows as default terminal


Go to Settings > Startup > Tasks and edit `{Bash::bash}` if it already exists, or press the + button at the bottom of the list if it doesnt. Set the task up to have the name `Bash::bash`, check all the boxes, set the task parameters to be:

    -icon "%USERPROFILE%\AppData\Local\lxss\bash.ico"
    
Set the console command in the big box at the bottom to be:
    
    %windir%\system32\bash.exe -cur_console:pm:/mnt --login -i -new_console
    
Don't forget to press the "Save settings" button at the bottom.

_NOTE: Getting the current directory in the tab title for Bash is a bit [more involved](http://stackoverflow.com/questions/39974959/conemu-with-bash-show-folder-in-tab-bar)._

### Keyboard shortcuts

These are some keyboard shortcuts I use to duplicate what iTerm got me used to.  Edit them by going to Settings > Keys & Macro.

#### Split horizontal/vertical

iTerm got me used to splitting terminal windows horizontally and vertically, filling the other half with a duplicate terminal.  The user shortcuts are called "Split: Duplicate active 'shell' split to bottom" and "Split: Duplicate active 'shell' split to right.  I use `Ctrl+Shift+H` and `Ctrl+Shift+V`.

#### Clear buffer shortcut

I like how iTerm allows the current buffer to be cleared with `Alt+K`.  I can't reproduce this behaviour exactly here (something to do with [not clearing buffers of running programs](http://superuser.com/questions/898426/clear-console-buffer-in-conemu-with-cygwin)), but it is possible to clear the buffer with `Alt+K` when no other command is running, by going to Settings > Keys & Macro, and adding a shortcut in one of the Macro slots.  Set the Hotkey to be `Alt+K` and the Description to be:
 
     print("\e echo -e '\0033\0143' \n")  
     
Note that this actually clears the buffer, rather than just scrolling to the end of it, which is very useful when running tests and things.  What this does is type a command to clear the buffer and press enter for you, but you won't want this command clogging up your bash history.  Once you've got Git Bash on the go (see below), you can tell it to leave all `echo` commands such as this one out of your history by adding this to your `.bash_profile`:
  
    export HISTIGNORE=echo*

## GPG

Leiningen uses GPG for repo authentication, and it works fine in bash on Windows (you can just generate a key and use it to encrypt credentials following the [leiningen deploy instructions](https://github.com/technomancy/leiningen/blob/master/doc/DEPLOY.md#authentication), but that's no good if you're using Intellij or other Windows-based tools.  Install [GPG4Win](https://www.gpg4win.org/) and select GPA.  This gives you the `gpg` command in a regular cmd shell.  Fire up such a shell and do this:

     gpg --gen-key
     gpg --list-keys
     
That'll walk you through creating a key, and show it once you're done.  You'll need to put your leiningen credentials in a file called `%userprofile%/.lein/credentials.clj`, and it should look something like this:

    {#"my\.datomic\.com" {:username "conan@your-email-address.com" :password "your-password"}}

Put in here your plaintext username/password for each authenticated leiningen repo you use.  Then encrypt it with GPG, but GPG is so crap that there's a trick to it:

    gpg --default-recipient-self -e %userprofile%/.lein/credentials.clj > %userprofile%/.lein/credentials.clj.gpg
    File `credentials.clj.gpg' exists. Overwrite? (y/N)
    
When it asks you this, say no (n).  If you say yes it'll break.  If you say no, it'll ask you for a new filename:

        Enter new filename: credentials.clj.gpg.2
        
It's still gone and written a 0-bytes file called `credentials.clj.gpg`, which we have to delete, but it's also correctly written our `credentials.clj.gpg.2`, which we must now rename:
        
        del credentials.clj.gpg
        rename credentials.clj.gpg.2 credentials.clj.gpg

# JDK 8

You can't just install Oracle Java, instead you have to do this:

    sudo add-apt-repository ppa:webupd8team/java
    sudo apt update
    sudo apt install oracle-java8-installer
    sudo apt install oracle-java8-set-default
    
Change the 8s for 9s if you want Java 9 (may or may not apply to the repo name).

# Leiningen

Follow the [official instructions](https://leiningen.org/#install).

# IDE

[IntelliJ](https://www.jetbrains.com/idea/download/) offers the most sophisticated development experience on Windows, matched only by Emacs - but if you're using Emacs, are you sure you wouldn't prefer OSX or Linux?  I don't know much about Emacs and Clojure on Windows, but I know you'll want [Cider](https://github.com/clojure-emacs/cider) and some refactorings like [clj-refactor](https://github.com/clojure-emacs/clj-refactor.el).  Special mention also goes to [LightTable](http://lighttable.com/), which is great at Clojure but not so much other things; given that Clojure development requires you to work with Java or JavaScript at some point, it doesn't cut the mustard for me.  For now I'm assuming you're using IntelliJ.

## Cursive

Clojure development in IntelliJ is made possible by the fantastic [Cursive](https://cursiveclojure.com/).  Head over to the [Getting Started](https://cursiveclojure.com/userguide/) guide, which will explain how to download and install the IntelliJ plugin.  

### Structural Editing

If you're writing a LISP, you'll want Structural Editing (or Paredit, in Emacs parlance).  It's well worth reading the Cursive documentation, but pay special attention to the [Structural Editing](https://cursiveclojure.com/userguide/paredit.html) section.  Go to Settings > General > Smart Keys and in the top section set:

* Surround selection on typing quote or brace: `true`

Further down in the Clojure section set:

* Use structural editing: `true`
* Maintain selection when surrounding with braces: `false`

### Keybindings

You can apply some default keybindings for Cursive's operations by going to Settings > Keymap > Clojure Keybindings.  It doesn't let you edit individual keybindings here (you do that in the usual IntelliJ way), but it can apply a whole set of Clojure-related keybindings for you in one go.  I recommend starting with the Cursive ones; open the Binding Set dropdown, select Cursive and press Apply.  This is also a really handy place to find out what commands are available that relate to Clojure.  You can also find commands in the regular IntelliJ menus at Edit > Structural Editing and Navigate > Structural Navigation.  I can't recommend enough trying out _all_ of the commands available, as you're guaranteed to find some that are essential to your style of working.

### Default settings

Cursive makes some weird choices with default settings, like indenting comments by 60 characters.  Go to Settings > Editor > Code Style > Clojure and select the General tab.  Set these:

* Comment alignment column: `0`
* Default to only indent: `true`

The first will stop your comments being autoformatted away from the end of your lines, and the second relates to:

### Indentation

The "Default to only indent" setting above is a little tricky to understand.  Cursive allows you to specify a number from 0 to 9 that defines how many forms after the first should be aligned with each other.  0 means all of them, "Indent" means none of them.  By clicking on a function or macro name in a form in your editor, you'll get a little yellow lightbulb that lets you adjust the indentation parameter for that symbol.  I recommend starting with Indent as the default (using the setting above) and adjusting individual forms that you want different behaviour from.  It's best explained with examples. 

#### Indentation: 0

When using a threading macro, I want all forms nicely aligned with the first so it's easy to read the flow:

{% highlight clojure %}
(->> cars-map
     (group-by :manufacturer)
     count)
{% endhighlight %}

#### Indentation: Indent

When using a `let` block, I never want anything aligned with the first argument to `let`, which is a vector of binding forms:

{% highlight clojure %}
(let [x-coord 1
      y-coord 2]
  (println "x: " x-coord ", y: " y-coord)    
  [x y])
{% endhighlight %}

Here the `println` and the vector that we're returning from the block are nicely aligned with Clojure's idiomatic 2-space indent.  There's no need to push them across the page to align with the binding form that defines `x-coord` and `y-coord`.

#### Indentation: n

I must confess, I very rarely use anything but 0 or Indent.  `update-in` is a good example of where we want to align later:

{% highlight clojure %}
(update-in db [:user] assoc :email "milicent@example.org"
                            :fave-colour "green")
{% endhighlight %}

We don't want the `print` any more than two spaces indented, but it's nice to have the exception name aligned with the exception Class.  Of course in practice, we'd put both of those on the same line.  Like I said, I rarely use this type of indentation. 

### Figwheel

If you write ClojureScript, you're probably using [Figwheel](https://github.com/bhauman/lein-figwheel).  It's possible to run  Figwheel in an Intellij REPL, details [here](https://github.com/bhauman/lein-figwheel/wiki/Running-figwheel-in-a-Cursive-Clojure-REPL).
  
Here's the short version: you'll need to add a small file, set up a build config to run in a normal JVM and add a parameter.  

1. Create a file called `dev/repl.clj`, containing this:

{% highlight clojure %}
;; This does not require a namespace declaration. See the README or
;; https://github.com/bhauman/lein-figwheel/wiki/Running-figwheel-in-a-Cursive-Clojure-REPL
;; for more details.
(use 'figwheel-sidecar.repl-api)
(start-figwheel!)
(cljs-repl)
{% endhighlight %}

2. Go to run configurations and create a new local Clojure REPL, and choose "Use clojure.main in a normal JVM process".

3. Add `dev/repl.clj` to the Parameters of the run configuration.

Voila!  Now you can run Figwheel within Cursive's REPL environment.

## Line endings and file encodings

Don't forget to set your line endings to LF in File > Line separators, and set your file encoding to UTF-8 by going to Settings > Editor > File Encodings.  That'll stop your non-Windows colleagues getting upset because their tools can't cope. 
