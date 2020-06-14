---
title: "Why You Should Write a CLI Tool for Your Organisation"
date: 2020-06-12T16:18:26+01:00
tags: [cli, tooling]
---

At my current place of work, I am (currently) the only Operations & Platform
focused engineer, in a technology team of around 30 people who are either
Software Engineers, Technical Leads and Technical Architects.

It is my job to ensure that engineers who are focused on Software Development
(all of the above), are able to achieve everything they need to, without
feeling too much drag from our internal tooling.

If Software Engineers are the drivers of Formula 1 cars, then I have to make
sure that the car itself isn't slowing them down.

I had dabbled with writing a command-line tool in a previous position, and when
I started here, one of the first things I did was write a command-line tool to
replace homespun scripts used by most of the Tech Team.

It also coincided with an interest in learning [Go](https://golang.org/), and
so I used [Cobra](https://github.com/spf13/cobra) to get me started.

## Replacing homespun scripts

It's not unusual to find developers using multiple homespun scripts to interact
with environments. For example, we had one script that listed the servers in
our environment; another script which started an SSH session to one of these
servers; and finally, a script which started a Rails or MySQL console on one of
these servers.

Each of these scripts had a slightly different name, and they existed in the
repo alongside our monolithic application.

I decided that if I could roll all of this functionality into a single tool, it
would be much easier for new starters to pick up a single tool. They would be
able to use `--help` if they got stuck, rather than having to root around in
our documentation on how to perform a particular function.

In the end, everytime I came across a homespun script, I incorporated it into
the command-line tool.

I worked hard to ensure it was easier to use my tool than any of the scripts,
in the hope everyone would choose to migrate across.

While I had very positive feedback, in the end we had to delete the old scripts
so we weren't supporting two sets of tools, and forced the remaining engineers
to use the new shiny tool instead.

This taught me that most engineers will not explore alternatives until it
directly impacts them. This is because most people are focused on the work, not
the tools, and this highlighted that having engineers focused on tools _as the
work itself_, is absolutely vital to reducing that drag.

## Powering up

Even before I started writing the tool, I had a vision that every interaction
we have with our development environment should have a command-line alter ego.

Everytime I interacted with an environment in the "traditional" way, I wondered
how I could convert this into the CLI.

I am an evangalist for automation, and am always mystified by engineers who are
quite willing to travel the long route round something when it could be so
easily simplified by code.

If I were able to introduce everyone to a simpler, easier way of doing things,
then people would willingly move to using the CLI.

Probably my greatest example of introducing this convenience, was to add the
`ci` command.

This performed a simple function:

* In the current working directory, discover the name of the Git repository
   and branch (I used the [go-git library](https://github.com/go-git/go-git)
   for this)
* Craft a link to the branch in CI
* Open a browser to the job (using the [open](https://github.com/skratchdot/open-golang) library)

I also included a couple of subcommands, `list` and `console` that output the
history of jobs and console output of the job respectively.

It didn't take me long to write, but I had overwhelmingly positive feedback
that it made engineers lives much easier by saving that mental workload of
having to find your branch in our CI tool.

The thing that struck me most was how surprised engineers were by something so
simple making their lives easier.

## To infinity, and beyond

With this momentum, everytime I got annoyed at having to do something, I added
it to the tool, and eventually it became the de-facto way to perform tasks.

Some example commands were:

* `alerts`: lists the current triggered monitors in
  [Datadog](https://www.datadoghq.com/), and the ability to mute them
* `app`: install dependencies, downloads and imports data snapshots, and an
  alternative to [Foreman](https://github.com/ddollar/foreman)
* `deploy`: trigger a deployment of the application, including deploying a
  branch in a test environment
* `docker`: small wrapper to help working with our images, particularly with
  [Amazon ECR](https://aws.amazon.com/ecr/)
* `scale`: scale AWS auto scaling groups in and out
* `secrets`: interacts with [AWS SSM Parameter
  Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)
  to add, rotate and delete secrets

## Feedback is key

Probably one of the tougher things I found was receiving feedback.  Often, if
something didn't quite work the way someone expected it to on the first go,
they would fall back to the "traditional" way, and I would never find out about
it to put in a fix.

I also made many of the decisions by myself in the implementation of the tool,
and appreciated any feedback anyone provided about how they interacted with the
tool, and what didn't make sense.

This is still an issue I struggle with, and I think the power dynamic for
internal tooling sometimes makes this difficult. Engineers are happy to use
tools that work; but they're often hyper aware of other engineers workloads, so
are less prone to raise bugs or UX feedback.

One of other things I struggle with is advertising the capability of the tool.
I often have to sneak into Slack threads and say "Hey! Did you know our tool
can do this?"

## What is the value to an organisation as a whole?

I am a strong believer that coming together to write a CLI tool has many
benefits beyond the convenience of automating away labourious tasks.

Firstly, and probably I think is the biggest benefit, is that working on
internal tooling in this way gives you a massive insight to the everyday
processes and Continuous Tasks that every engineer engages in.

This is key to understanding exactly what small things are taking precious
minutes away from each engineer everyday, and it also may bubble up some larger
organisational processes that should be changed in the process.

For example, we may have a deloyment process that involves going into a web
browser and clicking "deploy". We could move this into the CLI by writing
something that interacts with the CI tool API, but perhaps implementing
Continuous Deployment would be a more impactful change.

The next benefit is that it can be a project with minimal impact and immediate
feedback. I think this is why I love writing CLI tools. Engineers have a place
where they can write code for fun, without risk, and with the potential of
being able to unlock productivity for their colleagues.

It's also a place where everyone can come together and work on the same thing;
and perhaps it'll help you understand how your colleagues work in one team
compared to your own.

Last but not least, it is great for new starters to hit the ground running. One
of the most intimidating things when starting at a new place is getting to
grips with all the internal tooling, and being given access to 20 odd SaaS
accounts (a modern horror story).

Being able to install a tool, which is powerful enough to get you on your way,
is a wonderful way to start the journey.

It is my hope that I get to experience this at some point in the future, so
please everyone, start to write your own command-line tool!

Please feel free to [contact me on Twitter with your
experiences](https://twitter.com/surminus).
