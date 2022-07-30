---
title: 'Automata: A General-Purpose Automation Platform'
date: 2022-07-27 13:33:37
tags:
---

![](/images/Automata-thumbnail.png)

In this post, I summarise how I ended up building `Automata`, a platform to easily create and run arbitrary and powerful workflows that during their executions, can also store data as well as invoke alerts. 

If you want to skip the content and first see what Automata looks like, [click here.](#vBeta-Demo)

-----
## Introduction

Until mid-2020, I used to be a moderately active bug bounty hacker but then I decided to start working on automating my workflow of doing attack surface reconnaissance.

<blockquote>
FYI, Reconnaissance(recon) is gathering publicly available technical information about Target Company's Apps/Assets and throwing lots of TCP and HTTP packets at IPs, and domains hosting these to find interesting ones among thousands. Jason Haddix's Training  <a href="https://www.youtube.com/watch?v=uKWu6yhnhbQ">"The Bug Hunter's Methodology at DEFCON 28"</a>  goes in-depth, if you want to learn more about it.
</blockquote>

Later, what actually happened can be accurately summarised with an XKCD!

![](/images/automation-xkcd.png)
<p style="text-align: center; font-size:0.75rem; font-style: italic; margin-top: -10px;"> <a href="https://xkcd.com/1319/">(xkcd #1319)</a> </p>

Even till now, I haven't gotten back to actively bounty hacking. Obviously, work and personal life also took priority but this side project also got a big chunk from it. So, as you would ask, what happened?

## Lessons From the Past

#### 📄 Local Scripts 
Like every web hacker, I started with writing hacky scripts, which were just chaining some of the open source tools to get the job done. The scripts were running the usual tools for doing Passive Enumerations, Network Scanning, Application Layer Fingerprinting, etc.

This was helpful for starting manual hacking as I could now pick interesting assets to hack based on what recon automation found. But there were problems with it:

1. Long-running slow subtasks like Port Scanning, and Screenshotting were not finishing on targets with the huge number of assets.
2. Scheduling tasks with cron for Continous Automation, and Monitoring, was not possible on the local machine. Hence, the data would become stale quickly, and I was not able to trust this data even on the next day. 
3. Searching, aggregating, analytics, etc. on this unstructured & unindexed data dump was not possible.

These problems were not enough for me to solve them, I was pretty satisfied with the results from my core app hacking and was just using recon for kickstart. 

**Until I lost $8,500 in reward for a critical vulnerability getting duplicated due to a weird trail of events.**

Around 2019-2020, reporting critical CVEs was in fashion. There was a particular critical CVE that was trending at the time. The CVE was of Path Traversal + Partial Write Primitive = RCE for an asset that I had seen in the past on one of the very high-paying targets. I had not done a full recon for this target, just simple subdomain scraping and I did not have the fingerprint data, so decided to launch the scripts and browse through fingerprints on the search engines in parallel (Shodan, Censys, etc).

1. Friday, 9:00 PM - Start of Manual Recon and Automation. 
2. Friday, 9:30 PM - I am on `page Y` of Censys search results, scripts are still running.
3. Friday, 9:31 PM - A friend interrupted me for dinner. I close the laptop and don't come back to it till the next day.
4. Saturday, ~3:45 PM - I return back to look at the Recon work I was doing.
5. Saturday, 3:50 PM - I find the asset I was looking for on `page Y+1` of censys search as well as in the script's data dump and I check if it's vulnerable to the Critical CVE, it is Vulnerable! I get excited and report it.
6. Saturday, 4:13 PM - I get to know that I am the first to report the duplicate of a report submitted on `Saturday, 8:23 AM`.

![](/images/Automata-blog-Reports.drawio.png)

<p style="text-align: center; font-size:0.75rem; font-style: italic; margin-top: -10px;"> ($8.5k lost on a single report.) </p>

At that moment I thought, I lost the bounty just because I got interrupted. But soon after, I realized the `real root cause is not having a system with capabilities of continuous automation, monitoring and alerting`. 

Similarly <span style="font-weight: bold;">$50k+</span> worth of easy bounties I lost to all CVE duplicates in that period alone. Many hackers like me were just not able to capitalize on these easy wins at the time.

So, I decided to build a tool to solve this systemically.

#### 📸 Montage - The Handcrafted ASM Tool

In April 2020, I started developing a hardwired workflow service that would do what my scripts used to do and store all the results in the Database. This also used the stack's native task queue powered with Redis to distribute the workloads to multiple processes. Also wrote a UI-API for performing CRUD on this Data and invoking workflows.

I called it `Montage`.

At a high level it looked like this:

![](/images/Automata-blog-montage.drawio.png)
<p style="text-align: center; font-size:0.75rem; font-style: italic; margin-top: -10px;">(Montage HLD)</p>

At the time I did not know better and Montage was anyway a great improvement compared to my local clunky scripts.

1. Long executing subtasks were not a problem anymore as they were being run on a VPS with good resources.
2. Now I could run the automation continuously as well as in schedules, this solved problem of stale data.
3. The UI, API, and DB solved the problem with Data Usability (Now I could search for vulnerability signatures across all the Bug Bounty targets I was interested in, you can imagine this as my personal Shodan).

As you can see, on paper, it solved all of my previous problems right? 

But Montage came with its own problems (at least in the way how I designed and wrote it at the time), and they were always in the back of my mind from the start but I ignored those and thought of tackling them whenever they surfaced.

But they surfaced very soon.

1. I chose dynamically typed language for developing everything and wrote zero tests.
2. The code was not well structured from the start as I just wanted to get this done and move on to actual hacking. Updating and maintaining Workflow Service orchestration code became very hard.
3. It was not possible to deploy and distribute workflows on multiple machines, I was relying on just vertical scaling.  
4. I also did not pay much attention to UI code and design. It was coming up very poorly and was discouraging.
5. Anytime adding a new workflow or updating existing ones was breaking things and taking a lot of debugging time. Poor DevX. 
6. Nobody else could contribute their workflows themselves as they would have to learn the tech stack as well as the code. 
7. Workflow Service was a critical point of failure. Task failure handling was flaky.
8. Basically all the software engineering problems that come with a poorly designed system.
9. ... I can just go on. It had many problems.

These are all real problems but Montage was a good MVP and usable for Recon. I would have just gone ahead and used this anyway. At least at the time, I don't know how I would have designed it better on my own with just the knowledge I had.

I also thought of just utilizing what other people were developing but these tools suffered with their own problems, and customizing them to my needs was not possible. I wanted something according to my designs.

Luckily, when I was about 70% done with developing Montage, parallelly I was working on other projects which involved reviewing a CI/CD system for security issues and unrelated Automation.

1. Decomposing a CI/CD system for hacking, gave me a deeper understanding of how its different components worked and more importantly its Capabilities.
2. Automation project required running some OSS CLIs based on certain triggers. As I knew, designing and developing my own job runner is a pain (If you think, Montage at the core is also a job runner). 
3. The Infrastructure I was working with did not have access to any serverless self-scaling workload runner system.
4. But, the CI/CD system was self-scaling and container-native.
5. I used a single-step CI pipeline to execute the CLI so that now I don't need to worry about the problems with job runners, it's all now off-loaded to CI/CD system.

![](/images/Automata-blog-CI_CD.drawio.png)
<p style="text-align: center; font-size:0.75rem; font-style: italic; margin-top: -10px;">(... kind of using CI as a serverless compute platform.)</p>

This was a sustainable and minimal design that was easy to maintain. So now I knew better means to automate and I decided to sunset Montage.

## 🧩 Automata - A proper way to automate anything!

I started with listing what was ideally required:

1. Workflows:
    - It must be super easy to add, schedule, continuously run, test, update, and remove any number of workflows.
    - Workflows must be modular and we should be able to **define Code, Environment, Input, Output, and Resources (CPU/Memory)** required for these modules.
    - Workflows must be able to take advantage of multiple machines.
    - Previous executions of workflows must be auditable.
2. Data Persistence:
    - The data and logs produced during execution must be stored.
    - Should be easy to define storage schema.
    - We should be able to Search, Visualize, and Analyze data.
    - It should also provide artifact storage.
3. Alerting:
    - Workflows should be able to raise alerts while executing.
    - We shouldn't need to write code if we wanted to integrate a new alerting mechanism.
    - Alerts raised in past must be stored.
4. Usability:
    - The UI must be pretty.
    - Collaboration on workflows with teammates should be possible.

<!-- <blockquote> -->
**I called it "Automata". It is a reference to <a href="https://en.wikipedia.org/wiki/Automata_theory">"Automatons from Theoretical Computer Science."</a>**
<!-- </blockquote> -->

I wanted it to be an abstract and generic platform to create arbitrary and customizable automations which served any purpose. What I **did not want** it to be was another hardwired Attack Surface Management(ASM) tool for just security use-case.

<blockquote>
Basically not another ASM tool but a Low-code platform to CREATE ASMs or any type of such automations through UI.
</blockquote>

October 2020, I started deep diving into the components that I will be stitching together to build Automata. 

1. UI Design and Frontend App Development Frameworks.
2. Chosen CI and its underlying infrastructure.
3. Suitable Databases and design patterns.

On 22 November 2020, I submitted the first commit for the Automata repo with base code. It was the first commit of many.

![](/images/Automata-blog-github-activity-2021.png)
<p style="text-align: center; font-size:0.75rem; font-style: italic; margin-top: -10px;">(Github activity, Year 2021)</p>

At a very high level, the design looks like this:

![](/images/Automata-blog-Automata-HLD.drawio.png)
<p style="text-align: center; font-size:0.75rem; font-style: italic; margin-top: -10px;">(Automata HLD)</p>

I took a slow but consistent approach towards working on it and completed Automata's first MVP, in February 2022. I was about to invite some friends to try out the vBeta but then some high-priority tasks came and I was not able to get back to it. 

There is still some polishing work remaining on reliability, maintainability, and some minor features. But at this current moment, this is how the Automata looks :)

## vBeta Demo

<iframe style="margin-top: 10px;" width="100%" height="315" src="https://www.youtube.com/embed/Eq3A-o1Q5T4" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

<p style="text-align: center; font-size:0.75rem; font-style: italic;">(vBeta Preview)</p>

In the above video, you can see the gist of its capabilities. The workflows in the video are ["The Bug Hunter's Methodology" by Jason Haddix](https://www.youtube.com/watch?v=uKWu6yhnhbQ) and a standard github leak monitor.

And now, this is how easy it is for me to create new workflows ⬇️

<iframe style="margin-top: 10px;" width="100%" height="315" src="https://www.youtube.com/embed/l_qx-QxMl54" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

<p style="text-align: center; font-size:0.75rem; font-style: italic;">(Creating a sample workflow on Automata)</p>

<blockquote>
Automata makes it effortless to create arbitrary workflows. The other features i.e. Objects, Visualizations, Tripwires, etc. are complementary to workflows and make up for the functionalities other than automation.
</blockquote>

## Conclusion

After my senior year of college, I never got a chance to work on a full-fledged software project which did not involve security, all of my internships and work revolves around Security Engineering and Hacking.

For me, this side project was an enjoyable experience in Software Engineering and System Design. While developing I got to know about similar platforms, and it was a satisfactory validation of my ideas for Automata.

As no system is perfect, I would love to hear people's thoughts on how Automata can be better.

After looking at the demo, if you think Automata can be useful for you and want to try it out, or if you have any feedback, please reach out to me via this [Google Form](https://forms.gle/cXPdRdxJ2Bh3Hdx76) or social media ([Linkedin](https://www.linkedin.com/in/iamshoebpatel/), [Twitter](https://twitter.com/0xCaptainFreak)).

Thanks for reading till the end.

Best,
CF








