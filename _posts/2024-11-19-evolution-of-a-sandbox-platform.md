---
layout: post
title:  "The evolution of a sandbox platform"
subtitle: A journey from Vagrant, through VMs and Docker to Kubernetes
date:   2024-11-19 10:00:00 +0100
published: true
exclude_from_index: true
---

At a previous company, I worked on a Developer Experience platform to provide sandbox test environments to other engineers in the company. 

During my time there I saw the sandbox platform evolve over more than a decade to support the development of hundreds of microservices, the growing number of the engineers and their 
changing requirements.

## What is a sandbox?

In case you haven't come across the concept of sandboxes before, the idea is to provide an individual, ephemeral, isolated, deterministic test environments, ideally with a consistent set of test data that allows a number of use cases including:
* Development, maintenance and testing of software
* Running CI pipelines for pull requests
* Running CI pipelines for mobile apps prior to release to Appstore/Playstore.
* Demos for training, promotion videos etc.


## Disclaimer 

Before I go further, and describe all the versions we went through, I should make it clear that although I was a user of the sandbox platform from Version 2, I was only working as part of the sandbox team from the start of version 3 onwards. 

This, combined with the fact of it being around a decade ago since Version 2, means that some of these earlier details could be at best a bit sketchy and at worse incorrect.

## Version 1 - Vagrant laptops

At this time, the company probably had around 50 engineers developing a Perl monolith and a small number of Ruby microservices.

In addition to the production environment there were two shared test environments, which were smaller versions of the production environment in terms of infrastructure.

### The curse of shared test environments

These shared testing environments were often unstable and in an undefined state as multiple teams could be testing at the same time.

For example team A would deploy a test version of their microservice while team B were testing their microservice which had unrelated changes but used the REST API of team A's microservice. Team B's microservice would then started failing due to bugs introduced by team A.

### The birth of sandboxes

To solve this problem the first version of sandbox environments were conceived.  These allowed each engineer to have their own test environment on their laptop.  The sandbox team created sandbox setup scripts that engineers could run using [Vagrant](https://en.wikipedia.org/wiki/Vagrant_(software)) to manage and create a sandbox on their laptops.

Due to the small number of microservices (less than 10), this worked well and and was a great step forward from relying on shared environments.

## Version 2 - Virtualise everything

As the company's features grew, so did the number of microservices. Engineers were asking for laptops with more CPU and RAM
just to be able to run their sandbox environment.  By now the number of microservices had probably reached around 30 and a more scalable solution was required.

### No more sandboxes on laptops

The sandbox team obtained budget to setup a [VSphere](https://en.wikipedia.org/wiki/VMware_vSphere) environment in the company's data center.  They then developed a sandbox management system using Ruby and MySQL that communicated with the VSphere API (in SOAP 😱) to create sandbox environments on demand.  Each sandbox would be represented by a Virtual Machine (VM) running within VSphere. 

Engineers could use the UI of this sandbox management system to create up to 3 sandboxes, each with up to 4 CPUs and 32G of RAM.  For some specific use cases, engineers could request elevated permissions that would allow them to create sandboxes with 6 or 8 CPUs and more RAM.

This was a great advance - RAM and CPU constraints of laptops were not longer and issue and resources could be
optimised - when engineers no longer needed a sandbox they could stop the VM, thus freeing up the RAM and CPU.  Restarting the VM would take a few minutes, but they could start where they left off as the
DB contents were persisted in a VM volume.

If the environment became unstable or corrupted they could just click a button and recreate it via the sandbox management UI.

## Version 3 - Ship the containers

After a couple of years of running sandboxes using VSphere, the number of microservices had grown to over 100.  Additionally through acquisitions and organic growth the company had become more diverse in terms of programming languages and frameworks. 

This presented a number of challenges.

### The problems of the pre-container world

1. Previously, in Version 2, the sandbox provisioning scripts cloned the git repos of the microservices and then started the microservices in the normal way, for example for a Ruby on Rails microservice the steps might be:
```bash
    git clone http://github.com/acme/login-app
    cd login-app
    # Install the dependencies
    bundle install
    # Start the web server
    rails server
```
    However, on the backend, there was not just Perl and Ruby, but also PHP, Java, Scala and Node.  Different scripts and setups would be needed for each platform, and it was difficult for the sandbox team
    to have all knowledge of all these development environments to maintain these scripts.  It would be better if the team that owned the microservice could in some way own the setup scripts also.
    
    Teams were also asking for various hooks and customisations to support their specific setup.
2. Additionally there were challenges with the frontend. Previously mainly been a bit of JQuery (and [Coffeescript](https://en.wikipedia.org/wiki/CoffeeScript)) and some static CSS, which was typically built 
   using Ruby's asset pipeline.  
   
   Now there were independent complex frontend build pipelines using tools such as Webpack with SASS/SCSS.  Again, support was needed to build these.

3. There were issues with different microservices using different language versions, e.g. Ruby 2.6 vs Ruby 2.8.  For some languages there were tools available e.g. `RVM` for Ruby, `NVM` for Node etc, but 
   each language required its own version management tooling.

4. In some cases Node or Ruby used libraries which needed native Linux packages installing.  

   This became even more problematic when different microservices used different libraries that required
   different versions of the same Linux package version. For example one Ruby gem would require [ImageMagick](https://en.wikipedia.org/wiki/ImageMagick) version A and another version B.  It was just not possible
   to install both on the same VM.

5. Some teams had decided they wanted to use some hot new feature of MySQL version Y whereas everyone else was using MySQL X.  Running both concurrently was problematic.

Although this was all managed through scripts, there were very frequent problems with sandboxes not starting due to failed dependency installs - some team would update a Ruby gem version or Node package and
this would break all new sandboxes, with the team claiming *"It works on my machine"*.  

These problems were difficult to reproduce as the order of installing packages could affect the outcome.

### Dockerize everything

Fortunately just at this time Docker started to gain traction - our idea was that teams would just deliver us a Docker image with everything needed inside. We could just throw away 90% of our scripts and just create a huge Docker Compose file with an entry for each of the 100+ microservices:

* No more sending us pull requests to modify sandbox scripts to handle their latest JS build tool or language framework or asking us for extra hooks in the script.
* No more incompatible ImageMagick or Ruby versions.  The required version of everything would be inside the image.
* No more problems in running different versions of MySQL.  We could run 10 versions if we wanted (resource permitting) with each in its own container.

### A simple silver bullet?

Although this seemed like a technical silver-bullet it proved much more difficult organisationally.  

Microservice teams had little or no experience of Docker, and creating a Docker image of your microservice, if you have never done it before is non-trivial.  As Docker was quite new, there were not many online
resources to help, not so many prebuild base images and certainly no ChatGPT to do it for you.  

Additionally teams had their day-to-day work to do developing new features and didn't have time to dedicate a week
or so to learning Docker and building an image.  

There was also a chicken-and-egg scenario - we couldn't launch Docker based sandboxes without images for each
microservice but teams didn't see the benefit of building an image unless they could get a Docker based sandbox to run it on.

In the end we, the sandbox team, with the help of some other core teams, ended up building the majority of images for the 100 or so microservices.  It probably took
5 to 8 people working intensively for around a month.

But we got there in the end. Containerisation led us to the sun-lit uplands (for a while at least) with a much stabler environment and the ability to have any mixture of language frameworks and versions so
long as they could be dockerized.

## Version 4 - Bring on Kubernetes

Yet again, this worked well for us for a couple of years, but now the number of microservices was exploding to upwards of 200.  Furthermore there were additional services requested by teams such as PostgreSQL, 
Cassandra,Elastic Search and Kafka.

Therefore bigger and bigger VMs were required to run an individual sandbox.  Engineers could compromise and only start the microservices they needed on the sandbox to reduce resource demands but the dependency
graph of microservices was not always well understood so knowing what microservices you needed was not easy.  When a REST endpoint on service failed was it due to a dependent service not being started? Or was
it some other issue introduced by code changes.  

Debugging this was time-consuming.

### Why not just bigger VMs?

In addition to the cost of buying more hardware, adding more CPUs to VMs has diminishing returns in terms of performance. Although a set of VMs are split across physical hosts, a single VM only runs on a single host.  When the number of virtual CPUs of a VM is increased even if the physical machine is more powerful, contention and scheduling issues will not mean a proportional scale-up of performance.

To solve this problem, we needed to split the sandboxes across more than one physical machine and move away from VMs.  Hence [Kubernetes](https://en.wikipedia.org/wiki/Kubernetes) (K8S).

### K8S to the rescue

K8S was beginning to be used more widely in the IT industry so we worked on a proof of concept.  We deployed a mini-K8S cluster on our laptops and converted big sandbox Docker compose file into K8S definitions
with K8S Deployments for each microservice.  

Because everything was already Dockerized, part of the hard work had already been done. With the help of some great colleagues from another central team, we managed 
to get a prototype working in a couple of days - we could display the welcome page of the web site as it appeared in production, login with a test user and see their home page.

This proof-of-concept was just a long K8S YAML file with hardcoding. To have a developer friendly sandbox management system we would need to interface with the K8S API to generate and monitor
all the K8S resources that we needed.

### Complete rewrite

The current Ruby and MySQL management system was very tightly coupled to VSphere and imperative state.  

Imperative in this case means that a user would press a button in the sandbox management system to 
create a sandbox and this would then synchronously call the VSphere API to create a VM and then return when it was done.  The K8S approach is not imperative, but declarative - you tell K8S
what you want the final state to be via the API - e.g. "create a running container with this image somewhere in the K8S cluster" -  the API response is immediate and it the background K8S will try forever to achieve your request.

Furthermore the declarative approach of the existing sandbox management system had led to a lot of problems with not having a definitive source-of-truth for the state of the sandbox - if the VM was started but the MySQL DB said it was stopped, what should the state of the sandbox be and what should we do?

Therefore due to this VSphere coupling and problems in managing state we realised that just adapting the existing sandbox management system was not an option, so we decided to rewrite it using Elixir with
(initially) no DB.  

This took around 1 year to develop, but gave us a platform that could communicate with the K8S API asynchronously and provide many more features than the original
sandbox management system provided such as providing a REST API for CI use cases, viewing container logs through the UI, changing the docker image of already running microservices
and even running remote development environments within a sandbox.

With all this, 300+ engineers were able to run high performance sandboxes with over 400 microservices that could be up and running within around 2 minutes.  If more microservices or users were required in the
future the hardware could be easily scaled horizontally.