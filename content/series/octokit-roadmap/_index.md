---
title: "The future of Octokit.net - and a potential roadmap to suit"
---

I've found myself back in charge of [Octokit.net](https://github.com/octokit/octokit.net)
after a significant hiatus, and I've been thinking about the long-term plans
beyond "keeping the lights on" maintenance. And while I'd love to spend more
time on it, this isn't going to happen any time soon.

Client libraries like Octokit.net need continual investment as the the API
itself changes or adds new features. It becomes especially challenging when the
maintainers are no longer using it directly, and lack the bandwidth to keep up
with these changes.

Despite my lack of significant free time, these remain my goals for the
Octokit.net project while I'm the lead maintainer:

 - 100% support of GitHub and GitHub Enterprise APIs
 - A pragmatic and sensible C# API for working with GitHub
 - Good range of documentation and examples
 - API updates are easy and boring for maintainers

And while we have a lot of features supported, there are pain points that
make ongoing development challenging:

 - the codebase has been active for 7-ish years, and is at a size and scale
   where new contributors may find it intimidating to navigate and identify
   where changes should be made
 - adhering to the conventions of the codebase is somewhat covered by automated
   tests, but still requires guidance and feedback when reviewing changes to
   reduce time re-working contributions
 - keeping up with API changes is time-consuming, and things can fall
   through the gaps unless we're carefully following changes on
   [developer.github.com](https://developer.github.com)
 - our integration tests can be used to verify the client against the GitHub
   API, but this is hard for contributors to run as it requires a test account
   and setting several environment variables. There are also GitHub Enterprise
   integration tests added by a core contributor, but I've not gotten around to
   setting up these myself.

Suffice to say, this is a lot of things to do. But as I started thinking about
the bigger picture and ideas that could improve this sitation from a different
perspective I found myself thinking a lot about this question:

> **What if we could get to a place where the codebase itself didn't matter
beyond the value it provided, and instead contributors could focus on the stuff
that you can do with the project?**

Initially I felt like this was unimportant, as I had a good idea of the things
I knew could be done better, but after several years of contributing to the
project I wanted to see how I could reinvigorate things rather than continuing
along the same path.

After being unable to shake this question for a couple of weeks, I started on
an adventure to see if I could answer that question...
