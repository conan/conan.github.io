---
layout: post
title: "Clojure On Windows"
date: 2015-10-05 13:30:00
redirect_from: /clojure/clojurescript/windows/2015/10/05/clojure-on-windows.html
categories: clojure clojurescript windows
comments: true
---

# Introduction
### Updated: 17/12/2019

Most Clojure developers use Linux or OSX, and the community is slightly biased towards those platforms.  Nevertheless, there are plenty of us who use Windows for whatever reason, primarily with the [Cursive plugin for Intellij](https://cursive-ide.com) (although the [Calva plugin for VS Code](https://marketplace.visualstudio.com/items?itemName=betterthantomorrow.calva) is getting much better, and VS Code has the advantage of being able to run inside WSL)  Clojure development on Windows is smooth, and this is a guide to get you started (and help me remember how to do it).  Note that a lot of this is not specific to Clojure.

You can read more about the choices made by the wider Clojure community in Cognitect's [State of Clojure](https://clojure.org/news/2019/02/04/state-of-clojure-2019) survey results.

This guide shows how to set up a Clojure development environment using Intellij/Cursive running in Windows and all command-line stuff done in the Windows Subsystem for Linux, using your favourite distro (I use Ubuntu).  There are a couple of other tools I use that are also mentioned.

# Terminal setup

The best solution is to install the [Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/install-win10), and to use it with the [Windows Terminal (Preview)](https://www.microsoft.com/en-us/p/windows-terminal-preview/9n0dx20hk701), which is still in beta but is already very good. Good alternatives can be found in [ConEmu](https://conemu.github.io/) (previously my terminal of choice) and [Hyper](https://hyper.is/).
    
## Windows Terminal (Preview)

* [Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/install-win10)
* [Windows Terminal (Preview)](https://www.microsoft.com/en-us/p/windows-terminal-preview/9n0dx20hk701)

The best thing is to install a WSL distro before installing Windows Terminal, and it'll simply be available in the dropdown; if not you can [add WSL to Windows Terminal manually](https://windowsloop.com/add-ubuntu-to-windows-terminal/).  You can set the default by opening Settings and finding the `guid` of the profile corresponding to your distro (by looking at the `name` properties of the profiles) and pasting it in at the top under the `defaultProfile` property.

### Settings

Settings are additive, so things you add to your user `settings.json` get added on top of the defaults; you can find the defaults by holding down `Alt` whilst clicking on the Settings option in Windows Terminal.  Here are some settings I like.

#### Keybindings

``` javascript
 "keybindings": [
    {"command": "copy", "keys": ["ctrl+c"]},
    {"command": "paste", "keys": ["ctrl+v"]},
    {"command": "newTab", "keys": ["ctrl+n"]},
    {"command": "newTab", "keys": ["ctrl+t"]},
    {"command": "splitHorizontal", "keys": ["ctrl+shift+h"]},
    {"command": "splitVertical", "keys": ["ctrl+shift+v"]}
 ]
```

#### Profiles

Note that the WSL startingDirectory can be set, but you have to use different syntax compared to profiles for cmd and PowerShell:

``` javascript
        {
            "backgroundImage": "ms-appdata:///roaming/background.jpg",
            "backgroundImageOpacity": 0.3,
            "backgroundImageStrechMode": "fill",
            "colorScheme": "Dracula",
            "guid": "{2c4de342-38b7-51cf-b940-2309a097f518}",
            "fontFace": "Consolas",
            "fontSize": 14,
            "hidden": false,
            "name": "Ubuntu",
            "source": "Windows.Terminal.Wsl",
            "startingDirectory": "//wsl$/Ubuntu/home/conan/dev",
            "useAcrylic": false
        },
```

#### Colour schemes

Here's a handy guide to [Rolling your own colour scheme](https://dev.to/teckert/roll-your-own-color-scheme-in-windows-terminal-466b). I use [Dracula](https://github.com/dracula/windows-terminal):

``` javascript
    "schemes": [
        {
            "background": "#282A36",
            "black": "#21222C",
            "blue": "#BD93F9",
            "brightBlack": "#6272A4",
            "brightBlue": "#D6ACFF",
            "brightCyan": "#A4FFFF",
            "brightGreen": "#69FF94",
            "brightPurple": "#FF92DF",
            "brightRed": "#FF6E6E",
            "brightWhite": "#FFFFFF",
            "brightYellow": "#FFFFA5",
            "cyan": "#8BE9FD",
            "foreground": "#F8F8F2",
            "green": "#50FA7B",
            "name": "Dracula",
            "purple": "#FF79C6",
            "red": "#FF5555",
            "white": "#F8F8F2",
            "yellow": "#F1FA8C"
        }
    ],
```

### Background Image

If you want a background image (instead of using Acrylic, the Windows transparency feature) then put an image (`background.jpg`) in `%LOCALAPPDATA%\Packages\Microsoft.WindowsTerminal_8wekyb3d8bbwe\RoamingState` and update the relevant profile with this:

    "backgroundImage" : "ms-appdata:///roaming/background.jpg",
    "backgroundImageOpacity" : 0.8,
    "backgroundImageStrechMode" : "fill",
    
I recommend darkening the image in an editor because the opacity will only lighten it.

## Mount point (for Docker)

WSL mounts your disk to `/mnt/c/`, but it would be nice if it was simply at `/c/` (and you'll need this if you want to run Docker inside WSL). Follow the instructions in the "Ensure Volume Mounts Work" section of this guide to [Using Docker in WSL](https://nickjanetakis.com/blog/setting-up-docker-for-windows-and-wsl-to-work-flawlessly).
    
## Colours and PS1

I can't be bothered to learn about ANSI escape sequences and XTERM colours and stuff.  Put this [.git-prompt.sh](https://gist.github.com/conan/880d7c676de65816fe02005bdb918a41) in your home directory and paste this in your `.bash_profile`:

    source ~/.git-prompt.sh
    PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[38;5;197m\]$(__git_ps1 " (%s)")\[\033[00m\]$ '
    
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

    {#"my\.datomic\.com" {:username "your@email-address.com" :password "your-password"}}

Put in here your plaintext username/password for each authenticated leiningen repo you use.  Then encrypt it with GPG, but GPG is so crap that there's a trick to it:

    gpg --default-recipient-self -e %userprofile%/.lein/credentials.clj > %userprofile%/.lein/credentials.clj.gpg
    File `credentials.clj.gpg' exists. Overwrite? (y/N)
    
When it asks you this, say no (n).  If you say yes it'll break.  If you say no, it'll ask you for a new filename:

        Enter new filename: credentials.clj.gpg.2
        
It's still gone and written a 0-bytes file called `credentials.clj.gpg`, which we have to delete, but it's also correctly written our `credentials.clj.gpg.2`, which we must now rename:
        
        del credentials.clj.gpg
        rename credentials.clj.gpg.2 credentials.clj.gpg

# JDK 11

To install OpenJDK 11 in WSL Ubuntu (different versions might have different needs):

## OpenJDK

    sudo apt update -y
    sudo apt install openjdk-11-jdk
    sudo apt install ca-certificates-java
    sudo update-ca-certificates -f

# Dependency & Build tools

## Clojure CLI

Follow the [official instructions](https://clojure.org/guides/getting_started#_installation_on_linux) to install the clojure cli tools in bash in WSL. Cursive will use its own, built-in version for now (see below), although the [Clojure CLI tools for Windows](https://github.com/clojure/tools.deps.alpha/wiki/clj-on-Windows) are coming.

## Leiningen

Follow the [official instructions](https://leiningen.org/#install) to install it in WSL.  You can install it in Windows as well if you want to be able to run it from a cmd terminal, which might be useful if you're doing things like user-agent testing with webdriver.

# IDE

* [IntelliJ](https://www.jetbrains.com/idea/download/) with [Cursive](https://cursive-ide.com/) offers the most sophisticated development experience on Windows, matched only by Emacs. Note that it's not free.
* [Emacs](https://www.gnu.org/software/emacs/download.html) or [Spacemacs](https://github.com/syl20bnr/spacemacs) - but if you're using Emacs, are you sure you wouldn't prefer OSX or Linux?  I don't know much about Emacs and Clojure on Windows, but I know you'll want [Cider](https://github.com/clojure-emacs/cider) and some refactorings like [clj-refactor](https://github.com/clojure-emacs/clj-refactor.el) 
* [VS Code](https://code.visualstudio.com/) keeps getting better (in part thanks to help from [Clojurists Together](https://www.clojuriststogether.org/)), and has a great Clojure plugin called [Calva](https://marketplace.visualstudio.com/items?itemName=betterthantomorrow.calva) - although it's still short in some areas, such as formatting.
* [Atom](https://atom.io/) GitHub's desktop editor written in JavaSCript, with Clojure support from [ProtoREPL](https://atom.io/packages/proto-repl); this might be losing traction however.

For now I'm assuming you're using IntelliJ.

## Cursive & Intellij

Clojure development in IntelliJ is made possible by the fantastic [Cursive](https://cursive-ide.com/).  Head over to the [User Guide](https://cursive-ide.com/userguide/), which will explain how to download and install the IntelliJ plugin. 

### Line endings and file encodings

Don't forget to set your line endings to LF in File > Line separators, and set your file encoding to UTF-8 by going to Settings > Editor > File Encodings.  That'll stop your non-Windows colleagues getting upset because their tools can't cope. Do this for every project, in particular try to remember to do this when you create a new one.  I tend to create new projects from inside WSL to ensure this is correct.  You can see what line endings you're using in the bottom right of Intellij's status bar.

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

If you write ClojureScript, you may be using [figwheel-main](https://github.com/bhauman/figwheel-main).  It's possible to [run figwheel-main in an Intellij REPL](https://github.com/bhauman/figwheel-main/blob/master/docs/docs/cursive.md) (and you can also [run lein-figwheel in an Intellij REPL](https://github.com/bhauman/lein-figwheel/wiki/Running-figwheel-in-a-Cursive-Clojure-REPL).

### Shadow-cljs

You can [Use shadow-cljs with Cursive](https://shadow-cljs.github.io/docs/UsersGuide.html#_cursive), I recommend going down the route of using a leiningen `project.clj` for dependencies as it works very well. Also, I like to be able to restart my entire cljs compile/watch/hot-reload with a single click (i.e. not connecting to a remote nREPL), so I set up a local clojure REPL build config using clojure.main and passing the path to the following script in the Parameters (this is based on the figwheel approach above, with help from [Thomas Heller](https://github.com/thheller) - thanks!):

``` clojure
(require
  '[shadow.cljs.devtools.api :as api]
  '[shadow.cljs.devtools.server :as server])
(server/start!)
(api/watch :app)
(api/repl :app)
```

It's noisy (complains about version mismatches on startup, these are warnings and can be ignored) but it compiles everything right there in your cljs REPL.
