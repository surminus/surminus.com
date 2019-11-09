---
title: "My Favourite Tools"
date: 2018-03-02T17:15:11Z
categories: [tools]
tags: [lint, CI, terraform, cli, tooling]
---

The market is flooded with great tooling, so I wanted to list a few of my
favourites that I've used in the past.

## Vale

[Vale](https://github.com/ValeLint/vale) is a linting tool for documentation. Stick it in
CI pipeline to check for markdown (or whatever flavour documentation you use), tweak
your requirements with [styles](https://github.com/ValeLint/vale/tree/master/styles),
and set it to fail on poor grammar.

## asciinema

[asciinema](https://asciinema.org/) live records your terminal session and then uploads
somewhere providing you with a link to share. Perfect for demo'ing something you've made
or sharing how to do something without writing endless step-by-steps.

## Hugo

This blog is created using [Hugo](https://gohugo.io/). It is simple to use
and fuss free. Updating themes is very straight forward.

## adr-tools

I like [adr-tools](https://github.com/npryce/adr-tools) more for encouraging
the concept of writing ADRs than the tool itself, but it provides a useful
function to get your team writing up decisions.

## terraform-docs

If you write any amount of Terraform it's useful to be able to generate some
documentation that describes what your module or manifest does, and it's variables
and outputs. [terraform-docs](https://github.com/segmentio/terraform-docs) ensures
this is straight forward to do, and [here](https://github.com/alphagov/govuk-aws/tree/master/terraform/modules/aws/node_group)
is an example of how we used it in [GOV.UK](https://www.gov.uk).

## tfenv

Sticking with Terraform, [tfenv](https://github.com/kamatama41/tfenv) makes it easy
to switch between versions.

## Streisand

[Streisand](https://github.com/StreisandEffect/streisand) creates a VPN that can be
built on multiple cloud providers, and when it's finished, outputs instructions to pass
on to your family and friends. A superbly developed tool.

## gof3r

I'm giving [gof3r](https://github.com/rlmcpherson/s3gof3r) an honourable mention as
I used it in the past as a way to stream database dumps straight into an S3 bucket, which
means you don't have to do any pesky disk write operations.
