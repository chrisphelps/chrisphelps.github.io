---
layout: post
title: Winter Tech Forum
category: conferences
tags: conferences, learning, languages, pony, elm
---

I recently attended the [Winter Tech Forum](http://www.mindviewinc.com/Conferences/JavaPosseRoundup/) in Crested Butte, CO. Rebranded in 2015 from its previous name, the Java Posse Roundup, this is my 7th consecutive time attending this OpenSpace conference.

The basic idea of an OpenSpace conference is to take what is often the best parts of a traditional conference - the impromptu "hallway conversations" - and make them the entire conference. Instead of presentations, all the conference sessions are group conversations, and the topics are decided by the attendees. The entire format ends up very self-organizing and free-ranging, and is a great way to share approaches and new ideas, to pick brains and try to explore your "how are others solving this?" questions.

The venue and size of this conference contributes to this. This year we had around 35 attendees, about a third of whom attended this for the first time. As a destination, Crested Butte is remote, leading to shared travel and fewer outside distractions, and small, allowing participants to meet up in minutes or even run into one another while out in town. The ski resort and other outside activities also allow for some non-software activities (where the conversations still continue).

# Session Highlights

We held discussion sessions on 4 days. Not including the opening and closing sessions, we had 10 session blocks, with two or more discussions to choose from during each block. We talked about APIs, immutable infrastructure, simple design, testing, monitoring and metrics, hiring, and outages, to mention just a few of the topics I attended. A few of my highlights and takeaways:

Designing and evolving good APIs is difficult. REST as a paradigm has served us well, but it has its limitations and can be difficult to evolve. Some interesting discussion in this space was around the use of alternative querying paradigms, which can solve both the problem of needing to make many requests to the backend, and alleviate some of the versioning difficulties. [Falcor](https://netflix.github.io/falcor/starter/what-is-falcor.html) and [GraphQL](https://code.facebook.com/posts/1691455094417024/graphql-a-data-query-language/) are two interesting ideas in this space. We also highlit that streaming APIs are a good alternative to pagination. A fun idea is to provide a '''wat''' endpoint which your customers can post back to - if you give them some response which they cannot handle based on your API, they can post back the response and the original request. This would allow customers to report problems in the response which did not cause server-side failures or logging.

Monitoring systems involves many problems, both in what to monitor, and in how to respond to outages. A key point from the [attendees](http://diannemarsh.com/conference-summary-winter-tech-forum-2016/) from [Netflix](http://netflix.github.io/) is that system monitoring metrics should be based on business metrics - service counts is very system specific, but for example, the trend  of how users interact with some interface can show whether the system is no longer servicing the business requirements. Each team must identify its own metrics that drive to those business metrics, and is responsible for an on-call rotation for their services. I shared how we do wargames (outage simulations) at Tendril, and got some good information on how to grow from there towards a full [Simian Army](http://techblog.netflix.com/2011/07/netflix-simian-army.html) style of outage resiliency and practice.

[We think we do hiring well at Tendril](http://www.slideshare.net/ChrisPhelps2/what-makes-a-great-developer-develop-denver-2015), but it was very interesting to talk to others about their hiring practices and things we might consider. Whether to give a take-home assignment or just do coding exercises in person, how much to rely on work in the open, and how to help put the candidate at ease are all interesting facets. We are generally in agreement that we want the content of the interview to match the job, that we often want to hire for the [growth mindset](http://mindsetonline.com/whatisit/about/) (including priming the candidate into that mindset during the interview), and to try to have a diverse pipeline of candidates and minimize bias in order to achieve a diverse workforce. It was observed that "interviewing today is more like auditioning" [@GordonWeakliem](https://twitter.com/GordonWeakliem) but not clear that we think that's a bad thing.

# Hack Afternoons and Hack Days

In the afternoons, there is free time for outdoor activities, individual time, or some hands-on hacking. Additionally, we had a whole day hack session on Wednesday.

On Monday I experimented with [Finch](https://github.com/finagle/finch), a Scala web framework from Twitter, built atop [Finagle](http://twitter.github.io/finagle/). I'd heard of Finch, but since we use [Finatra](http://twitter.github.io/finatra/) at work, I had not taken much of a look yet. My [experiment](https://github.com/chrisphelps/atticus) was interesting, but not horribly compelling. I liked the auto-derivation of JSON parser typeclasses that you can take advantage of if you use [Circe](https://github.com/travisbrown/circe) but otherwise didn't find anything that would make me migrate my Finatra apps to Finch. I liked it well enough, though, so I'd probably give it more of a shake for something new.

On the Wednesday hack day, I worked with a group experimenting with a new language called [Pony](http://www.ponylang.org/). Pony is based on the actor model, and has an interesting feature called ''capabilities'' which describes explicitly how objects and actors can interact with a reference. For example, you can use the ''transition'' capability '''trn''' to indicate that you will read and write to a variable and provide read-only copies for other objects to reference. As part of our [hack day project](https://github.com/WinterTechForum/luktnypon) we built a [Slack](https://slack.com/) listener, some JSON traversal, and some simple encoder/decoder logic, in order to build a few bots which interact in a Slack channel, encoding and decoding messages, and otherwise paying attention to various keywords.

Other hack projects (which were presented Wednesday evening) included a [whisky-serving robot](https://www.youtube.com/watch?v=jdeE-XTgW8I) built with Arduino and Raspberry Pi, a [Kubernetes](http://kubernetes.io/) cluster setup, and a start on a [Rustport of HdrHistogram](https://github.com/ogeagla/HdrHistogram_Rust).

On Friday afternoon I worked with a large group checking out [Elm](http://elm-lang.org/). I had [done a little with elm before](http://chrisphelps.github.io/learning/2016/02/23/Learning-7-More-Languages/) so I mostly helped others get started, but I spent a little time trying to [hack an Elm example to post to Slack](https://github.com/chrisphelps/treehorse/blob/master/neigh/Neigh.elm). I deliberately hacked this out by making the minimum changes to the example, without really trying to go too deep into understanding what I was doing, as this is exactly the opposite of my usual process. This was quite interesting, but my feeling of rushing and not understanding proved ''why'' this is not usually my process.

# Lightning Talks

I gave two lightning talks at this year's conference, as well as helping present our hack day project. In previous years, lightning talks have required much more preparation for me, but we have been doing them at work and I have gotten much more comfortable at preparing talks. For the first talk, I took the challenge of learning Finch and presenting on it the same day. For the second talk, I co-spoke with [Jack Leow](https://twitter.com/jackgene) on [Cucumber BDD](https://cucumber.io/), after we discussed it at dinner with another attendee about an hour and a half previously. Both talks went well and provided a nice challenge.

# Overall

This is a great conference. I feel like I get more out of it every year. It's great to reconnect with old friends and meet new ones. If you get a chance to attend next year, give it a go!




