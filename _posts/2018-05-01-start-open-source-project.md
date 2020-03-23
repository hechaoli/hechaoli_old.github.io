---
layout: post
title: Start an Open Source Project
subtitle: Personal Experience
tags: [Open Source, Github, Software Management]
---

# Overview
There are many excellent articles on how to start an open source project. For
example: [Starting an Open Source
Project](https://opensource.guide/starting-a-project/) gives many valuable
suggestions, which I won't repeat. I will only share my personal opinions and
experiences based on the project I recently started on
[Github](https://github.com/vmware/ovsdb-client-library).

# Why Open Source?
Because my project is open-sourced under my company's repo, I first need to
create a ticket and get it approved. And one of the most important questions is,
why do you want to open source it? I had to make up reasons to persuade the head
of the open source office that open-sourcing this project is necessary.
Honestly, I would be more willing to answer the opposite question: Why not open
source? In my opinion, there are two reasons that a person / organization does
not want to open source a project. First, the product is so good that they are
afraid of being copied by others. Second, the code is so crappy that they are
afraid of losing users after revealing the truth of their product. Apparently
MacOS is the former (XNU kernel is open-sourced now) and Windows is the latter.
Therefore, why not open source your project if you are a generous person who is
confident with your code quality?

# When To Publish?
You can always start an open source project and push the code to Github. For me,
I don't like to deliver half-baked work so I will only publish the code when it
is ready for use or if I have been using it for a while. In other words, all or
nothing. It may also be a good idea to publish a work in progress so that people
can input suggestions. After you finish coding, you may find out it is the
easiest thing in maintening an open source project!

# How to Choose a License?
You may not have even one line of code in your initial commit, but you have to
have a license with it. Chooing an open source license is really a headache.
Actually anything on the legal side is a headache. I really can't say much in
this part because I really don't want to mislead people. I've ever spent the
whole afternoon reading terms in popular licenses but still have no idea which
one to choose. Look at how many licenses are there!
[Open Source Licenses by Category](https://opensource.org/licenses/category). I
used this web site while choosing a license:
[https://choosealicense.com/](https://choosealicense.com/). On the home page it
shows 3 and I chose the one in the middle - Apache License 2.0. But the
company's legal team then suggested BSD-2. There are also thousands of articles
about how to choose a license, after reading which I am still confused. One
suggestion is that you find your favorite open source project or an open source
project that is closest to yours and use the same license as them. I hope there
is a software that we can use to choose a license in the form of a
questionaire. The software outputs the appropriate license for us based on our
answers. Maybe this can be your first open source project!

P.S. The only license is the Beerware License:

```
/*
 * ----------------------------------------------------------------------------
 * "THE BEER-WARE LICENSE" (Revision 42):
 * <phk@FreeBSD.ORG> wrote this file.  As long as you retain this notice you
 * can do whatever you want with this stuff. If we meet some day, and you think
 * this stuff is worth it, you can buy me a beer in return.   Poul-Henning Kamp
 * ----------------------------------------------------------------------------
 */
```
I would like to use a ICECREAM-WARE licence if possible.

# What to Put in README?
Generally speaking, people contribute to open source project that they care. In
other words, people first use a software and if they find an issue they will try
to fix it (contribute to the project). Thus, the README file must help us retain
users because they are potential contributors. Making it user-friendly should
have higher priority than making it contributor-friendly. In my opinion, in
README file you should only put the instroduction of what does your software do
and how to use your software. You may add a link to the detailed documentation
at the end but the main content has to be foolproof. When I opened a project
with a README with detailed introduction of the architecture and design
decisions  but not even one line mentioning how to use the software, I usually
start searching for an alternative which just provides a simple example of usage
in the README. I have to admit that the former README is very
contributor-friendly.  But it may scare off users in the first place. If your
software is really good, users are willing to explore more and try to find the
detailed documentation themselves. If you still wish to put thoese stuff, always
put them after the foolproof content. Somebody also puts FAQs in the README,
which are also helpfuly for starters. Checkout [Awesome
README](https://github.com/matiassingers/awesome-readme).

# What to Put in CONTRIBUTING?
Now it's time to make it convenient for the adorable contributors. As a
contributor, the following items are necessary in a CONTRIBUTING file ordered by
importance:

* How do I file a bug?<br>
After using your software, I found a bug and I want to report so I am here.
* How do I suggest a new feature?<br>
After using your software, I have ideas about new features so I am here.
* How do I set up the environment and run the tests?<br>
No body fixes my bug or fullfill my feature request, then I will just do it
myself. Please tell me how to do it. This should also be as simple as possible.
For example, a maven project can simply be built by `mvn clean compile` and be
tested by `mvn clean test`. If you have other steps to build the code or run the
tests, try to write the simplest script for it so that the beginers can run it
in one command instead of spending time on this initial step. It may also scare
off contributors if it is too complex to build and test.
* How do I push my code?<br>
My fix works well for me. Now I want to push my code and benefit the community.
What should I do? Typically, the workflow is, people fork your repo and push
code to the forked repo and create pull requests (PRs) against the orignal repo.
You may put related git commands here because you may not assume everybody is a
git expert.
* What is the code style?<br>
My code is rejected because it doesn't follow your code style. What is your
style? Though it is not critical, it is still important that people in the same
community follow the same standard. For me, I am too lazy to build my own
standard and I am willing to use any common standard that looks good to me. I am
now using [Google's style guide](https://github.com/google/styleguide). Though I
can't stand the 2-space indent in the first place, I still adapt myselft to it.

You can find more useful suggestions
[here](https://mozillascience.github.io/working-open-workshop/contributing/).

# What to Put in Wiki?
I highly recommend using Github Wiki. Though your project may have a cool web
site with detailed documentations in it, I still suggest to have it mirrored,
in Wiki. In this way, your project is self-contained on Github so that the
contributors does not need to switch between tabs while developing. As far as I
am concerned, a good Wiki should contain the following items:

* Getting Started<br>
This can be same as it in the README. The most common and basic use case should
be shown here.
* User Manual<br>
The user will have advanced requirements after using your software for a while.
They may consult this page to find detailed usages. For example, how to tweak a
parameter or how to enable/disable a feature. The user manual should contain all
the public APIs.
* Architecture and Design Decisions<br>
As the users become contributors, they want to know how does each component work
together and why the code is developed in this way instead of the other.
* FAQs<br>
Put frequently asked questions here so that you don't need to answer them again
and again.
* How to build and run the tests (Optional) <br>
You can put it eithe in the CONTRIBUTING file as we mentioned above or put it
here or both. Hopefully it is just one command.

# Other Tips

## Nice badges
It looks nice when there are shiny badges appeared on your README file. You can
find almost all available badges [here](https://shields.io/).

## Contiuous Integration (CI)
It is nice to set up a CI pipeline for your project so that the code won't break
because of stupid typos. Basically, it runs all unit tests and configuration
tests when ever you push a commit or when a pull request is created. You can
also set up regular build to make sure your code is stable. The build passing
badge can be your first badge! The CI I am using is [Travis
CI](https://travis-ci.org/). travis-ci.org is free for open source project and
travis-ci.com is for private or proprietary project.

## Code Coverage
One of the cool badges is the code coverage badge. It shows the percentage of
code that is covered by your tests. People will have more faith in your project
if they see a higher number. And this number can inspire you to write more tests
for your software. Try to get 100%! I use [coveralls](https://coveralls.io/)
together with [coveralls maven
plugin](https://github.com/trautonen/coveralls-maven-plugin) and
[cobertura](http://cobertura.github.io/cobertura/).

## Release
When you are confident with the reliablity of your code, you can consider the
first release. You can create a release on Github with your deliverables. For a
library that can be used in other projects, you can put your binaries on a
dependency management central repository such as the maven central repository.
For a maven open source project, the easiest way is to use [Open Source Software
Repository Hosting (OSSRH)](http://central.sonatype.org/pages/ossrh-guide.html).

# Conclusion
This is my first time launching an open source project and everything I said is
based on my personal experience. They are all fundamental stuff that I put
together for future reference as well as for people who may need. I could have
said something insane or something stupid. Please feel free to point them out
before misleading more people.

# Resources
[Understanding Open Source Software](https://www.websiteplanet.com/blog/what-is-open-source-software/)
