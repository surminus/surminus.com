---
title: "Over Engineering a Text File"
date: 2018-03-10T11:36:16Z
---

In the ultimate exercise of procrastination, last week I decided I would like to have a to-do list that I could
edit via command line to help reduce context switching and I would like this list to be in Github so I could
sync between home and work devices. I also thought friends or colleages could raise PRs if they wanted me to do something (a
feature no-one has yet taken advantage of, fortunately).

To begin, I [created a repository in Github](https://github.com/surminus/to-do) with my [list in markdown](https://github.com/surminus/to-do/blob/master/to-do.md).

Then I decided rather than do any of things on the list, I would write some Bash script to help me manage this text file.

It started as some simple aliases, and then blossomed into having an archiving function (I want to have an archive of everything
I've "achieved") and auto updates against Github (I don't want to have to remember to sync between devices).

Traditionally I've used [Google Keep](https://www.google.com/keep/), but I often forget to check it and am more inclined to get into a
routine when using command line. Shame there [isn't an API for Keep](https://stackoverflow.com/questions/19196238/is-there-a-google-keep-api) as far as I can find.

I have been reminded that there are already a whole variety of tools out there to help you manage
to-do lists. However, this would have been significantly less fun, and I probably would have had to do the things on my list.

<script src="https://asciinema.org/a/mjKRWIGj0VB1yO40yJDpg2yWq.js" id="asciicast-mjKRWIGj0VB1yO40yJDpg2yWq" async></script>
