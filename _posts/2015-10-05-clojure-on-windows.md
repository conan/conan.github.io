---
layout: post
title: "Clojure On Windows"
date: 2015-10-05 13:30:00
redirect_from: /clojure/clojurescript/windows/2015/10/05/clojure-on-windows.html
categories: clojure clojurescript windows
comments: true
---

# Introduction
### Updated: 07/08/2019

Most Clojure developers use Linux or OSX, and the community is slightly biased towards those platforms.  Nevertheless, there are plenty of us who use Windows for whatever reason, primarily with the [Cursive plugin for Intellij](https://cursive-ide.com).  Clojure development on Windows is smooth, and this is a guide to get you started (and help me remember how to do it).  Note that a lot of this is not specific to Clojure.

You can read more about the choices made by the wider Clojure community in Cognitect's [State of Clojure](https://clojure.org/news/2019/02/04/state-of-clojure-2019) survey results.

This guide shows how to set up a Clojure development environment using Intellij/Cursive running in Windows and all command-line stuff done in the Windows Subsystem for Linux, using your favourite distro (I use Ubuntu).  There are a couple of other tools I use that are also mentioned.

# Terminal setup

The best solution is to install the [Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/install-win10), and to use it with [ConEmu](https://conemu.github.io/).
    
## Windows Terminal (Preview)

The Microsoft team have released a new [Windows Terminal](https://www.microsoft.com/en-gb/p/windows-terminal-preview/9n0dx20hk701) that supports tabs, resizing and lots of other nice things.  It's usable but **not ready for primetime** (too many crashes, copy/paste isn't quite sane yet, etc.). I expect it'll be my preferred solution by the end of 2019.

If you've already enabled WSL and installed a distro, it'll simply be available in the dropdown.  You can set the default by opening Settings and finding the `guid` of the profile corresponding to your distro (by looking at the `name` properties of the profiles) and pasting it in at the top under the `defaultProfile` property.

If you want a background image (instead of using Acrylic, the Windows transparency feature) then put an image (`background.jpg`) in `%LOCALAPPDATA%\Packages\Microsoft.WindowsTerminal_8wekyb3d8bbwe\RoamingState` and update the relevant profile with this:

    "backgroundImage" : "ms-appdata:///roaming/background.jpg",
    "backgroundImageOpacity" : 0.8,
    "backgroundImageStrechMode" : "fill",
    
I recommend darkening the image in an editor because the opacity will only lighten it.
    
## Colours and PS1

I can't be bothered to learn about ANSI escape sequences and XTERM colours and stuff.  Put this [.git-prompt.sh](https://gist.github.com/conan/880d7c676de65816fe02005bdb918a41) in your home directory and paste this in your `.bash_profile`:

    source ~/.git-prompt.sh
    PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[38;5;197m\]$(__git_ps1 " (%s)")\[\033[00m\]$ '

## ConEmu

Install [ConEmu](https://conemu.github.io/).  It provides a tabbed, split environment that can host any other consoles. It's worth having a look through all the settings as it's extremely customisable, but here are a few key ones. 

### Buffer size

You want a long buffer.  You can't have a infinite one like on Linux, but you can make it long in Settings > General > Size & Pos > Long console output.  Set it to `32766` (the maximum).

### One tab per group

When displaying multiple consoles in a window, I want them all also to be within a single tab.  Go to Settings > Main > Tab bar and check the "One tab per group" checkbox.

### Copy/Pasting

There are a number of settings related to copy/pasting behaviour, but there's one in particular that I find annoying.  ConEmu will pop up a warning menu if you paste more than 200 characters or an enter keypress by default, but you can turn these off in the Settings > Keys & Macro > Paste section under "Confirm <Enter> keypress" and "Confirm pasting more than X chars". 

### Set Bash for Windows as default terminal

Conemu now asks you which terminal you want to use when you first start it, so make sure WSL is installed and working first, and select it.  For reference (Conemu will set this for you), your `{Bash:bash}` command will look like this:

    set "PATH=%ConEmuBaseDirShort%\wsl;%PATH%" & %ConEmuBaseDirShort%\conemu-cyg-64.exe --wsl -cur_console:pm:/mnt

Go to Settings > Startup > Tasks and edit `{Bash::bash}` if it already exists (or press the + button at the bottom of the list if it doesnt and set the command in the big box to be as above). Set the task up to have the name `Bash::bash`, check all the boxes, set the task parameters to be:

    /icon "C:\Program Files\WindowsApps\CanonicalGroupLimited.UbuntuonWindows_1804.2018.817.0_x64__79rhkp1fndgsc\ubuntu.exe"
    
This will get you a nice icon.  If you've got a slightly different version of Ubuntu, the path might be slightly different; to find it, open the Start menu, search for "Ubuntu", right click on the Ubuntu command result and choose "Open file location".
    
Don't forget to press the "Save settings" button at the bottom.

_NOTE: Getting the current directory in the tab title for WSL Bash is a bit [more involved](http://stackoverflow.com/questions/39974959/conemu-with-bash-show-folder-in-tab-bar), and I've never got it working._

### Keyboard shortcuts

These are some keyboard shortcuts I use to duplicate what iTerm got me used to.  Edit them by going to Settings > Keys & Macro.

#### Split horizontal/vertical

iTerm got me used to splitting terminal windows horizontally and vertically, filling the other half with a duplicate terminal.  The user shortcuts are called "Split: Duplicate active 'shell' split to bottom" and "Split: Duplicate active 'shell' split to right.  I use `Ctrl+Shift+H` and `Ctrl+Shift+V`.

#### Clear buffer shortcut

I like how iTerm allows the current buffer to be cleared with `Alt+K`.  I can't reproduce this behaviour exactly here (something to do with [not clearing buffers of running programs](http://superuser.com/questions/898426/clear-console-buffer-in-conemu-with-cygwin)), but it is possible to clear the buffer with `Alt+K` when no other command is running, by going to Settings > Keys & Macro, and adding a shortcut in one of the Macro slots.  Set the Hotkey to be `Alt+K` and the Description to be:
 
     print("\e echo -e '\0033\0143' \n")  
     
Note that this actually clears the buffer, rather than just scrolling to the end of it, which is very useful when running tests and things.  What this does is type a command to clear the buffer and press enter for you, but you won't want this command clogging up your bash history.  You can leave all `echo` commands such as this one out of your history by adding this to your `.bash_profile`:
  
    export HISTIGNORE=echo*
    
## Git 

Conventional wisdom is to set `core.autocrlf` to `true` on Windows, but I think this is wrong.  The idea is that because you're on Windows, you need CRLF line endings.  I've never come across a Windows tool that doesn't correctly handle LF line endings (except notepad), but many Linux and OSX tools can't cope with CRLF; plus I'm in Linux for all terminal activities.  I always set

    git config --global core.autocrlf input
    
to make sure that I'm always using LF everywhere.  This is the only setup that doesn't give me occasional problems around line endings, and is much easier to think about - if it's not LF anywhere, on my machine or anyone else's, it's wrong.  Do this in both WSL and in cmd.

### Create ssh keys

GitHub explain [how to create ssh keys in Git Bash](https://help.github.com/articles/generating-ssh-keys/) better than I can.  Follow steps 1 and 2 of their guide - we'll sort out the ssh agent in a better way next.  Don't forget to add your new ssh key to GitHub or BitBucket or wherever if you use a centralised git repo; your new public key file is in `~/.ssh/id_rsa.pub`.

### Remember git ssh key passphrase with ssh-agent

Add `eval $(ssh-agent)` to your `.bash_profile`.  Then make the agent remember your git passphrase by creating an SSH config file:

    touch ~/.ssh/config
    chmod 600 ~/.ssh/config
    
Put this in it:

    AddKeysToAgent yes

This ensures that you'll only be prompted for your passphrase once per session, and credentials will be cached in the ssh-agent.  Alternatively, if you'd like to be prompted when you start a new terminal, add `ssh-add` to your `.bash_profile` on the line after the `eval $(ssh-agent)`.

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

### Beyond Compare (or other Windows GUI diff/merge tool)

I use [Beyond Compare 4](https://www.scootersoftware.com/download.php) for resolving git merge conflicts.  There's a good [blog post](https://www.sep.com/sep-blog/2017/06/07/20170607wsl-git-and-beyond-compare/) on how to set it up, but it's focused on solving the problem that you can't edit files on the wsl filesystem from Windows applications.  Now you can [access WSL files from Windows](https://betanews.com/2019/02/16/access-linux-files-from-windows/), but actually there's no need.  

Put your code in your Windows home directory (`C:\Users\conan\dev` in my case), and you can find it in WSL (at `/mnt/c/Users/conan/dev` for me), as your `C:\` drive is automatically shared at `/mnt/c` - I think other drives are similarly shared automatically.  Then Intellij, Beyond Compare and any other Windows tools you like can happily edit the files, as can WSL tools. 

_Note:_ if you're [connecting to a Windows Docker installation from inside WSL](https://nickjanetakis.com/blog/setting-up-docker-for-windows-and-wsl-to-work-flawlessly), you may have rebound your `C:\` drive share from `/mnt/c` to `/c`, in which case you'll need to change the command below.

_Note 2:_ WSL2 has file access performance optimisations that will improve access to files on the Linux filesystem, and for WSL2 Microsoft are recommending to [put your Linux files in your Linux root filesystem](https://devblogs.microsoft.com/commandline/wsl-2-is-now-available-in-windows-insiders/). WSL2 allows Windows to access those files safely, so this advice will change when WSL2 is released properly.

WSL can also [run Windows commands](https://docs.microsoft.com/en-us/windows/wsl/interop#run-windows-tools-from-wsl) from WSL now, so we have all we need.  

To set Beyond Compare as your merge tool, in WSL edit your `~/.gitconfig` file to have the following:

    [merge]
        guitool = bcomp
        tool = bcomp
    [mergetool]
        prompt = false
    [mergetool "bcomp"]
        trustExitCode = true
        cmd = '"/mnt/c/Program Files/Beyond Compare 4/BCompare.exe"' \"$LOCAL\" \"$REMOTE\" \"$BASE\" \"$MERGED\"

You can do similar for your difftool, I just use the terminal display for that personally.

To stop leaving `.orig` files around after merge, run this:

    git config --global mergetool.keepBackup false

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

To install JDK 8 in WSL (JDK 8 is reliable for Clojure):

## Oracle

    sudo add-apt-repository ppa:webupd8team/java
    sudo apt update
    sudo apt install oracle-java8-installer
    sudo apt install oracle-java8-set-default
    
Change the 8s for 9s if you want Java 9 (may or may not apply to the repo name).

## OpenJDK

    sudo add-apt-repository ppa:openjdk-r/ppa
    sudo apt update -y
    sudo apt install -y openjdk-8-jdk 
    sudo apt install ca-certificates-java
    update-ca-certificates -f

# Dependency & Build tools

## Clojure CLI

Follow the [official instructions](https://clojure.org/guides/getting_started#_installation_on_linux) to install the clojure cli tools in bash in WSL. Cursive will use its own, built-in version for now (see below), although the [Clojure CLI tools for Windows](https://clojure.org/guides/getting_started#_installation_on_windows) are coming.

## Leiningen

Follow the [official instructions](https://leiningen.org/#install) to install it in WSL.  You can install it in Windows as well if you want to be able to run it from a cmd terminal, which might be useful if you're doing things like user-agent testing with webdriver.

# IDE

* [IntelliJ](https://www.jetbrains.com/idea/download/) offers the most sophisticated development experience on Windows, matched only by Emacs. 
* [Emacs](https://www.gnu.org/software/emacs/download.html) or [Spacemacs](https://github.com/syl20bnr/spacemacs) - but if you're using Emacs, are you sure you wouldn't prefer OSX or Linux?  I don't know much about Emacs and Clojure on Windows, but I know you'll want [Cider](https://github.com/clojure-emacs/cider) and some refactorings like [clj-refactor](https://github.com/clojure-emacs/clj-refactor.el) 
* [VS Code](https://code.visualstudio.com/) keeps getting better, and has a great Clojure plugin called [Calva](https://marketplace.visualstudio.com/items?itemName=betterthantomorrow.calva)
* [Atom](https://atom.io/) GitHub's desktop editor written in JavaSCript, with Clojure support from [ProtoREPL](https://atom.io/packages/proto-repl)

For now I'm assuming you're using IntelliJ.

## Line endings and file encodings

Don't forget to set your line endings to LF in File > Line separators, and set your file encoding to UTF-8 by going to Settings > Editor > File Encodings.  That'll stop your non-Windows colleagues getting upset because their tools can't cope. Do this for every project, in particular try to remember to do this when you create a new one.  I tend to create new projects from inside WSL to ensure this is correct.  You can see what line endings you're using in the bottom right of Intellij's status bar.

## Cursive

Clojure development in IntelliJ is made possible by the fantastic [Cursive](https://cursiveclojure.com/).  Head over to the [Getting Started](https://cursiveclojure.com/userguide/) guide, which will explain how to download and install the IntelliJ plugin. 

### Structural Editing

If you're writing a LISP, you'll want Structural Editing (or Paredit, in Emacs parlance).  It's well worth reading the Cursive documentation, but pay special attention to the [Structural Editing](https://cursiveclojure.com/userguide/paredit.html) section.  Go to Settings > General > Smart Keys and in the top section set:

* Surround selection on typing quote or brace: `true`

Further down in the Clojure section set:

* Use structural editing: `true`
* Maintain selection when surrounding with braces: `false`

You can easily change Structural Editing style in the bottom-right corner of the Intellij window:

![image](https://user-images.githubusercontent.com/3037019/62624803-6a45fe80-b913-11e9-9030-198e10e0c8a2.png)

### Keybindings

You can apply some default keybindings for Cursive's operations by going to Settings > Keymap > Clojure Keybindings.  It doesn't let you edit individual keybindings here (you do that in the usual IntelliJ way), but it can apply a whole set of Clojure-related keybindings for you in one go.  I recommend starting with the Cursive ones; open the Binding Set dropdown, select Cursive and press Apply.  This is also a really handy place to find out what commands are available that relate to Clojure.  You can also find commands in the regular IntelliJ menus at Edit > Structural Editing and Navigate > Structural Navigation.  I strongly recommend trying out _all_ of the commands available, as you're guaranteed to find some that are essential to your style of working.

### Default settings

Cursive makes some weird choices with default settings, like indenting comments by 60 characters.  Go to Settings > Editor > Code Style > Clojure and select the General tab.  Set these:

* Comment alignment column: `0`
* Default to only indent: `true`

The first will stop your comments being autoformatted away from the end of your lines, and the second relates to:

### Indentation

The "Default to only indent" setting above is a little tricky to understand.  The indentation setting determines how many spaces are added at the start of a line when you add a line break inside a form. Cursive allows you to specify a number from 0 to 9 that defines which item after the first in a form should be used for alignment.  "Indent" means always to use 2-space indent; "0" means to align everything with the item at index 0 (i.e. the first) _after_ the first item in the form (the function or macro).  

By clicking on a function or macro name in a form in your editor, you'll get a little yellow lightbulb that lets you adjust the indentation parameter for that symbol.  I recommend starting with Indent as the default (using the setting above) and adjusting individual forms that you want different behaviour from. Cursive has a [handy animation](https://cursive-ide.com/userguide/formatting.html) showing where to find this setting. 

### Java classpath length workaround

The JVM on Windows limits the classpath length, after which any attempt to start a JVM will fail (and if you do it from IntelliJ, it will hang).  Clojure projects - especially ones combined with ClojureScript projects - can easily hit this limit in Cursive, as the path to each dependency can be quite long.  Cursive now has a workaround for this, and I recommend setting up every new build configuration to use it.  

Open up your build configuration, and you'll see a dropdown entitled "Shorten command line" (if you don't see it, you may need an EAP build of Cursive, instructions in the [user guide](https://cursive-ide.com/userguide/)). Select `classpath file` and Cursive will write the classpath into a file and use that, bypassing the classpath length limitation.

### Clojure CLI

If you're using tools.deps in a project with a `deps.edn` file, you'll need the clojure cli tools.  Always choose to let Cursive "Use tools.deps directly", and be sure to download and use the latest version (as it's all still in alpha).  You'll find the option at: Build, Execution & Deployment > Build Tools > Clojure Deps in the Intellij settings.

### Figwheel

If you write ClojureScript, you're probably using [figwheel-main](https://github.com/bhauman/figwheel-main), which replaces the deprecated [lein-figwheel](https://github.com/bhauman/lein-figwheel).  It's possible to [run figwheel-main in an Intellij REPL](https://github.com/bhauman/figwheel-main/blob/master/docs/docs/cursive.md) (and you can also [run lein-figwheel in an Intellij REPL](https://github.com/bhauman/lein-figwheel/wiki/Running-figwheel-in-a-Cursive-Clojure-REPL).
