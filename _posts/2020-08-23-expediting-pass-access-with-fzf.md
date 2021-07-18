---
title: "I bundled FZF with my password store flow. Why does one operate without it?"
author: ArcherN9
date: 2020-08-23 15:21:00 +0000
description: "*nix password manager is great. It takes away the necessity to remember individual passwords, its FOSS and builds on the Unix Philosophy. Can we take it a step further?"
category: [Productivity]
tags: [Unix, Linux, Mac, Command Line, FZF, Password Store]
---

This post requires familiarity with [FZF](https://github.com/junegunn/fzf) and
*nix [PasswordStore (Pass)](https://www.passwordstore.org/). Unlike my other
posts, I intend this to be a retrospective & my experience rather than a tutorial.

I may not have violated rule number one of password management by storing
passwords in plain text but perhaps rule number two: Maintaining the same password
for each of my accounts because I found it cumbersome & impossible to maintain
different passwords for each service I access. Sometime ago, I stumbled upon
`Pass`. It took away the complexity of memorizing passwords and stores them
securely as an encrypted file. I quickly migrated my 25 odd account passwords and
though I took sometime to consider the architecture by which I wanted to store
them, I realised that an approach of a narrow & deep and shallow & wide tree
hierarchy increased my access time alike. The tree below is indicative, not my
primary hierarchy. 

![Tree Hierarchy](/assets/img/2020-08-23-1.png)

My flow requires access to some of my passwords multiple times a day and due to
the frequency, I usually kept a separate terminal window just for accessing pass.
I ended up over complicating the one thing that pass attempted to simplify. Not
to mention it was more time consuming because now I had to remember bits and
pieces of the tree hierarchy that I had just created and navigate through it to
get to my password. Yes, with time I got the hang of it but the experience wasn't
as I hoped it'd be. Below is a benchmarking test of attempting to access 5 stored
passwords through the regular pass interface with a little help from ZSH. I
averaged **34 seconds**.

![Benchmarking old API](/assets/img/2020-08-23-2.gif)

When I found FZF, I immediately realised these two programs could be paired up
for faster access. After all, FZF simply fuzzy finds objects and returns them as
a string. I found an AUR package `pass-clip` by ibizaman - This is exactly what I
was looking for but with a more focussed approach. I stripped some elements from
the script to make it my own and extend support for OSX. The approach entailed
creation of a pass extension that:

* Traverses through the `.password-store` directory
* Identifies `.gpg` saved passwords 
* Strips the path to make it readable
* Lets the user fuzzy find passwords
* Executes the `pass show --clip` command on the user selection

The result was swifter access to one of my most frequently used resources and I
fail to understand why did I not use this from the beginning. As before, I
benchmarked this effort as well. I averaged just under **23 seconds**. I added an
alias `pc` to `pass clip` as a cherry on top.

![Benchmarking new API](/assets/img/2020-08-23-3.gif)

The `pass-clip` project with OSX support may be found at my
[Github](https://github.com/ArcherN9/pass-clip) and the original AUR project
[here](https://github.com/ibizaman/pass-clip).
