+++
title = "The Owl, The Scientific Method, and Claude Code: A Debugging Story"
date = 2025-10-23
+++

I was trying to update a dependency in my project. For the third time already!

Previous attempts over the couple of months were not successful, as I tried to upgrade too many libraries at once (sometimes it works, ok?). I won't bother you with the details of the previous attempts, short story is that it was breaking transaction isolation in the tests that we achieved via nested transactions.

This time I was very methodical, and as the starting point I made my code version-agnostic on a superficial level, so I can easily switch between working and broken versions. Because of course the dependency update introduces an unrelated breaking change, as Python community didn't learn too much from the whole 2->3 upgrade mistake.

After that I bisected and found a specific commit in the dependency that was breaking my code. It's a 1595-line behemoth of a rewrite that is described by the commit message "draw the rest of the owl". I hope you drew that owl really beautifully, pal.

Didn't really want to spend a couple of days understanding this nocturnal bird of prey. So I launched Claude Code, explained the situation, gave the diff, made it launch tests with the good commit, then with the bad commit, to convince it that it is indeed the dependency upgrade is messing things up, and we don't need to rewrite tests "to accommodate current system behaviour" or whatever. You know how it likes to fail.

Claude Code tried this and that approach, and, as is quite usual, started getting lost in the woods. At some point some idea gets stuck in its head and it tries to bulldoze through to make it work. Unsuccessfully, of course, because the idea is wrong.

At this point I was starting to get frustrated, but then I remembered my own advice that I gave many years ago on talks about how to debug properly.

**Just use the scientific method and write down everything!**

Which I immediately communicated to Claude with: "Write what we're trying to do, all hypotheses, supporting and conflicting evidence into a file `wip.md`."

They are the magic words for sure. It wrote five different hypotheses with pros and cons, and for one of the hypotheses there were no cons. It was even marked with caps as ROOT HYPOTHESIS. But after compaction it's not The One True Idea in its head - it's just one of many! And what we're trying to do with a hypothesis when we don't have conflicting evidence? We certainly don't try to write 150 lines of code imagining that it's true. We do a minimal experiment to try to falsify the hypothesis and more observations.

Now it goes much more smoothly, and we pretty quickly find what was the actual problem: it was Object Oriented Programming with multiple inheritance. Method Resolution Order (MRO) analysis revealed that one thing created a wrapper class that overrode a method from other thing, and this wrapper creation was conditional on just passing some parameters inside it.

What's interesting, is that with all the OOP, multiple inheritance, overrides and indirection, even Claude was having trouble producing a fix. In the end I just wrote a short test-specific function that's used only in our tests and configures things correctly, bypassing that OOP kerfuffle.

Three unrelated observations here:

1. It seems like Python community grew older, but didn't mature, did not become wiser. Nowhere near Clojure level.
2. Today I also learned that a group of owls is called a "parliament". If we call each class an owl, then our OOP code becomes Parliament Of Owls (POO).
3. Use scientific method for debugging. Does not matter if it's a human-craft debugging session, or an LLM-based one. Form hypothesis, observe, and experiment. And write **everything** down.
