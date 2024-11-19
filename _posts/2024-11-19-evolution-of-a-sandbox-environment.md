---
layout: post
title:  "The evolution of a sandbox environment"
subtitle: A journey from Vagrant, through VMs and docker, to Kubernetes
date:   2024-11-19 10:00:00 +0100
published: true
exclude_from_index: true
---

At a previous company I worked on a Developer Experience platform to provide sandbox test environments to other engineers in the company.

## What is a sandbox?

In case you haven't come across the concept of sandboxes before, the idea is to provide an individual, emphemeral, isolated, deterministic test environments, ideally with a consistent set of test data that allows a number of use cases including:
* Development, maintenance and testing of existing microservices
* Running CI pipelines for pull requests for changed microservices
* Running CI pipelines for mobile apps prior to release to Appstore/Playstore.
* Demos of the platform for training, promotion videos etc.


At my time at the company I saw the sandbox environment evolve over more than a decade to meet the changing requirements of the engineers.  Although I was a user at almost all stages apart from the first, I was
only working on the platform from version 3 onwards, so some of the details could be at best a bit sketchy, and at worse incorrect.

## Version 1 - Vagrant laptops

At this time, the company probably had less than 10 microservices which ran in production on dedicated servers and around 50 engineers.  These were developed either in Perl or Ruby, with a MySQL database.

The team created sandbox setup scripts that engineers could run using [Vagrant](https://en.wikipedia.org/wiki/Vagrant_(software)) to manage create a sandbox on their laptops.

Due to the small-ish number of microservices, this generally worked.

## Version 2 - Virtualise everything

As the company's features grew, so did the number of microservices and engineers were asking for laptops with more CPU and RAM
just to be able to run their sandbox environment.  By now the number of microservices had probably reached around 30 and a better solution was required.

The team obtained budget to setup a [VSphere](https://en.wikipedia.org/wiki/VMware_vSphere) environment in the company's data center.  They then developed a system using Ruby and MySQL that
communicated with the VSphere API to create sandbox environments on demands.  Engineers could have up to 3 sandboxes with up to 4 CPUs and 32G of RAM.  For some specific use cases, engineers
could request elevated persmissions that would allow them to create sandboxes with 6 or 8 CPUs and more RAM.

This worked well as the microservces were either Perl or Ruby and generally similar versions of Ruby.

## Version 3 - Ship the containers

After a couple of years of running sandboxes using VSphere, the number of microservices had grown to over 100.  Additionally through acquisitions and organic growth the company had become more polyglot in terms of programming
languages and frameworks. 

This presented a number of challenges:

1. Previously, in Version 2, the sandbox provisioning scripts cloned the git repos of the microservices and then started the apps in the normal way, for example for a ruby app the steps might be:
```bash
    git clone http://github.com/acme/login-app
    cd login-app
    bundle install
    bundle exec rail s
```
    However, on the backend, there was not just Perl and Ruby, but also PHP, Java and Scala and Node.  Different scripts and setups would be needed for each platform, and mainentance of these
    scripts should really lie with the team that maintains the microservice and not the sandbox team.  Teams were asking for various hooks and customisations to support their specific setup.

2. Additionally on the frontend, there had previously mainly been a bit of JQuery (and [Coffeescript](https://en.wikipedia.org/wiki/CoffeeScript)) and some static CSS, which was typically built using Ruby's  
   asset pipeline.  Now there were independent complex frontend build pipelines using tools such as Webpack with SASS/SCSS.  Again, support was needed to build these.

3. There were issues with different microservices using different language versions.  For some languages there were tools available e.g. `RVM` for Ruby, `NVM` for Node etc, but each language required its
   version management tooling.

4. In many some cases Node or Ruby used packages or gems which needed native Linux packages installing.  This became even more problematic when different microservices used different packages that required
   different versions of the same Linux package version. For example one ruby gem woudl require [ImageMagick](https://en.wikipedia.org/wiki/ImageMagick) version 3 and another version 4.  It was just not possible
   to install both on the same vitual machine.

5. Some teams had decided they wanted to use some hot new feature of MySQL 6 whereas everyone else was using MySQL 5.  Running both concurrently was problematic.

Although this was all managed through scripts, there were very frequent problems with sandboxes not starting due to failed dependency installs - some team would update a Ruby gem version or Node package and
this would break all new sandboxes, with the team claiming "It works on my machine".  These problems also difficult to reproduce as the order of installing packages could affect the outcome.

Fortunately just at this time Docker started to gain traction - teams would just deliver us a Docker image with everything needed inside. We could just throw away 90% of our scripts and just create a huge Docker Compose file with an entry for each of the 100+ microservices:

* No more sending us pull requests to modify sandbox scripts to handle their latst JS build tool or language framework, or ask for extra hooks in the script.
* No more incompatible ImageMagick or Ruby versions.  The correct version of everything would be inside the image
* No more problems in running different versions of MySQL.  We could run 10 versions if we wanted (resource permitting) with each in its own container.


Although this seemed like a technical silver-bullet it proved much more difficult organisationally.  Microservice teams had little or no experience of Docker, and
creating a Docker image of your app, if you have never done it before is non-trivial.  As Docker was quite new, there were not many online resources to help, not
so many prebuild base images and certainly no ChatGPT to do it for you.  Additionally teams had their day-to-day work to do and didn't have time to dedicate a week
or so to learning docker and building an image.  There was also a chicken-and-egg scenario where we couldn't launch Docker based sandboxes without images for each
microservice and teams wouldn't get the benefit of building an image until all other teams had done so.

In the end, we the sandbox team, with the help of some other core teams, ended up building the majority of the images for the 100 or so microservices.  It probably took
over a month for 5 to 10 people.

But in the end we got there and containerisation lead us to the Promised Land with a much more stable environment.


## Version 4 - K8S

## Version 4.5 - AWS










At NW we have a micro-services architecture based on Kubernetes (K8S).  This consists of hundreds of different apps using a variety of technologies, but all using docker images.

In many organisations with micro-services, integration testing is challenging, and is often solved in a couple of different ways:
1. Having a small number of testing environments (often 1) which try to simulate production to some extent.
2. Engineers run their own testing environment on their laptops using docker or a local K8S setup.

Both of these approaches have problems.

Using shared testing environments often means that these enviornments are in a unstable and undefined state, often with inconsistent data.  For example team A deploys a test version of their app and team B also wants to test their app which has unrelated changes but uses the REST API of team A's app, and starts failing during to bugs introduced by team A.  To avoid this often teams need to book time on the test environment where is not being used by others.

This problem is solved by running a testing environment on your laptop, but does not scale once you have more than a dozen micro-services, and even with small numbers can be very heavy on resource utilization needing engineers to have high spec machines.  It also requires a lot of setup which although can be automated, is difficult to support if things do not work well, and can produce inconsistent results.  A recently newer problem with this approach is that if developers have Macs with M1 processors, these may have difficultly running x86 docker images.

We aimed to address both of these problems by providing a individual, ephemeral testing environment that can be scaled if needed to run all the microservices has its own set of consistent test data and runs remotely rather than on engineers' laptops.

This provides for a number of different use cases, including:

