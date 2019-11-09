---
title: "Installing Private Gems During a Docker Build"
date: 2018-05-10T17:46:52+01:00
tags: [docker, ruby]
---

If you want to install [RubyGems](https://rubygems.org/) using [Bundler](http://bundler.io/)
during a [Docker](https://www.docker.com/) build, life becomes a little bit more complicated
when some of these Gems are hosted in a private Github repository.

The problem is that we do not want to expose any secret credentials in any of the
container images that are built, and ideally we want to keep the build environment
as simple as possible.

So what can we try?

## SSH agent forwarding

Plenty of people have described this method, and it is the thing that pops up
most when searching for solutions. It involves mounting your host ssh agent
against the container you're building:

 - https://dchua.com/2016/01/15/ssh-agent-forwarding-with-your-docker-container/
 - https://blog.cloud66.com/pulling-git-into-a-docker-image-without-leaving-ssh-keys-behind/

Unfortunately if you're using Docker for Mac, there is [an open issue](https://github.com/docker/for-mac/issues/410)
which means this solution turns into more of a hassle than you'd like.

## Use a Gem server

It's [pretty trivial to run your own Gem server](http://guides.rubygems.org/run-your-own-gem-server/).

One solution could be to fetch the Gems you need in your build environment,
push them to your own Gem server, and then configure your Docker build to fetch
from this server.

I wasn't a fan of this solution because it involves a lot of scaffolding within a build environment, and there were also complications in potentially making the repository available to other people
working on the code.

## Private Gem repository

Host your private Gems in a private repository such as [Gemfury](https://gemfury.com/)
that allows you to pass in a token as an environment variable at build time.

If you're already using a repository to host private Gems, I'd suggest this
solution. We weren't, and it felt like a lot of over engineering for what
I was trying to solve.

## Bundle package

I liked this solution, but it isn't the most efficient, and may also contribute to
a larger image size.

Run `bundle package --all` on the host to fetch the Gems. During the build,
`COPY` them to the container as use `bundle install --local`.

If you're building for a different platform, for example, building on MacOS
for a Linux container, also use the `--all-platforms` flag.

This increases operation time as you must fetch on the host and copy to the container
rather than fetching directly.

## Use a Github personal access token

This is my preferred solution. You can pass in an environment variable
without chaning _too_ much in the build environment, but there are some
considerations for the developers working in your repository.

Create a [personal access token](https://github.com/settings/tokens) your Github account (or the account you're
using during build time). This can be passed in as the `BUNDLE_GITHUB__COM` environment
variable and Bundler will [use this token to authenticate with Github](http://bundler.io/v1.16/bundle_config.html#CREDENTIALS-FOR-GEM-SOURCES). The format of the value should be:

`BUNDLE_GITHUB__COM=abcd0123generatedtoken:x-oauth-basic`

In the Dockerfile, set this environment variable using `ARG`. Then pass it in
using the [`--build-arg`](https://docs.docker.com/engine/reference/commandline/build/#set-build-time-variables---build-arg)
flag. This ensures that the value will not be present anywhere within the image,
which ensures we can fetch the Gems securely. For example:

`docker build -t myimage --build-arg BUNDLE_GITHUB__COM=mytoken:x-oauth-basic .`

It's worth noting you can just also just pass in an environment variable within that command:

```
export BUNDLE_GITHUB__COM==mytoken:x-oauth-basic
docker build -t myimage --build-arg BUNDLE_GITHUB__COM .
```

This will require ensuring that your `Gemfile` is fetching Gems using `https`
rather than an SSH key. For example, where you once had:

`'mygem', git: git@github.com:myrepo/mygem`

You would now have:

`'mygem', git: https://github.com/myrepo/mygem`

If you were changing this in your Gemfile, it would force any developers to set
a token as well. This can be added globally by running:

`bundle config GITHUB__COM abcd0123generatedtoken:x-oauth-basic`
