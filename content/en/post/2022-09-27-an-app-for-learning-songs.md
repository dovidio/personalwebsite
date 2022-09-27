---
title: "An App for learning songs"
date: 2022-09-27T10:42:16+01:00
draft: false
image: /images/uploads/about_xs.png
tags: ["appstore", "ios", "mobile", "guitar"]
---
Imagine you want to learn a song by ear, wouldn't it be great if there was an app that gave you access to the Apple Music Catalog while giving you the option
focus on the part you are learning, while giving you the option to change the speed?

<!--more-->

Recently I have been playing quite a lot of guitar. One approach I use to learn songs is to just play along with the original song.
This approach has a couple of problems though:

1. When you are in the early stages of learning, you are rather learning the song piece by piece, or riff by riff
2. It's usually hard to play the song at the original speed right from the start

Guitar Pro offers an excellent solution to this problem, you can download the guitar tab, select a song part, loop it, and even change the speed, or set up a speed progression. However I noticed a few inconveniences with it:

1. Not all songs have a guitar tab, and some of the guitar tabs are low quality
2. You can waste a lot of time on the internet trying to find the right tab, and you can get easily distracted and forget about what you were searching in the first place

That's why I decided to build a small app I could use to learn songs.
At the beginning, the idea was to have a native Mac App. On a basic level, user could add a song, select a part of the song and even modify the speed.

After building the first prototype, I've realized that it would be much cooler if user wouldn't have to manually add a file, but could just search the songs inside the app. Luckily, Apple provides [Music Kit](https://developer.apple.com/musickit), a library that allows to programmatically interact with the Apple Music catalog. I've decided to use this library, and since it currently [does't work correctly on Mac](https://developer.apple.com/forums/thread/698902), I've pivoted to iOS and iPadOS.

Here's [Loop Trainer](https://apps.apple.com/us/app/looptrainer/id1634854221) on the Apple App Store.
Users can search any songs from the Apple Music Catalog, select a section, loop it, and even change the speed or define a speed progression.
I am still planning to release the initial Mac prototype since it might still be useful for someone, but this will come at a later time.


