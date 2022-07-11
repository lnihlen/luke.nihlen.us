---
title: "Explorations Around Leaving GitHub"
date: "2022-07-10"
author: "Luke Nihlen"
description: "Sharing some observations around planning a move of my project repositories away from GitHub."
---

Between the relationship with
[ICE](https://www.theatlantic.com/technology/archive/2020/01/ice-contract-github-sparks-developer-protests/604339/) and
the commercial release of Copilot, it's time to [leave GitHub](https://sfconservancy.org/GiveUpGitHub/).

One benefit of laboring in obscurity is that a change in project hosting impacts almost nobody but me. As I start to
recruit other developers, however, the cost of this change will slowly start to creep up. Furthermore, adding things
like build automation will drive the pain of moving up, as I will likely have to refactor those integrations.

Things have been a bit bumpy so far in the search for a new home. GitHub is free, and while I'm willing to pay for a
reliable service, paying for an always-on virtual machine on a cloud provider seems like overkill for my humble
OSS purposes.

#### Self-Hosting at Home

Before GitHub, I self-hosted several repositories on homemade Linux servers, but I've been around enough real system
administrators to know I'm not one. Keeping a site secure and stable on the modern Internet is a full-time job, and one
neglects maintenance, upgrades, and backups at their peril. Time spent tinkering with a Linux computer in my basement is
project time I could have spent writing code.

#### Self-Hosting on Nearly Free Speech

I found a [blog post](https://www.noamross.net/2019/12/15/git-hosting-for-the-distraught-and-the-restless/) from Noam
Ross about setting up [Gitea](https://gitea.io/en-us/) on [Nearly Free Speech](https://www.nearlyfreespeech.net/)
(NSFN), the hosting site I use for all of my static HTML websites (including this one!). Things went pretty smoothly
following the guidance from Noam Ross's site until we got to the SSL setup. I think something has changed with the way
Gitea hosts static HTML files, adding an `assets/` path to the URL for any file, which breaks the request from NSFN to
Let's Encrypt. I tried a few different tricks, including forking the NSFN TLS setup script, but no dice. I posted a
question on the Gitea Discourse server, but so far, no dice.

I can think of possible ways I could likely fix that. However, I notice that the link to Noam Ross's Gitea instance is
now down. But the fact that I can't even get a simple question answered on the support forum does not bode well. What
happens when I'm stuck on a workflow-critical issue? Am I going to be able to get help?

#### SourceHut

The Software Freedom Conservancy article recommended [SourceHut](https://sr.ht/). I've set up an account and even
managed to get a [clone of Hadron](https://git.sr.ht/~luken/hadron) posted. I like a lot of what I've learned about
SourceHut so far. My primary concern is that I don't think I understand the intended patch workflow. I think SourceHut
supports the classical Linux kernel email-based workflow. That's a change from the Pull Request workflow that GitHub
uses.

I worry that having to learn a new workflow on a new development website would have a chilling effect on potential new
contributors. I would want to write some extensive documentation about the intended workflow.

#### WushNet

Another friend recommended [WushNet](https://wush.net/wn/home). They use it for private project hosting, opting to host
public projects on GitLab. I noticed WushNet bundles Git hosting with [Trac](https://trac.edgewall.org/) for project
management, which I have fond memories of from my self-hosting days. Unfortunately, their billing site is currently
down, preventing me from signing up to evaluate.

#### Decision Time

I plan to migrate a few other repositories to SourceHut, to try and get a sense of the workflow. I'll check back on
WushNet in a few days. Still, for my Open Source projects, I think SourceHut is likely my final destination, barring
some discovery of incompatibility or roadblock.