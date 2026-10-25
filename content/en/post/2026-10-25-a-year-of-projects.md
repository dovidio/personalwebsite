---
title: "A year of projects"
date: 2026-10-25T08:00:00+02:00
draft: false
description: "Seven side projects built between a nappy change and a feeding session, and the year AI finally got good."
tags: ["projects", "ai", "fatherhood"]
translationKey: "a-year-of-projects"
---

Oh boy, it's been a year since I wrote my last blog post.
Don't scold me, I have a good excuse. The day after I wrote [my last blog post](/post/2025-10-28-talos-hetzner-part-3/), I became a dad.
I was prepared to sacrifice time for my hobbies, but didn't think it would be this hard.
Turns out it really takes a village!

But let's stop singing praise to the joy of fatherhood, this is a sort of technical blog after all (or it tries to be).
So it turns out that AI [finally got good](https://x.com/dhh/status/2007504187568074843). It went from generating a time-tracking app with the brilliant strategy of `setInterval(() => secondsTracked += 1, 1000)` to generating coherent code, faster than a human can produce and arguably more correct (if your domain was ingested in the training set).

Even though we had less time to devote to projects, it was a productive year for both my wife and me.
And it's time to celebrate all those little projects that filled our time between a nappy change and a feeding session.

[Labora](https://getlabora.app): This was entirely conceived and developed by my wife with AI assistance. It helps track contractions and gives hints on when it's time to run to the hospital. Its simplicity and UX have helped 3,000+ moms deliver babies to the world.

[Innuce](https://innuce.app): A YouTube summarizer Chrome extension and web app. It was kickstarted by my wife, and I took it over during the Christmas holidays, in an [AI psychosis](https://lucumr.pocoo.org/2026/1/18/agent-psychosis/) episode. We thought we were going to make a company out of this one :D
We are kind of proud of the result, and we keep using it in our daily life, but it did not exactly generate any traction. It didn't help that YouTube itself created a summary function on its website. Btw, at some point I'll probably open-source it and take it down.
I learned a lot from this. A few things:
1. Chrome extension development.
2. CloudFormation and AWS thingies (Cognito, Lambda, CloudWatch, DynamoDB)
3. Ruby on Rails, where I decided to move after being burned out by the AWS thingies
4. The beauty and the business of rotating proxies, client pooling
5. SSE, WebSockets and connecting to agent APIs (using OpenRouter at first and going Mistral eventually)

[Zenith](https://dovidio.github.io/zenith/): One of the most recent ones. The idea is that since we write less code (or none), we should rethink code editors and optimize them for reading, not editing. As a matter of fact, Zenith is a read-only code navigator.

[Colino](https://getcolino.umberto.xyz): Half-baked terminal-based news aggregator. It was one of the first projects where I heavily used AI, and it kind of shows. It's still not super coherent in terms of features. One day I'll revisit it. But basically you can aggregate news using RSS, read it from the terminal and even have AI summarize it.

[Kita-lugano](https://dovidio.github.io/kita-lugano/): I needed to see which kitas were available in Lugano and couldn't find a central place to filter them. So here it is. I hope this saves some time for busy parents.

[Snappacci](https://github.com/dovidio/snappacci): A store-and-forward image/blob library in Java, taking advantage of the new native memory API. Hasn't been used in production yet, but it was a good excuse to learn about formal verification and the feasibility of using TLA+ in the age of agents.

[TimeTrackete](https://dovidio.github.io/time-tracking/): Native iOS app to track time using a single file as an event ledger. It took a couple hours to build and it's what I use to track time at work. The coolest thing about it is that it ships Shortcut actions you can associate with a double tap, so you don't even need to open the app to start or stop the tracking.
