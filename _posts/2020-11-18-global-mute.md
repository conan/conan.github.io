---
layout: post
title: "Global Mute for Zoom and Google Meet in Windows"
date: 2020-11-18 20:00:00
categories: tooling 
comments: true
published: true
---

Can't remember the mute shortcut?  Mute doesn't work because you're working in another window?  This post is for you.  I'll explain how to make all your mute keyboard shortcuts the same, and how to make them work all the time* - even when in another application.

*If you use Windows and Google Chrome. Coming soon: Slack, Microsoft Teams

# Zoom

Click on your profile picture in the top right and choose Settings, then head to Keyboard Shortcuts and set two things: the "Mute/Unmute My Audio" shortcut (I've used Ctrl+Shift+1) and be sure to check the "Enable Global Shortcut" checkbox:

![Zoom Settings](/assets/images/zoom-settings.png)

# Google Meet

Meet doesn't have built-in support for global keyboard shortcuts, so we need a [Chrome extension](https://chrome.google.com/webstore/detail/mute-for-google-meet/joahjeghhofndcnokdadmgjnoleckonf).  Add it to Chrome, then open the Chrome menu (three dots in the top right) and go to More Tools > Extensions:

![Chrome Extensions Settings](/assets/images/chrome-settings-extensions.png)

Paste [chrome://extensions/shortcuts](chrome://extensions/shortcuts) into chrome. Find the "Mute for Google Meet™" section.  Set the mute keyboard shortcut to the same one you chose for Zoom, and change it from "In Chrome" to "Global":

![Mute for Google Meet Settings](/assets/images/chrome-settings-extensions-mute.png)


