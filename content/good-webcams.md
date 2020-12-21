+++
title = "Why can’t you buy a good webcam?"
date = 2020-12-22
[extra]
draft = true
+++

Some people are not satisfied with a webcam from their notebook or an old webcam
they bought many years ago.

My brother decided he wants to try streaming. Some people can just start and
other people want to research all the bits and pieces beforehand. Our chat
history is comparable in length to all my other chats combined, so I learned a
lot of it as well. The craft is full with small details and inconveniences.

He found that there are basically no good webcams. A lot of the time an image
quality is meh, and the rare presence of 4k is usually not better than 1080p,
because resolution is not quality. Some camera apparently does upscaling from
1080p to 4k in its driver, indicated by poor quality and high CPU load. He also
has a really good camera, Fujifilm X-T1. As any other good camera, it can't be
used as a webcam when connected to a USB port. Usually it's possible to buy an
HDMI-to-USB converter, but his camera can't output a live stream into HDMI, only
a recorded video.

So we had an idea - maybe it is possible to create a good webcam ourselves. Why
would anybody need that? Well, it’s almost 2021 now, streaming is still on the
rise, pandemic is going strong and video meetings are everywhere. People are
willing to spend some money, as indicated by countless videos on YouTube
regarding the problem.

Let’s start with defining what is a good webcam.

A good webcam, in my opinion, is a device that provides a decent sound quality,
at least on par with recent MacBooks, and an image quality on par with top
smartphones, such as iPhone 11/12, Huawei P40 Pro, etc.

If you’ve read at least a couple of articles on the topic of video production,
you will know that to make quality content from the technical standpoint you
need three things, in that particular order:

1. Sound
2. Light
3. Video quality

If smartphones have a good image quality, then we could probably manufacture a
webcam from smartphone parts that are cheap and easily available. How hard can
it be?

Let's discuss these properties and later how to manufacture it.

### Sound

If you have a recent MacBook, it provides a decent sound quality with its
three-mic array. I haven't researched it and don’t know much about other
notebooks' audio quality. If you have a desktop PC, you have to have some
external mic anyway.

If the web camera has shitty audio quality, you’ll have to buy additional
gear. When I researched the topic, one of the recently released webcams
(available from October 2020) with RRP of $180 was producing a sound as if a
person sits in a big plastic barrel with a closed lid. Their voice was very
muffled with lots of resonant rattles. That level of quality wasn’t acceptable
10 years ago, how this piece of gear could have been produced in 2020 escapes my
understanding.

There are many good enough USB mics for $50+, you also can buy a great XLR
microphone with $100, but then you’ll need an external sound card with phantom
power supply. A decent one can easily cost you $200.

With almost any of these options you’ll have to install and configure some
software that will provide sound level compression, noise suppression (because
your notebook fans are spinning like hell) and won’t produce clicks in the
process.

I would like my webcam to do this by itself, no rattling, minimized echo, etc.

### Light

A lot of what makes a picture great comes from lighting the scene in a correct way.

The simplest thing is to sit in a room so the window is in front of you and the
Sun isn’t shining into a lens from behind your shoulder.

Webcam cannot provide decent lighting capabilities by itself, so I won’t spend
much time here. There is a webcam by Razer with a built-in LED circle, but it’s
necessarily small, always shines from a webcam direction and is mostly a
gimmick.

We can rely somewhat on the good sensor here. If it provides good dynamic range,
HDR capabilities and good low-light performance, then a patch of sunlight on the
wall won't turn everything else black, or a poorly lit room won't turn into a
gray noisy picture with a bleak silhouette.

### Image quality

It’s impossible to get a good lens and image sensor into a notebook lid, because
lids are very thin. They’re often something like 4 mm thick all together, and
there is no escaping physics. Of course, it's possible to put a webcam into a
thick part below the lid, but then it'll be useful for expressing anger at our
colleagues with wide-open nostrils and not much else.

Maybe some crazy engineer in the future will develop an array of small sensors
with an array of tiny lenses and somehow combine all that into a good image
quality, but we don't have this tech yet. Any smartphone with a good camera has
that camera with an ugly protruding lens as a thickest part of the phone.

So, if you want something better than a notebook's webcam, you can go get some
Logitech webcam with an image quality going back to 2012. It’s still better than
a laptop, but nowhere close to current smartphones.

What if you want better still? The next option is to install an app on the phone
and use it as a webcam. This gives a very good image quality, but it’s very
inconvenient and in some applications doesn’t work at all. For some reason it
isn’t always presented in applications as an option when choosing a video
source.

I don’t want to constantly attach, detach, charge my phone and tune the position of
a small tripod. What’s the next option?

The next option is to just buy a mirrorless photo-camera with a big fat 4/3” or
even APS-C sensor. It will cost $700+ for a new one (you can buy used for
cheaper). You will also need some gear to hold the camera in place, as it won’t
just rest on the top of the screen like a regular webcam. You also need an
HDMI-to-USB video grabber, and you need to be sure that the camera you buy can
output full resolution live onto that HDMI. Also make sure that it's possible to
turn off all on-screen info like `1/100s F3.5 ISO 800`, which is useful when
you're shooting photos and videos through viewfinder, but not so much when
you're in a video meeting.

Of course, you will have a lot of control over the whole stuff - set a
diaphragm, get good bokeh, set ISO, set white balance, etc. Do you actually need
it? Well, some of you do, and a lot of you do not. I learned all that stuff when
I was 16, but now I’m pretty happy with how my phone chooses everything for me
and just does great photos.

All that sounds pretty involved, isn’t it? It is a lot of time to research what
you need, how to set it all up, and a lot of money too.

### Manufacturing it

So there is a market gap between so-so webcams for $100-200 and a full-blown
setup with a mirrorless camera, an external mic and lighting panels that will
cost a grand.

I thought that it is a good idea to create an "upmarket" webcam for $250-300
that will provide good audio and video quality, on par with current smartphones,
actually using the same parts. As smartphones are made in tens of millions, all
relevant parts should be cheap and easily obtainable.

I quickly contacted some people who could help me do the project - firmware
engineer, electric engineer and a company that does mechanical design.

What features are needed for a good webcam for streamers and video meetings?
Audio: good parts (mics and ADC), good mechanical design and assembly (so
nothing would rattle), and some post-processing. We certainly want some
beam-forming capabilities to be able to focus on the person in front of the
webcam, so we will need more than one microphone, maybe up to 4. These parts are
easy to obtain, there are many in stock, it's not a problem.

Even if I think that 4k is not needed, I’m pretty sure it’s really a requirement
to have in an upmarket webcam, as it will be really hard to sell a pricey webcam
that can only do 1080p.

Also would be very nice to have some neural network inference on the chip, so we
can run some AI stuff directly on the webcam without bothering a user’s computer
that can be busy with other things.

I easily found a three-piece assembly for the iPhone 11 Pro for fifty bucks,
two-piece from a Huawei P40 Pro for twenty five and a one-piece from P40 Lite for
ten dollars. That’s with a lens, and bought in retail. Does it mean it costs
peanuts (less than $5 for a sensor) to buy in bulk to produce my own webcam?
Well... Maybe. I can’t just go and buy a couple thousands of these sensors. I’m
pretty sure I can buy a couple thousands of replacement parts for phones, but
sensors themselves are not sold freely. It's easy to buy some "random" sensor
that was around for years and most certainly has "meh" quality by today's
standards, but we need a good one, with good dynamic range, HDR capabilities and
low-light performance.

I wrote to several vendors asking them what are the prices and conditions, and
basically their answer is either “we’ll sell it if you will buy hundreds of
thousands per year” or silence. Oh, and a lead time (from order to shipment) is
occasionally 16+ weeks.

I've found suitable System-on-Chips, and it's the same - if we want to have
hardware 4k encoder, it's either "sorry, it was a demo part and we don't do it",
or "we have limited support capabilities, so we can only sell if you will buy
hundreds of thousands, maybe you want some SoMs from our partners?". It is
actually possible to buy System-on-Modules even in very small quantities, but
it's going to be pretty expensive and will be good for a prototype, but for
production it will blow past our budget. System-on-Module will cost almost the
same as the web camera itself.

It means I cannot just build five hundred webcams, see how good they are selling
and build a couple thousands, and so on.

There is also a plastic injection step that requires upfront investment to
manufacture molds, and of course design and engineering costs. All together it
was tens of thousands of dollars. I was willing to risk it and was pretty sure
that in the worst case scenario it would be just a cost of my experience
building hardware product.

But with this new information it means I would need to pour a million dollars
(that I don’t have) to manufacture an obscene amount of devices that I don’t
know how to sell.

I thought about it for a couple of days and understood that I approached the
problem from the wrong angle at the beginning.

The actual problem is not how to build a good web camera. The real question is
how to sell it in quantity! Is it possible for me to sell hundreds of thousands
of pricey webcams per year? If I know I will sell it, I can find an investment
for that.

Let's describe my situation here. I’m a co-founder at prophy.science which is a
SaaS, and I work on it full-time. I can probably spend 20% of my time on
something else, but I certainly don’t want to abandon it completely. I also
have two small beautiful kids, and I really don’t want to move to Shenzhen to
organize a big production there.

### Going to market

Ok, what’s the current size of the market? Luckily, there are public companies
in the market, so we can go look at their reports to see if there is anything
meaningful. And there is, Logitech says they have about $130M in revenue from
sales of PC webcams. That’s... not much? I think they’re the biggest player in
the market by a long shot, and they sell something like a few millions of
webcams a year.

It means that I have to sell like 10% of the Logitech shipment size, right from
the start.

Sorry, what?

It’s an established player in the market with all the distribution in place,
with a very recognizable brand, and here am I, knowing nothing about how to sell
hardware products, and there is seemingly no way to slither into the market and
scale.

Well, maybe there is, but I don’t see it. Maybe I could pull it off by diving
deep down and going full-time on the problem for years, but it seems much more
risky now.

Also I should mention that this market is basically indefensible. What
prevents established players from doing pretty much the same, when they will see
that there is a good niche? They just go to the same suppliers, get same or
better parts/prices and crush a startup using their established distribution
channels.

I remember a staple phrase from a lot of interviews, something along the lines
of "If I knew how hard it would be, I wouldn't do it". Here we are, knowing how
hard it will be, and not doing it.

I'm just going back to my SaaS, which is much more defensible, thank you very
much.

Answering the question in title: you can't buy a good webcam because existing
players seemingly don't think it's a good opportunity, and for newcomers the
market is hard to enter.
