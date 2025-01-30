---
title: "TIL: Ghostty - a new and quite promising terminal emulator"
image: "/covers/drawings/20250118.jpg"
draft: false
date: 2025-01-18T22:30:00+01:00
tags: ["til", "tech", "tools"]
---

Hello there! If you've been following my blog, you might have noticed that I'm usually leaning towards longreads as the style of my posts. And while I think such posts are great in general, it takes forever to prepare, write and edit them before publishing, which ends up in the very infrequent and inconsistent blogging ratio on my end. 

Lately, I've come across the [“What to blog about”](https://simonwillison.net/2022/Nov/6/what-to-blog-about/) article by Simon Willison which, alongside other things, had a great insight about the “Today I Learned” type of short posts to share small bits of recently acquired knowledge with the audience. So, this is my attempt to incorporate that approach alongside the longreads that I will keep writing (I have a few topics planned, so stay tuned and [subscribe to my newsletter](https://mail.n0rdy.foo/subscription/form) to receive an email once I publish a new post). Let's gooooooo!

Like many other people in tech, I like trying new apps, libraries, languages, and do my best to stay curious and open-minded. At the same time, there are a few tools that I've been using for a while, and I'm carrying them with me since forever: for example, Firefox, as a default browser, Sublime as a default GUI text editor, etc. The same applies to the terminal app: since I switched to MacBook 6 years ago, I've been using iTerm2 with `oh-my-zsh` add-on and `robbyrussell` theme on top, and I've been happy even since. I appreciate the flexibility of its configuration, history-based commands' complete, possibility to scroll with the mouse/touchpad. Additionally, I've accumulated quire a decent number of aliases that I'd need to migrate, should I choose to move to a new terminal.

However, something has happened lately: the SSH-related security vulnerability has been [discovered and patched](https://iterm2.com/downloads/stable/iTerm2-3_5_11.changelog) in iTerm2 (if you are the user and haven't updated the version, I'd strongly encourage you to do that asap).

An immediate disclaimer: I have no negative emotions about the security vulnerabilities in the free open-source apps. I have a big respect and appreciation for the people who are willing to dedicate their free time to build and maintain those tools and share them with the world. In my case, it was just a trigger “hey, it's been a while since you checked the state of terminal emulators on the market” that sparked my curiosity. 

## Wind of change

I remember that [Julia Evans](https://jvns.ca/), whose blog I follow, mentioned a few time that she uses [Fish](https://fishshell.com/). Also, some days ago I came across [this post](https://fishshell.com/blog/rustport/) about Fish rewrite to Rust from C++, which sounds like a cool thing to do. However, I tried it some time ago, and while pretty neat, I wasn't convinced to switch to it completely.

At the same time, in the internal Slack of the company I work for, my colleague asked the security team whether we have any policies about the apps, as they'd like to start using [Ghostty](https://ghostty.org/) as their terminal emulator. I took a look at it, and it immediately caught my attention: a fresh look, a zero-config setup, platform-native UI (discovered in details in the “[Ghostty Is Native—So What?](https://gpanders.com/blog/ghostty-is-native-so-what/)” post by Gregory Anders) and GPU acceleration, and FOSS with very permissive MIT license ([here is the GitHub repo](https://github.com/ghostty-org/ghostty)). I googled the author ([Mitchell Hashimoto](https://mitchellh.com/)), and discovered that he is a co-founder of HashiCorp, that brought Terraform, Vargant, Consult, Vault, and others to the world. That's quite a list. And, last but not the least, [Zig](https://ziglang.org/) as the main programming language was an interesting factor as well. 

So, I ran `brew install --cask ghostty` (the command is from [here](https://ghostty.org/docs/install/binary#homebrew)), and gave it a try. 

![image](/images/drawings/20250118-0001.jpg)

The first thing that I obviously saw was the design, and it looked minimalistic, distraction-free, and, somewhat, fresh. I had to increase a font size a bit to match my comfort level, though.

![image](/images/screenshots/20250118-0001.png)

While design is an important part to some degree, there is something more that I've become observing and, therefore, liking lately: the reasonable default configs of the apps, which mean that the majority of the users will never need to mess with configs at all. Here is [a great post](https://arne.me/blog/we-need-more-zero-config-tools) by Arne about this trend which lists such tools like Fish (mentioned above), [Helix](https://helix-editor.com/), [Lazygit](https://github.com/jesseduffield/lazygit), [Zellij](https://zellij.dev/), [k9s](https://k9scli.io/), etc. And that a very user-friendly approach: install and use right away! I believe that Ghostty would be a good addition to the list. For example:

- all of my aliases and environment vars configured within the `~/.zshrc` were available right away, as Ghostty appears to be able to load them. So, I can keep typing `k` if I need to use `kubectl`, or `r` for `ranger`, and dozens of others more complex ones. That's very-very cool, as no copy-pasting required to make it work.
- history-based autocomplete is available outside the box
- syntax highlighting works by default
- familiar shortcuts, like `⌘ + T` to open a new tab, `⌘ + W` to close it, `⌘ + D` to split the window into 2 tabs.
- not exactly config-related, but the terminal feels extremely fast without any settings tinkering

There was a tiny detail that I found frustrating in iTerm2: when I pressed `⌥ + Left Arrow` (which is in my muscle memory), instead of jumping over the word on the left, the terminal typed `D` . The same attempt, but in the other direction, resulted in `C`. No biggies, but annoying when the command consists of many words. In Ghostty this worked as I expected it to: 5 points to Gryffindor!

I've mentioned above that I've been using `robbyrussell` (from [here](https://github.com/ohmyzsh/ohmyzsh/wiki/themes)) for my iTerm2 terminal setup, so I wondered if the same one is available. I was about to start googling the list of available topics, but I've noticed [a hint](https://ghostty.org/docs/features/theme#listing-available-themes) in the Ghostty doc to run `ghostty +list-themes` in the terminal, and here is what has happened:

![image](/images/screenshots/20250118-0002.gif)

Isn't that cool? I haven't found the `robbyrussell` theme there though, maybe it has a different name now?

One more thing that amazed me was the embedded dev tool, which uses the same key shortcut as Firefox and Chrome DevTools: `⌥ + ⌘ + I` :

![image](/images/screenshots/20250118-0003.png)

Honestly, I haven't come up with the use case for this feature in my daily terminal use, but it shows the level of attention to the small details Ghostty has to offer.

Ghostty seems to become a hyped thing lately (well-deserved, imho), as I see more and more content about it. Most of the people appreciate its performance, which I didn't have an opportunity to test properly yet, rather than trying to insert the text of this post into terminal to see how fast it renders it:

![image](/images/screenshots/20250118-0004.gif)

Not a rocket speed, but my MacBook is 6 years old, so hard to blame only Ghostty for that. iTerm2 was slightly slower. Please, let me know if you have tried the same on M1 or newer MacBook, I'd be curious to know the results.

## Instead of conclusions

As of today, I'm still in the testing Ghostty phase, but haven't had the need to return to the old terminal setup yet, which is a good sign.  However, that I might happen, as the terminal is still missing `⌘ + F` search functionality, which comes handy from time to time. There is an [open issue](https://github.com/ghostty-org/ghostty/issues/189) and [a message from the tool creator](https://github.com/ghostty-org/ghostty/issues/189#issuecomment-2558909414) that this is one of the top priorities, so I hope it's the matter of time when it is coming.

Issues-wise, I haven't noticed any, except that resizing window sometimes has a tiny lag, when the top bar is a few milliseconds behind the main window. Also, something that is outside the terminal standards, but I'd personally love to have is:

- `⌘ + A` to select the current command instead of entire window text (or with any other keybinding for this action)
- `⌥ + ⇧ + Left/Right Arrow ` to select the left/right word of the entered but not yet submitted command (another muscle memory thingy), but Ghostty types `D` instead

I'll update the post if I notice anything or change my mind and go back to the previous setup, so stay tuned. So far, only good vibes, so if you feel like trying something new, that can be a good place to start.

Have fun! =)

P.S. Something obvious, but need to mention it nevertheless: I'm not affiliated with Ghostty anyhow, and nobody asked me to test or review it, that's my honest opinion of the tool which I found worth sharing with the community. 

P.P.S. I hope you liked this style of short-ish (still working on the "short" part) posts. If you don't want to miss the future ones:


- follow me on [Bluesky](https://bsky.app/profile/n0rdy.foo), [Mastodon](https://mastodon.social/@n0rdy), or [Twitter](https://x.com/_n0rdy_)

 