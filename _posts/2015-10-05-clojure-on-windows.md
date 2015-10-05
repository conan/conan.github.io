---
layout: post
title:  "Clojure On Windows"
date:   2015-10-05 13:30:00
categories: clojure windows
comments: true
---

Most Clojure developers seem to use Linux or OSX, and the community is heavily predisposed towards those platforms.  Nevertheless, there are plenty of us who use Windows for whatever reason.  Clojure development on Windows is surprisingly smooth, but here's a guide to get you started anyway (and help me remember how to do it).  Note that a lot of this is not specific to Clojure.


# RapidEE

The first thing I do when setting up a Windows dev environment is to install [RapidEE](http://www.rapidee.com/en/about).  It gives you a graphical interface to your Windows environment variables which isn't crap like the built-in one, and it's free.  If you like it, please consider [donating](http://www.rapidee.com/en/donation) (I have no affiliation with the project, I just think it's great).


# Terminal setup

I prefer a *nix-like terminal experience, but I recommend a similar setup if you want to use Windows cmd.

## Git Bash

I used to use Cygwin, but let's face it, nobody likes it.  [Git for Windows](https://git-for-windows.github.io/) is a much cleaner experience, and is sufficient for my needs.  If you want to use Windows cmd as your terminal instead of Git Bash, you should still install Git for Windows.  During the installation you'll get a choice to run git only from inside your Git Bash terminal, to add the git commands to Windows cmd as well, or to inject all the Bash stuff into your cmd.  I recommend the second option, as it lets you use git everywhere, but won't break anything.

_NOTE: Conventional wisdom seems to be to set `core.autocrlf` to `true` on Windows, but I think this is almost always wrong.  The idea is that because you're on Windows, you need CRLF line endings.  I've never come across a Windows tool that doesn't correctly handle LF line endings, but many Linux and OSX tools can't cope.  I always set my `core.autocrlf` to `input`, to make sure that I'm always using LF everywhere.  This is the only setup that doesn't give me occasional problems around line endings, and is much easier to think about - if it's not LF anywhere, on my machine or anyone else's, it's wrong._

### Create ssh keys

GitHub explain [how to create ssh keys in Git Bash](https://help.github.com/articles/generating-ssh-keys/) better than I can.  Follow steps 1 and 2 of their guide - we'll sort out the ssh agent in a better way next.  Don't forget to add your new ssh key to GitHub or BitBucket or wherever if you use a centralised git repo; your new public key file is in `~/.ssh/id_rsa.pub`.

### Remember git ssh key passphrase with ssh-agent

It's annoying to type your ssh key passphrase every time you do a git operation that contacts a remote repository, but it's even more annoying to configure Windows to remember the passphrase; luckily GitHub have done it for us.  Go to your Windows home directory (also your Git Bash home directory) and create a file called `.bash_profile` - you might have to use a terminal because Windows doesn't like letting you create files whose names start with a full stop.  Copy the [script to auto-launch the ssh agent](https://help.github.com/articles/working-with-ssh-key-passphrases/) and paste it into your `.bash_profile`.  When Git Bash next starts it'll ask for your ssh key passphrases, and remember it for future use.

### Ignore filemode

Windows doesn't understand file mode permissions in the same way as Linux or OSX.  Tell git to ignore them by running:

     git config --global core.filemode false

### Aliases

You've probably got your own favourite aliases for git commands.  Git supports aliases in your `.gitconfig` file, but this isn't as good as regular bash aliases, as you still have to type `git` at the start of each command.  Here are all my aliases, stick 'em in your `.bash_profile`:
 
    alias ll='ls -la'
    alias gs='git status'
    alias gl='git log --pretty=format:"%h%x09%an%x09%ad%x09%s"'
    alias ga='git add .'
    alias grp='git remote prune origin'

## ConEmu

iTerm is the feature I've always been most jealous of OSX users for.  Whilst the Windows 10 console is hugely improved (it now resizes and copy/pastes properly), for me there's no alternative to [ConEmu](https://conemu.github.io/).  It provides a tabbed, split environment that can host any other consoles. It's worth having a look through all the settings as it's extremely customisable, but here are a few key ones mainly useful for reproducing iTerm's behaviour. 

### Buffer size

You want a long buffer.  You can't have a inifinite one like on Linux, but you can make it long in Settings > Main > Size & Pos > Long console output.  Set it to 9999.

### One tab per group

When displaying multiple consoles in a window, I want them all also to be within a single tab.  Go to Settings > Main > Tab bar and check the "One tab per group" checkbox.

### Copy/Pasting

There are a number of settings related to copy/pasting behaviour, but there's one in particular that I find annoying.  ConEmu will pop up a warning menu if you paste more than 200 characters or an enter keypress by default, but you can turn these off in the Settings > Keys & Macro > Paste section under "Confirm <Enter> keypress" and "Confirm pasting more than X chars". 

### Add Git Bash as default terminal

Go to Settings > Startup > Tasks and press the + button at the bottom of the list.  This creates a new task.  Give it a name like "Git Bash", and set the startup command to be:
 
    "C:\Program Files\Git\bin\bash.exe" --login -i 
 
(or the equivalent if you installed somewhere else).  The `--login -i` makes the shell read your `.bash_profile` and starts it as an interactive shell.  

Using a name like "Shells:git" will cause your new task to be grouped with others in the menu, in this case the task is called "git" and will appear in the "Shells" folder.  I don't bother with this, but if you have loads of terminals then it might be useful.  Check the "Default task for new console" checkbox if you want Git Bash to be your default ConEmu terminal, but I recommend trying it out in ConEmu once before you do that - if the default terminal is broken then it's annoying to fix ConEmu, so check it works before setting it as default.

### Keyboard shortcuts

These are some keyboard shortcuts I use to duplicate what iTerm got me used to.  Edit them by going to Settings > Keys & Macro.

#### Split horizontal/vertical

iTerm got me used to splitting terminal windows horizontally and vertically, filling the other half with a duplicate terminal.  The user shortcuts are called "Split: Duplicate active 'shell' split to bottom" and "Split: Duplicate active 'shell' split to right.  I use `Ctrl+Shift+H` and `Ctrl+Shift+V`.

#### Clear buffer shortcut

I like how iTerm allows the current buffer to be cleared with `Alt+K`.  I can't reproduce this behaviour exactly here (something to do with [not clearing buffers of running programs](http://superuser.com/questions/898426/clear-console-buffer-in-conemu-with-cygwin)), but it is possible to clear the buffer with `Alt+K` when no other command is running, by going to Settings > Keys & Macro, and adding a shortcut in one of the Macro slots.  Set the Hotkey to be `Alt+K` and the Description to be:
 
     print("\e echo -e '\0033\0143' \n")  
     
Note that this actually clears the buffer, rather than just scrolling to the end of it, which is very useful when running tests and things.  What this does is type a command to clear the buffer and press enter for you, but you won't want this command clogging up your bash history.  Once you've got Git Bash on the go (see below), you can tell it to leave all `echo` commands such as this one out of your history by adding this to your `.bash_profile`:
  
    export HISTIGNORE=echo*


# Install JDK 8

Oracle seem to change their URL structure every week, so the [Oracle JDK 8 Download](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) link will probably be dead by the time you read this. Anyway, download it and install to the default location - you know the drill.  Update your `.bash_profile` with a `JAVA_HOME` environment variable like this:

    export JAVA_HOME=C:\Program Files\Java\jdk1.8.0_60

Add the same env variable to your Windows env variables using RapidEE, or by digging into Control Panel > System > Advanced System Settings > Environment Variables, or by some other route if your Control Panel uses the weird new categorised interface.  Also add `%JAVA_HOME%\bin` to your path.

# Install Leiningen

If you're using Clojure, you'll need [Leiningen](https://github.com/technomancy/leiningen), the excellent build tool used by almost all Clojure projects.  There is a [Windows installer](http://leiningen-win-installer.djpowell.net/), but I prefer to install manually - I think there's less fuss and we need to know what's going on.  Create a new folder in your home directory called `.lein`, and download the [installer batch file](https://raw.githubusercontent.com/technomancy/leiningen/stable/bin/lein.bat) into it.  This file is both the installer and the command to run - it has a `self-install` task that will download the latest version of leiningen.

Open a Git Bash terminal and run:

    cd ~/.lein
    ./lein.bat self-install
    
Now open a cmd terminal and navigate into the `.lein` folder.  We need to create a symbolic link to the `lein.bat` file so we can use it properly in Git Bash.  Run this:

    cd %userprofile%\.lein
    mklink lein lein.bat
    
The `mklink` command is the Windows equivalent of `ln` on Linux or OSX, and can be used to make [symbolic links](https://msdn.microsoft.com/en-us/library/windows/desktop/aa363878(v=vs.85).aspx), [hard links and junctions](https://msdn.microsoft.com/en-us/library/windows/desktop/aa365006(v=vs.85).aspx), but is not available in a Git Bash environment (although the links it creates work just fine).