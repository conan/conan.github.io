---
layout: post
title:  "Clojure On Windows"
date:   2015-10-05 13:30:00
categories: clojure windows
comments: true
---

Most Clojure developers seem to use Linux or OSX, and the community is heavily predisposed towards those platforms.  Nevertheless, there are plenty of us who use Windows for whatever reason.  Clojure development on Windows is surprisingly smooth, but here's a guide to get you started anyway.  Note that a lot of this is not specific to Clojure.

# Install JDK 8

Oracle seem to change their URL structure every week, so the [Oracle JDK 8 Download](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) link will probably be dead by the time you read this. Anyway, download it and install to the default location - you know the drill.

# Terminal setup

I prefer a *nix-like terminal experience, but I recommend the same setup either way.

## ConEmu

iTerm is the feature I've always been most jealous of OSX users for.  Whilst the Windows 10 console is hugely improved (it now resizes and copy/pastes properly), for me there's no alternative to [ConEmu](https://conemu.github.io/).  It provides a tabbed, split environment that can host any other consoles. It's worth having a look through all the settings as it's extremely customisable, but here are a few key ones mainly useful for reproducing iTerm's behaviour. 

### Buffer size

You want a long buffer.  You can't have a inifinite one like on Linux, but you can make it long in Settings > Main > Size & Pos > Long console output.  Set it to 9999.

### One tab per group

When displaying multiple consoles in a window, I want them all also to be within a single tab.  Go to Settings > Main > Tab bar and check the "One tab per group" checkbox.

### Copy/Pasting

There are a number of settings related to copy/pasting behaviour, but there's one in particular that I find annoying.  ConEmu will pop up a warning menu if you paste more than 200 characters or an enter keypress by default, but you can turn these off in the Settings > Keys & Macro > Paste section under "Confirm <Enter> keypress" and "Confirm pasting more than X chars". 

### Default terminal

I use Git Bash as my default terminal, so I want that when I start ConEmu or open a new ConEmu window.  You can add extra terminals into ConEmu (see below), and then set the default one by going to Settings > Startup > Tasks, clicking your fave terminal and checking the "Default task for new console" checkbox.

### Keyboard shortcuts

These are some keyboard shortcuts I use to duplicate what iTerm got me used to.  Edit them by going to Settings > Keys & Macro.

#### Split horizontal/vertical

iTerm got me used to splitting terminal windows horizontally and vertically, filling the other half with a duplicate terminal.  The user shortcuts are called "Split: Duplicate active 'shell' split to bottom" and "Split: Duplicate active 'shell' split to right.  I use `Ctrl+Shift+H` and `Ctrl+Shift+V`.

#### Clear buffer shortcut

I like how iTerm allows the current buffer to be cleared with `Alt+K`.  I can't reproduce this behaviour exactly here (something to do with [not clearing buffers of running programs](http://superuser.com/questions/898426/clear-console-buffer-in-conemu-with-cygwin)), but it is possible to clear the buffer with `Alt+K` when no other command is running, by going to Settings > Keys & Macro, and adding a shortcut in one of the Macro slots.  Set the Hotkey to be `Alt+K` and the Description to be `print("\e echo -e '\0033\0143' \n")`.  Note that this actually clears the buffer, rather than just scrolling to the end of it, which is very useful when running tests and things.

## Git Bash

I used to use Cygwin, but let's face it, nobody likes it.  [Git for Windows](https://git-for-windows.github.io/) is a much cleaner experience, and is sufficient for my needs.  If I ever do something more serious, I'll use a Vagrant VM for it.  During the installation you'll get a choice to run git only from inside your Git Bash terminal, to add the git commands to Windows cmd as well, or to inject all the Bash stuff into your cmd.  I recommend the second option, as it lets you use git everywhere, but won't break anything.

### Add Git Bash to ConEmu

Fire up ConEmu, go to Settings > Startup > Tasks and press the + button at the bottom of the list.  This creates a new task.  Give it a name like "Git Bash", and set the startup command to be `C:\Program Files\Git\bin\bash.exe` (or the equivalent if you installed somewhere else).  Using a name like "Shells:git" will cause your new task to be grouped with others in the menu, in this case the task is called "git" and will appear in the "Shells" folder.  I don't bother with this, but if you have loads of terminals then it might be useful.  Check the "Default task for new console" checkbox if you want Git Bash to be your default ConEmu terminal, but I recommend trying it out once before you do that - if the default terminal is broken then it's annoying to fix ConEmu, so check it works before setting it as default.

### Create ssh keys

GitHub explain [how to create ssh keys in Git Bash](https://help.github.com/articles/generating-ssh-keys/) better than I can.  Follow steps 1 and 2 of their guide - we'll sort out the ssh agent in a better way next.

### Remember git ssh key passphrase with ssh-agent

It's annoying to type your ssh key passphrase every time you do a git operation that contacts a remote repository, but it's even more annoying to configure Windows to remember the passphrase; luckily GitHub have done it for us.  Go to your Windows home directory (also your Git Bash home directory) and create a file called `.bash_profile` - you might have to use a terminal because Windows doesn't like letting you create files whose names start with a full stop.  Copy the [script to auto-launch the ssh agent](https://help.github.com/articles/working-with-ssh-key-passphrases/) and paste it into your `.bash_profile`.  When Git Bash next starts it'll ask for your ssh key passphrases, and remember it for future use.

