---
title: An app for learning songs
date: 2022-09-27T17:09:14.800Z
image: /images/uploads/frame-1.png
---


R﻿ecently I have been playing quite a lot of guitar. One approach I use to learn songs is to just play along with the original song.
This approach has a couple of problems though:

1. When you are in the early stages of learning, you are rather learning the song piece by piece, or riff by riff
2. I﻿t's usually hard to play the song at the original speed right from the start

G﻿uitar Pro offers an excellent solution to this problem, you can download the guitar tab, select a song part, loop it, and even change the speed, or set up progressive speed. However I noticed a few inconveniences with it:

1. N﻿ot all songs have a guitar tab, and some of the guitar tabs are low quality
2. Y﻿ou can waste a lot of time on the internet trying to find the right tab, and you can get easily distracted and \forget about what you were searching in the first place

That's why I decided to build a small app I could use to learn songs.\
A﻿t the beginning, the idea was to have a native Mac App. The idea was that user could add a song, select a part of the song and even modify the speed.\
After building the first prototype, I've realized that it would be much cooler if user wouldn't have to manually add a file, but could just search the songs inside the app. Luckily, Apple provides \[https://developer.apple.com/musickit](Music Kit), a library that allows to programmatically interact with the Apple Music catalog. I've decided to use this library, and since it currently does't work correctly on Mac OS, I've pivoted to iOS and iPadOS.\
N﻿ow users can search any songs from the Apple Music Catalog, select a section, loop it, and even change the speed or define a speed progression.

