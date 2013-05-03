---
layout: post
title: "Prepare business operations"
date: 2013-03-05 13:41
comments: true
categories: startup
---

With my [attention](http://shawn.dahlen.me/blog/2013/02/25/shifting-my-attention/)
now focused on the startup, I spent this week bootstrapping the business. The
objective was to establish a project plan achievable within thirty weeks while
preparing technical and financial operations. Before I dive into the
particulars of the schedule, let me provide a look into what I will be working on.

<!-- more -->


The Product
-----------

Before I made the leap, I had several ideas worth pursuing. Most were mid to large
in size. I would have had to create a working prototype over the thirty week
time period and raise venture capital to take it to the next step. Instead, I
decided to focus on a niche and implement a full solution within the same timeframe
with the mid-term objective of generating semi-passive income that could seed
bigger ideas.

The niche I will be targeting is a task management app that provides first-class
support for the [Pomodoro Technique](http://www.pomodorotechnique.com/). Why you ask? First and foremost, it scratches my own itch. As Paul Graham penned in
[How To Get Startup Ideas](http://www.paulgraham.com/startupideas.html), the best
ideas are ones in which the founders themselves want and can build. While
there exists many first-class task management solutions, none provide rich support
for arguably one of the most [popular productivity methods](http://lifehacker.com/5890129/five-best-productivity-methods). I will initially launch with a web-based
interface and follow with iOS and Android versions pending traction of the
product. More information will follow regarding the product, its features, and
user experience in several weeks.

The Plan
--------

The plan is simple. I have divided the thirty weeks into five sprints each
with a specific work package (objective). The work package is further decomposed
into six tasks that are approximately one week of effort each. This approach
afforded me a clear weekly focus that I could subsequently share with you.

I opted to tackle the product architecture first versus the typical approach of
defining product requirements. I felt that more risk existed with the product
architecture since I had not operated systems on the Internet before while the
base requirements are already well-defined within the Pomodoro Technique
[book](http://www.pomodorotechnique.com/download/pdf/ThePomodoroTechnique_v1-3.pdf).

With the current plan, I intend to launch the product in early October (accounting
for some vacation time with my family).

**Sprints**

1. Define and prototype the product architecture
    - Week 1: Automate provisioning of secure servers
    - Week 2: Setup Node.js servers within a load-balanced configuration
    - Week 3: Setup a MongoDb database cluster
    - Week 4: Setup an automated functional testing framework
    - Week 5: Setup a performance testing framework
    - Week 6: Setup a monitoring and backup strategy
2. Define product user experience
    - Week 7: Draft personas, goals, and feature requirements
    - Week 8: Draft paper storyboards
    - Week 9: Establish a visual design
    - Week 10: Draft design compositions for time and inventory features
    - Week 11: Draft design compositions for reporting and gameplay features
    - Week 12: Draft design compositions for registration and payment features
3. Implement registration, time, and inventory features
4. Implement reporting, gameplay, and payment features
5. Conduct testing and draft marketing plan

Other Activities
----------------

Beyond defining the schedule and the associated research required to scope it, I
tackled a few other activities in preparation for week one kickoff:

- I incorporated my business, tenatively named *Whimsical Bits, Ltd.*, through
  [Rocket Lawyer](http://www.rocketlawyer.com/) and drafted my LLC Operating
  Agreement.
- I implemented a [script](https://github.com/smdahlen/mac-bootstrap) to bootstrap
  my mac for software development that includes setup of homebrew, vim, irssi,
  rbenv, and a few ruby gems (which I will discuss next week). Additionally, I
  refined my personal [dotfiles](https://github.com/smdahlen/dotfiles) that
  intend to push to all servers I work on.
