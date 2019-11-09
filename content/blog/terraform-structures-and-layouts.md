---
title: "Terraform Structures and Layouts"
date: 2018-04-02T13:27:53+01:00
tags: [terraform, cli, tooling]
---

Having recently moved role, one of my first tasks is to try and combine a mixture of
CloudFormation JSON and custom Ruby scripts using Fog into a single repository using
[Terraform](https://www.terraform.io/).

The biggest challenge to me was what structure the code should take.

Hashicorp provide ["best practices"](https://github.com/hashicorp/best-practices/tree/master/terraform)
but these do not provide any conclusion on the structure.

There is also [Terragrunt](https://github.com/gruntwork-io/terragrunt), which
provides tooling around an opinionated structure (Everything-is-a-Module).

What I detail below is a breakdown of an opinionated structure we devised when [I worked
on GOV.UK](https://github.com/alphagov/govuk-aws/blob/master/doc/architecture/decisions/0010-terraform-directory-structure.md),
and which I found myself drawn to after starting a fresh project.

## One statefile to rule them all

The first question to ask yourself is: should I use a single statefile, or split
my infrastructure into multiple states?

A single statefile can help simplify things greatly; all your variables and outputs
will be defined in one place, and you can directly reference all your resources:

`vpc_id = "${aws_vpc.my_vpc.id}"`

In this structure, our project looks like this:

```
project
├── dns.tf
├── ec2.tf
├── main.tf
├── outputs.tf
├── rds.tf
├── sg.tf
├── variables.tf
└── vpc.tf
```

With some simplification of code comes some cost: what happens if there is a problem
with a resource in your statefile? Terraform will quit, and anything after will not
be applied. If there is a problem at the beginning of the chain of resources, for example
with some VPC settings, then it will mean you are stuck deploying until you have
resolved this issue.

It's worth noting you can use the `-target` flag when deploying, but this is a bit like
using `--skip-broken` on a package manager. When did that ever feel right?

## Multiple states, many deploys

Enter multiple statefiles and data sources. By splitting your code into multiple
statefiles, you are reducing the impact that a broken statefile will have on your
infrastructure.

Let's break up our code for the VPC, DNS, security groups, EC2 and RDS into separate
statefiles. If our DNS code breaks, then it won't stop deploys of RDS, which seems
sensible.

Referencing other resources does add a little complexity. You need to add
a [Terraform remote state](https://www.terraform.io/docs/providers/terraform/d/remote_state.html)
data source, and reference the resource using that data (ensuring you have set the [output](https://www.terraform.io/docs/configuration/outputs.html) in the upstream code):

`vpc_id = "${data.terraform_remote_state.my_vpc_project.my_vpc_id}"`

This now means we have created a dependency tree. We must ensure the VPC code
has been deployed before we can run any code that depends on it.

It also means that to deploy our infrastructure, we need to run Terraform five times
in the right order.

We've made our code less vulnerable to problems, but added complexity for deployment.

This is even before we start to consider the versioning of the code that is deployed. We made a change
to a core piece of the infrastructure that everything is dependent on, but have we
deployed everything so it's up to date?

[Terraboard](https://github.com/camptocamp/terraboard) could be a nice solution to this,
though I have yet to deploy it in a real environment.

Our project now looks like this:

```
project
├── dns
│   ├── main.tf
│   ├── outputs.tf
│   └── variables.tf
├── ec2
│   ├── main.tf
│   ├── outputs.tf
│   └── variables.tf
├── rds
│   ├── main.tf
│   ├── outputs.tf
│   └── variables.tf
├── sg
│   ├── main.tf
│   ├── outputs.tf
│   └── variables.tf
└── vpc
    ├── main.tf
    ├── outputs.tf
    └── variables.tf
```

## Multiple environments

If you're working on a project of any size, you will probably be working with multiple
environments (Production, Staging, Dev) which you want to keep roughly in sync.

The pattern for achieving this is using data structures. In Terraform you can [specify
variables using "var files"](https://www.terraform.io/intro/getting-started/variables.html#from-a-file).
These are a good way to define a set of data in a readable format.

We can create different variable data for different environments, and these can be split
into specifically named files. We can also split our backend configuration to be
environment specific, so we can reference different backends or keys.

**Always use a [backend](https://www.terraform.io/docs/backends/index.html) to store your state! In current versions of Terraform they are easy to configure, so there is no excuse to keep your statefile in Github.**

We can reference the environment in an environment variable for deployment:

```
export ENVIRONMENT=staging
terraform init -backend-config=$ENVIRONMENT.backend
terraform apply -var-file=$ENVIRONMENT.tfvars
```

The amount of code is expanding:

```
project
├── dns
│   ├── main.tf
│   ├── outputs.tf
│   ├── production.backend
│   ├── production.tfvars
│   ├── staging.backend
│   ├── staging.tfvars
│   └── variables.tf
├── ec2
│   ├── main.tf
│   ├── outputs.tf
│   ├── production.backend
│   ├── production.tfvars
│   ├── staging.backend
│   ├── staging.tfvars
│   └── variables.tf
...
```

## Hierarchy of data

There is a strong chance you have variables which are common across all environments.

When referencing variable files in Terraform, the variables contained in the last
referenced file will override any previously specified variables.

For example, in `staging.tfvars` we have:

```
colour = "red"
```

In `common.tfvars`:

```
colour = "yellow"
```

When we deploy the code, we can reference the var files in a hierarchy:

`terraform apply -var-file=common.tfvars -var-file=staging.tfvars`

This ensures that the `staging` specific variable has the correct data for that environment. `"${var.colour}"` will return `red`.

Having a common data file ensures that we can remove repeated data in the environment specific
data file:

```
project
├── dns
│   ├── common.tfvars
│   ├── main.tf
│   ├── outputs.tf
│   ├── production.backend
│   ├── production.tfvars
│   ├── staging.backend
│   └── variables.tf
├── ec2
│   ├── common.tfvars
│   ├── main.tf
│   ├── outputs.tf
│   ├── production.backend
│   ├── staging.backend
│   └── variables.tf
```

If we have enough environments, it may make sense to move the variables to a
dedicated data directory. We could also implement common data that is
the same everywhere, or environment specific data that is common across
all manifests.

To split out the data, we may structure it in this way:

```
project
├── data
│   ├── common.tfvars
│   ├── dev.tfvars
│   ├── dns
│   │   ├── common.tfvars
│   │   ├── dev.tfvars
│   │   ├── production.tfvars
│   │   └── staging.tfvars
│   ├── ec2
│   │   ├── common.tfvars
│   │   ├── dev.tfvars
│   │   ├── production.tfvars
│   │   └── staging.tfvars
│   ├── production.tfvars
│   └── staging.tfvars
└── manifests
    ├── dns
    │   ├── main.tf
    │   ├── outputs.tf
    │   ├── production.backend
    │   ├── staging.backend
    │   └── variables.tf
    ├── ec2
    │   ├── main.tf
    │   ├── outputs.tf
    │   ├── production.backend
    │   ├── staging.backend
    │   └── variables.tf
...
```

Our deployment may now look something like this:

```
cd manifests/ec2
terraform init -backend-config=staging.backend
terraform apply \
  -var-file=../../data/common.tfvars \
  -var-file=../../data/staging.tfvars \
  -var-file=../../data/ec2/common.tfvars \
  -var-file=../../data/ec2/staging.tfvars
```

Suffice to say, it is at this point when wrappers are created to simplify deployment
logic.

I'd love to see a hierarchy of data added as a native function within Terraform. Perhaps
a data source that uses [Jerakia](http://jerakia.io/), or an extension on how variables
are implemented.

## Variables and outputs

There seems to be a pattern in which variables and outputs are put into
separate files. When using the structure defined above, I strongly advocate
that this is not necessary. Instead, put everything in one place, and if your
code is getting too unwieldy, package it up in a module.

If a variable is specific to the code inside the manifest file, it makes sense
to keep it in the same file. A similar rule should apply to outputs.

I like reading manifests that look like this:

```
/*
*
* My Manifest
*
*/

### Variables

variable "foo" {
  default = "bar"
}

### Resources

resource "some_resource" "foo" {
  name = "${var.foo}"
  ...
}

### Outputs

output "my_output" {
  value = "${some_resource.foo.name}"
}
```

If there are variables which are used across multiple files, then it makes sense
to have `variables.tf` file, but I would argue it's often not necessary.

```
project
├── data
│   ├── common.tfvars
│   ├── dev.tfvars
│   ├── dns
│   │   ├── common.tfvars
│   │   ├── dev.tfvars
│   │   ├── production.tfvars
│   │   └── staging.tfvars
│   ├── ec2
│   │   ├── common.tfvars
│   │   ├── dev.tfvars
│   │   ├── production.tfvars
│   │   └── staging.tfvars
│   ├── production.tfvars
│   └── staging.tfvars
└── manifests
    ├── dns
    │   ├── main.tf
    │   ├── production.backend
    │   └── staging.backend
    ├── ec2
    │   ├── main.tf
    │   ├── production.backend
    │   └── staging.backend
    ├── rds
    │   ├── main.tf
    │   ├── production.backend
    │   └── staging.backend
...
```

## Summary

This structure helps ensure you do not repeat code and definitions of variables,
and ensures you can work with multiple environments against a backdrop of multiple
statefiles.

It introduces some problems, such as what version of the code is deployed and
dependency management.

I would love to see a structure and layout that works with native Terraform. Terraform
is used most often in automated pipelines, and I would like to see a style guide
formed by the community that could aid in consistent pipelines across organisations.
It's always worth noting that Terraform hasn't hit v1.0 yet, so plenty of time
for these ideas to be formed.

## Vs Terragrunt

As mentioned at the beginning, [Terragrunt](https://github.com/gruntwork-io/terragrunt) is
a widely used wrapper that is dependent upon the defined Gruntworks structure,
and also requires adding Terragrunt specific configuration.

I looked at using this when I started my latest project - it handles things like
dependencies, albeit you have to add configuration to specify the dependencies. I
decided not to use it because if you're going down that path you have to really
commit to Everything-as-a-Module philosophy.

I would suggest modules should abstract away an atomic set of resources, but feel uneasy
at structuring absolutely everything within the module structure. I prefer
the flexibility on calling on a module when required, but plainly defining resources
in your main manifest file.
