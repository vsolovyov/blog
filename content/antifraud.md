+++
title = "On simple anti-fraud"
date = 2021-01-08
[extra]
draft = true
+++

We have this site with classified ads in Ukraine, [OLX](olx.ua) and it is completely
plagued with spam and fraud. I create an ad to sell something I don’t need
anymore, and within a couple of hours I have some stupid fraud in my inbox. What
baffles me is that it’s very easy and simple to fix, and they do not (cannot?)
do that.

Back in 2006 I was involved in one of the forks of an open source project
[L2J](https://www.l2jserver.com/) (so-called “server emulator”), which was a
re-creation of the server part of then-popular game [Lineage
II](https://en.wikipedia.org/wiki/Lineage_II) of MMORPG genre.

A lot of the servers were plagued with bots (official servers too). There were
two variants for the bot software - a standalone one that communicated with a
server on its own, and another one that controlled an interface of an actual
client (in-game bot).

In-game bot required much more resources to run (basically, a separate computer
per bot), so majority of the problems were created by standalones, you could
easily run ten of them from a regular computer.

We discussed this problem in a developer chat and had an idea - maybe these
standalone bots have some differences while talking to the server? We tried to
investigate that and indeed, the order of packets was completely different
between a standalone bot and a regular game client.

It was straightforward to detect and reject bots on connect, but then it would
be too obvious. So we started detecting them on connect, and disconnecting and
banning after some random time online, like 2..5 minutes.

Of course, generally bot developers aimed at official servers and couldn’t care
less about some anti-bot defense on free emulator servers, so we didn’t spend
too much brain activity on it. Nevertheless, it was committed into Subversion
repository with a completely unsuspicious “small fix” message.

It sounds a lot like security by obscurity, as I’m writing it now. I am totally
not an anti-fraud expert, however, it looks like it really is - we need to
detect some suspicious activities, and then stop them in a way that doesn’t
immediately tell fraudsters about it. Why? To make “debugging” around our
defense a much harder problem.

For example, if we needed to do better with those bots, one way to go was to
keep track about a cumulative bot time on a player’s account in a database, and
ban it some time into the future. Maybe an hour or two, maybe a day, or even a
week.

There is also a technique called “shadow ban”, that’s sometimes used on
forums. For a shadowbanned person everything looks the same, they can view
everything and post comments, just their responses are not visible to anyone
else (except moderators), so they’re not causing stir on forum, and do not
launch attacks onto it from other places, because they do not perceive it as an
attack on them.

We could have employed something akin to shadow ban. Stop bots from interacting
with other users from time to time. Drop their packets a lot when they’re in a
battlefield, so they and their party-members end up being killed by mobs.

I don’t think it’s always possible to employ these delayed responses - anything
financial and we’re out of luck? I’m totally not familiar with that field, never
worked at a bank or a payment processor. Would be interesting to learn a bit
about that.

So, back to classified ads. They could raise their defenses a bit, and with a
clever shadow ban technique (mark messages as read when a recipient logs in,
etc) it would eliminate a lot of the fraudulent activity from the platform.

...continue thoughts on how this (de-)incentivizes a fraudster
