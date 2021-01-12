+++
title = "Fix your fraud problems with this simple trick"
date = 2021-01-13
+++

Despite your expectations I’m going to support that cheesy header with a real
talk! Fraudsters will hate you.

We have this site with classified ads in Ukraine, [OLX](olx.ua) and it is
completely plagued with spam and fraud. I create an ad to sell something I don’t
need anymore, and within a couple of hours I have some stupid fraud in my
inbox. What baffles me is that it’s very easy and simple to fix, and they do
nothing.

Back in 2006 I was involved in one of the forks of an open source project
[L2J](https://www.l2jserver.com/) (so-called “server emulator”), which was a
re-creation of the server part of then-popular game [Lineage
II](https://en.wikipedia.org/wiki/Lineage_II) of MMORPG genre.

A lot of the servers were plagued with bots (official servers too). There were
two variants for the bot software — a standalone one that communicated with a
server on its own, and another one that controlled an interface of an actual
client (in-game bot).

In-game bot required much more resources to run (basically, a separate computer
per bot), so the majority of the problems were created by standalones, you could
easily run ten of them from a regular computer.

We discussed this problem in a developer chat and had an idea — maybe these
standalone bots have some differences while talking to the server? We tried to
investigate that and indeed, the order of packets was completely different
between a standalone bot and a regular game client.

It was straightforward to detect and reject bots on connect, but then it would
be too obvious. So we started detecting them on connect, and disconnecting and
banning after some random time online, like 2..5 minutes.

Of course, generally bot developers aimed at official servers and couldn’t care
less about some anti-bot defense on free emulator servers, so we didn’t spend
too much brain activity on it. Nevertheless, it was committed into the
Subversion repository with a completely unsuspicious “small fix” message (open
source!).

It sounds a lot like security by obscurity, as I’m writing it now. I am not an
anti-fraud expert, however, it looks like it really is — we need to detect some
suspicious activities, and then stop them in a way that doesn’t immediately tell
fraudsters about it. Why? To make “debugging” around our defense a much harder
problem.

For example, if we needed to do better with those bots, one way to go was to
keep track of a cumulative bot time on a player’s account in a database and ban
it some time into the future. Maybe an hour or two, maybe a day, or even a week.

There is also a technique called “shadow ban”, that’s sometimes used on
forums. For a shadowbanned person everything looks the same, they can view
everything and post comments, just their responses are not visible to anyone
else (except moderators), so they’re not causing a stir on a forum, and do not
launch attacks onto it from other places, because they do not perceive it as an
attack on them.

We could have employed something akin to a shadow ban. Stop bots from interacting
with other users from time to time. Drop their packets a lot when they’re in a
battlefield, so they and their party-members end up being killed by mobs.

I don’t think it’s always possible to employ these delayed responses — anything
financial and we’re out of luck? I’m not familiar with that field, never worked
at a bank or a payment processor. Would be interesting to learn a bit about
that.

So, back to classified ads. They could raise their defenses a bit, and with a
clever shadow ban technique (mark messages as read when a recipient logs in,
etc) it would eliminate a lot of the fraudulent activity from the platform.

If a fraudster tries this and tries that and there is no apparent benefit and it
seems like everything works, but people just don’t fall for a scam — will they
continue to try more and more sophisticated attacks? To a certain extent, later
they’ll get bored with having no results.

We also need good anomaly detection mechanisms. It starts with good logging and
monitoring, and with user complaints, just like any other performance problem or
a bug. Of course, later you can start to employ different machine learning
techniques, but it shouldn’t be the first step, and you need to do it really
carefully.

It’s very maddening for legit users to fall prey to some stupid algorithm that
can be hard to fix. Gather more cases and re-train a model? Maybe employ
heuristics to fix some corner-cases? In any case, a model's decision should be
overridable by moderators.

Sometimes people will pierce through your defenses anyway, so you’ll have to
react quickly, identify new patterns, shadow ban them, etc.

Sometimes it can be fixed by a product change. For example, if there is some
kind of bonus system, make it so users can spend bonuses only on themselves,
ship products paid with bonuses to the existing address only (that has a history
of successful deliveries from a given account), etc.

Removing an incentive to fraud is the best strategy.

After running a service for some time you’ll get a grip on how these things are
done.

Even Twitter seemingly managed to remove that Bitcoin “ElonMuskque
giveaways”. What took them so long I don’t understand, but if even Twitter did
it — you can do it too.

Just do it.
