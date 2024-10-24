---
title: Why I view E2EE the same as critical infrastructure.
date: 2024-10-21
last-modified-at: 2024-10-21
tags:
  - controls
  - ramblings
---

This is just a compilation of a very long thread I made on Mastodon so I can share this better. For more context, this is all in reference to my replies to Soatok’s ongoing responses to the Matrix dev team re: Soatok’s responsible disclosure of security flaws. The TL;DR is Matrix was using an educational implementation of a thing, that explicitly said to not use it in production, and didn’t do anything about that for seven years until Soatok found it.

[Here’s a link to my first of the reply chain directly talking about that.](https://furry.engineer/@senil888/112964632462775370)

## I (kind of) worked in the power industry.

The main project I had with my (now former) company in my short time there was for a hydro dam up in the Idaho panhandle, in Boundary County - not joking, it's called that. That’s not too important, though.

What is important is that this fairly small hydro dam, which can output about 4 megawatts during peak spring runoff, helps power the nearby towns via a city-run utility. While it’s not the end of the world for the dam to be offline, them being offline means the utility has to buy even more power than if it was running. Which costs the city a fair bit more money than running the dam themselves, because you have the power generation places that need their money to run their facilities, the transmission companies that need to fund their projects, and on and on and on.

So, it’s pretty important to help them get back on their feet. And we had a big task - we were to replace their old Allen-Bradley PLCs with our own, and toss in a SCADA system too.

The SCADA system is fairly straightforward when it comes to data inputs - I could just take the data, label it, maybe do some conversions as-needed, and toss it up the chain. For this system, it went from generator-connected programmable logic controller (PLC) to a beefier controller running more sophisticated software, as well as a basic HMI, up to a control house computer that ran a COPA-DATA Zenon HMI that we also designed and developed. **We were responsible for a lot of stuff.**

{% figure "/assets/images/stickers/ScaredSenil.png", "Art by [Dragonjourney](https://x.com/Dragonjourney)" %}

## The Controls

The hard part comes in when we actually add in the controls. Which is, y'know, important since we’re replacing everything their old PLCs did with our own system and solution. Which meant we needed reference code, since the customer knows their system the best (mostly, we talked with the operators who knew how to run the thing, but didn’t 100% know the backbones). What we got? Was in ladder logic. Which looks very different to structured text, which is probably what you think of when I say “what does a programming language look like typed out?” So… not only did we need to re-create that, we had to “translate” it into ST as well. Fun!

This is where the controls joy comes in. Each PLC acts mostly the same when it comes to general operations. These outputs are for lights, this output needs to do this thing and get this signal for some time back before another thing can even think about asserting, etc.

Spinning up a hydro generator takes a decent bit of time. Not forever mind you, but there’s a lot of steps involved. First you need to get water to, well, the generator's turbine inlet. But you can’t just send water straight from the turbine inlet valve (TIV) to the turbines. You need to balance the water pressures between everything because if you don’t? Congratulations, you just broke the turbine and this generator has to be rebuilt.

Once the TIV bypass does its thing for a while, we can actually open the TIV and start to send water to the turbine. Once water is getting to the turbine, we open up the wicket gates - which actually finally let water to the turbines. They’re controllable, so we can define exactly how much water gets to flow, and thus how much power a generator actually outputs.

Then it’s the usual generator stuff. Get the thing up to speed and, once it’s up to speed and stays there for a little while, ya gotta sync the generator phases and frequency to the grid phases and frequency. If the phase angles are off too much, you end up kicking the generator forwards or backwards when the breaker closes to the grid. That can be violent, and harm the generator over time - or outright break it if the phase angle is off by too much. So we need that to be as precise as possible, and to close as soon as we’re confident it’ll safely kick in without breaking anything. So that’s the gist of the start-up sequence at the dam. All is well and good there.

## "Edge cases cannot exist."

However. There’s a fail over system with the Zenon HMI. See, the two powerhouse controllers gather all available data in this system as a form of redundancy - not via the physical network, but via the hardware. This is fine when it comes to inputs, since that version of Zenon can only listen to one controller at a time, but we run into problems with outputs. The "Zenon only talks to one controller at a time" becomes the enemy, in a way.

While realistically rare, or outright "would never happen", it is still technically possible for an operator to send a binary output command or update an analog setting *while Zenon changes which controller it listens to*. This can be for a variety of reasons - the operator told the controllers to switch over, Zenon lost access to one controller so it's starting to talk to the other (a process that takes a bit of time), etc. **The absolute last thing we need is for a command to be sent twice during this switchover,** because output data cannot be shared between the two because of the whole "Zenon can only talke to one controller at a time."

Our first main problem was fixing this control switchover problem, brought upon the very nature of our design scheme and how Zenon works (which I think they’ve fixed since that version, but I never worked on Zenon myself). Again, we have to account for every possible problem which means figuring out how to make sure a command sent from Zenon “syncs” between devices. The solution I came up with was pretty clunky, to be frank, but it worked - I literally just told the controller that knew it was talking to Zenon to periodically pulse the analog setpoint to the other controller, so it at least knew what was going on and could report back to Zenon appropriately.

What ended up happening was one generator - which was being rebuilt - wouldn’t report how far the wicket gates were open. I shouldn't need to say that "this is bad" because if we can't tell how far the wickets are open, **we have no idea how much water, if any, is getting to the turbine proper.**

{% figure "/assets/images/stickers/ConfusedSenil.png", "Art by [Dragonjourney](https://x.com/Dragonjourney)" %}

The sensor was a hall effect one, made of steel. There is a window above it. This is a dam, close to the river and a waterfall. I hope you can guess what the problem is. Hint, it involves steel not liking moisture. We cleaned out a COPIOUS amount of rust, and hey look the sensor was finally working again, we got wicket gate data. So we could finally spin up this generator. Which we were able to do…

Except, y'know. Problem time. The generator was vibrating, and vibrating a LOT. Consistently. Every time it started up, some code I adapted from the original ladder logic would run and trigger an emergency stop. The code runs on the controller directly talking to the generator, quietly monitoring vibration data. If ANY of the four input sensors report values over some threshold for more than a few seconds? Emergency stop time.

So something is still wrong with this generator. We trust the sensors since we’re getting good data as the generator spins up, it just… shakes too much. We actually temporarily disabled the logic that trips the generator to monitor its behavior if it didn’t e-stop in time. It kept vibrating, more and more. It would probably shake itself apart if you let it keep going. It also did it consistently, and it was one bearing’s sensor that was consistently tripping. *(aside: that bearing had also seized up TWICE during prior trips to commission this generator. It also did it again after we left the site. That bearing did not like existing, apparently.)*

The only reason we were able to check this case is because what I had written worked and we knew it worked. In part because that same logic was on three other generators, but we were able to accidentally prove that hey, if this happens, it’ll trigger an emergency stop no problem. It turned out the people rebuilding the generator didn’t balance the flywheel when they fixed the bearing before, so oops! You can’t spin that thing up just yet.

## Remember: Lives are at stake.

Imagine how bad this whole thing could’ve been if we weren’t as rigorous as we were. Generators could’ve broken apart. People could’ve been hurt. Expensive equipment could’ve gotten seriously damaged. Hell, it might not even have started up because of how much other equipment we had to interface with that we couldn’t investigate ourselves. That client trusted us to get their dam up and running with new hardware that would treat them well for many years to come, and to build a partnership with a firm they could trust. In the end, we delivered and they’re supposedly working on more (smaller) projects with them.

I don’t know the specifics, but BECAUSE we were persistent in making sure every edge case we could think of was resolved, and fixing everything in real-time as problems came up (and making sure our fixes wouldn’t introduce NEW problems) we were able to tell them “you can trust us.” Because ultimately, **“no one is obligated to buy from you. You can go out of business tomorrow if everyone decides to never work with you.”**

*(aside again: this is something I try to tell anyone who is running any kind of small business. Nobody is obliged to commission you, to hire you, to whatever. It's on you to stay on top of things and make people want to work with you, and keep working with you.)*

## This is still about Matrix, trust me.

It’s largely because of my experience with the hydro project that I just can’t trust companies, projects, and teams that hide vulnerabilities and failure points and hope nobody finds them. It makes me question how many they actually have hidden when someone does find one. How many defects does their thing have, if some of the most obvious ones are let through? If this obvious error got through their QA checks, what else is lying under the hood that we don’t know about yet?

{% figure "/assets/images/stickers/AngrySenil.png", "Art by [Dragonjourney](https://x.com/Dragonjourney)" %}

**Can I trust this system to be robust enough in the event of a catastrophic failure?**

I know cryptography - E2EE in particular - isn’t remotely comparable to what I’ve worked on, but what I’ve worked on is super important. Lives are at stake with these projects. I need to be confident that what I do isn’t going to hurt people. That it isn't going to break equipment. That it isn't going to fail in two years because of an oversight, *or something we knew about and ignored because it's "technically impossible".*

I need projects in critical fields that take themselves seriously to ACT like lives are at stake, because maybe some are.

But if you’re taking yourself seriously, in something as important as cryptography, something that is increasingly important every single day in every field - critical infrastructure included - you need to act like lives are at stake if you fuck up.

I know some of this shit is just a chat protocol, “why are you so pressed” blah blah blah. A lot of people herald E2EE services as a secure way to also talk about protests and labor organization without significant risk of getting caught. Of course, if you don’t regularly use E2EE normally the fact you use it might implicate you. When it comes to organizing? If a court requests data from some service, as little data as possible should be sent out. Because people get jailed or imprisoned for organizing. Those lives are on the line if your system isn’t up to snuff.

This isn’t to say “just pick any E2EE service, it’s fine” because it’s not fine to do that. People need a service they can trust, that’s reliable, that takes security seriously and does everything in their power to make sure what they do is as secure as possible. From what they know is a fail point, and what they later find out is a new possible fail point. People need services they can trust that has their best interests in mind. Clients need businesses they can trust that can hold up their end of the contract and deliver a quality solution that fits their client’s needs.

**Why should anything like this be any different?**

People die when we fuck up in the power industry. We can’t allow that to happen, ever. Our trust is built upon proving a track record of quality work that is done timely. Work that stands the test of time as systems operate. I know a lot of shit is nothing compared to critical infrastructure, but I wish more folks took it remotely as seriously as people in my industry have to. Because if we fuck up, people get hurt. People lose power. Facilities get damaged. Industry stops. And, ultimately...

 ***If the power goes out... so does everything else.***