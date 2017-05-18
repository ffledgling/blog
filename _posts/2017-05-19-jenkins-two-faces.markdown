---
layout: post
status: publish
published: True
title: "Jenkins: The Two Faces"
author:
  display_name: ffledgling
  login: ffledgling
  email: ffledgling@gmail.com
date: '2017-05-18 18:35:40 -04:00'
comments: true
categories:
- Technical
tags:
- CI
- Jenkins
---

The Wikipedia entry for Jenkins describes it as "an open source automation server written in Java.", but it goes on to say it "helps to automate the non-human part of the whole software development process with now common things like continuous integration and by empowering teams to implement the technical aspects of continuous delivery".

This brings me to a question that has become increasingly pertinent to my work over the past few months: Is Jenkins a CI/CD system or is it a general purpose automation system?

"Why is such a distinction necessary?", you might ask. Or maybe you're confused about why it can't be both. Allow me to elaborate.

My claim: A CI/CD system and an automation system have fundamentally different User Experience requirements [1], which makes something that tries to do both a somewhat complex and unwieldy beast that has identity issues.

To explain why I think the User Experience requirements are different, I will first try to identify the stakeholders for both use cases.
There are three broad groups of stakeholders:

- The "Users", developers running their builds or people running/triggering automated tasks.
- The "Administrators", the folks who manage and optimize the working, configs, uptime, speed and quality of the application.
- The "Watchers", the people interested in getting an overview of things. The sheriffs, release managers or stray folks.

While the stakeholders remain the same for both classes of systems, their expectations from the system change based on the use case.

I will mostly talk about the conflict between the expectations and needs of "Users" and "Administrators" and set aside the "Watchers" because they're not nearly as demanding and their expectations do not change drastically from system to system.

Let's talk about CI. For "Users" a CI system is inherently best viewed as a black box. How it operates, schedules, configures your build machine is something they do not need to know or care about. A User's contract with the CI system is simple. He defines:

- The build environment
- The repository containing his code
- Build steps to run [2]
- Notifications needed

The "Administrators" also want the CI system to be _treated_ like a black box. Having a clean interface has many documented advantages[3]. This allows for flexibility to change the architecture and working of the CI system itself. Things like changing the scheduler, the machine provisioner, the underlying pipeline architecture, even the entire build system can be (and should be) swapped out when necessary. As an Administrator if all my users use to talk to me is a YAML or JSON file to describe their needs, I can theoretically switch from Jenkins to Travis to Concourse without letting the user know of anything as long as I don't break the contract defined by the YAML file.

Now the Automation System. For "Users", an Automation system is an easy way to offload tasks that they do frequently or sometimes it's simply an easy way to setup a thing and have others be able to trigger/run it with a click. This is inherently a good practice. Automate whatever you can. It eliminates human error, tends to make things reproducible and documents your processes to some extent.

The inherent demand from a User from a system like this though, is customizability. People automating things typically care about what machine something is being run from, how it's run and want to be able to tweak and customize it. Maybe they need to go through a jump host in a particular data center, maybe they're creating binaries or doing a controlled deployment and there are a few set of legal values they need to select from or enter into a form that are actually passed to the task.

This requirement for flexibility and customizability means that "Users" interact with the system a lot more, need additional control over what they do and how they do it.

This as you might've guessed is at odds with what "Administrators" wanted from their CI system. Clean abstractions, well defined contracts and so on. For an Automation system, the role of the "Administrators" changes and their role is ideally limited to guaranteeing the uptime and availability of the platform that allows users to Automate their tasks, Automation-Platform-as-a-Service if you will.

As you might know, a CI system and an APaaS setup are very useful pieces to have, necessary even. But what happens when something tries to slip on both pairs of shoes? The same thing that happens when you try to put on two different pairs of shoes together.

Let's Circle back to Jenkins now (see what I did there?).

The CI part of Jenkins is typically expected to Just Work™, integrate with your review system, code hosting and email but mostly stay out of our way. An invisible piece that no one really needs to see except a status page or similar.  
The Automation part of it however is something users (if you let them), will directly interact with. They will define their own jobs, they will need integrations, plugins, status summaries, visualizations, custom form fields and so on so forth. Being the explorers that developers are by nature, they will generally try to stretch the limits of what they can achieve or do with your system.

That's the user side of the story. The problem for a Jenkins Admin is now trying to walk the fine line that satisfies both use cases. As anyone who's ever been responsible for any sort of service or application will tell you, having people run free in something you've configured never ends well.

Just Works™, goes out the window. You, as an "Administrator", either let users configure and create jobs, or you don't. You either accept user requests to add new fangled plugins that extend job functionality or you don't. You either limit how or where your users can run their automation jobs or risk having your entire build farm overrun by a misconfigured user job.

There are a lot of trade-offs to be made, and the fine line gets exceedingly precarious to walk. At large enough scale, both in terms of users and jobs on your Jenkins, it becomes exceedingly untenable to keep everyone happy. You can either have something stable or something flexible and quickly evolving to adapt to new needs, but I don't really know anyone who manages to do both successfully. Moving fast (this is what your automation users want/are/do) often leads to breaking things (CI users hate this).

Now here's my ground breaking, earth-shattering, novel idea to reconcile this problem... are you ready for it? ... here it goes:

Don't. Do. It.

Yep, you read that right, don't do it.

The CI System and the Automation System are two separate use cases and users expect very different things from them. There is no earthly reason to use a single Jenkins instance or even a single tool to try and solve both use cases.

Are your "Users" really tech savy developers that eat C for breakfast and use Bash to wipe their mouth after? Hook them up to Concourse/Buildbot/the-thing-your-intern-wrote for your CI system and tell them to go use Ansible to automate their stuff.

Are your "Users" business people or developers that don't want to think too much about anything and just "want to get work done"? Stick 'em with Travis/Bamboo/Circle for CI and give them a Jenkins instance to do automation.

Are you an "Administrator" left scratching their head about this entire situation or don't get the point of all these new fangled tools and solutions? Are you wondering why people can't just use Jenkins for everything? If it ain't broke, don't fix it, right? Well, don't worry, Jenkins is a very versatile tool (overly so, even) and can adapt to most use cases you might have.

Do yourself a favour, setup two Jenkins instances, lock one down and throw it behind ci.mycompany.com and open another one up to the winds of fate and throw it behind automation.mycompany.com.

<hr />

[1] I use the term User Experience a little loosely here to mean

[2] I use build steps as a blanket term for compiling, building, testing and other things. The distinction between them is useful, but the lack of granularity does not take away from the overall point and is thus left out.

[3] Can one of you Software Engineering folks lend me a citation?
