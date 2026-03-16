---
title: Revisiting Docker Management and Troubleshooting
date:
tags:
  - homeprod
  - docker
  - logs
---

{% image '/img/dockhandgeneric.png', '' %}
<br> </br>

In this post I revisit the topic of homelab Docker container management and troubleshooting. I introduce a few new to me Web User Interface-based (WUI) tools. I'll dig into why I've moved away from Terminal User Interface (TUI) tools towards the more complex architecture of an agent-based management console with Dockhand.

**Overview in Brief**

- {% linkprimary "Why WUIs", "https://christopherbauer.xyz/blog/docker-trouble2/#why-wuis" %}
	- {% linkprimary "Dozzle", "https://christopherbauer.xyz/blog/docker-trouble2/#dozzle" %}
	- {% linkprimary "Arcane", "https://christopherbauer.xyz/blog/docker-trouble2/#arcane" %}
	- {% linkprimary "Dockhand", "https://christopherbauer.xyz/blog/docker-trouble2/#dockhand" %}
- {% linkprimary "Conclusion", "https://christopherbauer.xyz/blog/docker-trouble2/#conclusion" %}

## Why WUIs
In a [previous post](https://christopherbauer.xyz/blog/docker-trouble/) I lamented the state of tools for troubleshooting Docker containers. It seems that all I needed to do to solve my problem was broaden my horizons from TUI-based to WUI-based tools! In the WUI space we're spoiled for choice. Many are new to me, though according to {% linkprimary "Ethan Sholly", "https://selfh.st/post/2025-favorite-new-apps/" %} Arcane came out in 2025, so some are new to many as well. Sholly points out elsewhere that there has been an explosion of these kinds of apps recently, with a lot of overlap in capabilities. I've tried a few, but I am not making any claims about this post being exhaustive. 

Why move away from TUIs? The simple answer is that I have a growing homelab container environment. I tended to favor the simplicity of TUIs over an orchestration console when I had a half dozen containers because I found it easier to SSH directly into a single server and start up a TUI. I favored [Oxker](https://github.com/mrjackwills/oxker), for no other reason than the spatial layout made sense to me, and it was more organized than using Docker native commands.

Now that my containers have grown beyond a dozen, and they exist on multiple different machines, the complexity of command-line (CLI) operations has increased my cognitive load. When I began looking into other Docker container management options, the WUI came as a breath of fresh air in managing this distributed environment.

Below I list out a few of these options in the order in which I encountered them. They all offered "free as in beer" versions at the time of this writing. In keeping with the rest of my home lab, I self-hosted these Docker containers using compose files.

## Dozzle
{% image '/img/dozzle.png', '' %}
<br></br>

I was initially drawn to {% linkprimary "Dozzle", "https://dozzle.dev/" %} because it seemed like a lightweight WUI and it offered insight into each container's logs. This is nice over a TUI since I was able to quickly switch between container logs. Previously, using Oxker, I could switch between logs without a mouse but the interface was slow and a bit cumbersome.

Generally, I'm not crazy about the way Dozzle organizes containers into groups and loose containers in the left-hand pane. Grouping like containers together seems like a peevish thing to criticize, but when I'm troubleshooting I tend to want to see all containers at a glance, specifically in alphabetical order please.  Of course, you can simply unfold the individual containers from the grouping. I acknowledge this critique is a bit nitpicky, but while Dozzle was a good first step into WUIs, it just wasn't for me. 

In my research I only got as far as how to run a single Dozzle instance for a single Docker host. 

## Arcane
While I was initially interested in {% linkprimary "Arcane", "https://getarcane.app" %} for its increased features and slick dashboards, I became disenchanted with it in short order. In particular, what I liked about Arcane was that it offered more detail on the host's resource usage than Dozzle. Arcane had deeper insights into volumes and images that Dozzle lacked as well. However, despite those advantages, I found something a tad clunky about the way that Arcane displayed information and integrated different displays into a workflow. I don't have detailed criticism here beyond the UI not sitting well with me. I quickly switched from Arcane to Dockhand, so I'll leave it at that.

## Dockhand
I like two things about {% linkprimary "Dockhand", "https://dockhand.pro/" %}. First, the log search within any container and, second, the Hawser-agent that can provide state and control over multiple Docker server hosts. The former feature alone makes Dockhand stand out against the other two apps in this post. While having access to logs makes WUIs nice, having a search on logs facilitates finding errors instead of manually sorting through them. That alone is enough to push Dockhand ahead of the pack.

With the Hawser-agent feature, Dockhand also moves into a higher class by controlling multiple containers on different machines. This is nice if you are looking for a single-point of management. Combining these two features, I can have relatively quick and painless access to logs for any Docker server, whether in my homelab or on a VPS, and isolate errors quickly from one space.  I haven't found the agents to be resource-heavy and they aren't difficult to install as they have their own Docker Compose files.

Among the other features I use regularly is an automatic check for container updates that has changed up my approach to patch management on containers. Dockhand has several other features, including Git management and shell access to each container, but I have yet to explore these.

Finally, Dockhand has a few well-known themes to choose from, like Gruvbox and Tokyo Night. So you don't have to stick with the default white if it's not for you.

## Conclusion
As my homelab has evolved, I can count the transition to WUIs as one of the surprise developments. I never thought I'd prefer a WUI over a TUI, to say nothing of an agent-based setup.  However, I can safely say I've transitioned to these more complex WUI options. I'm currently pretty happy with Dockhand and plan to stick with it for a while. If you need a lightweight WUI, I can recommend Dozzle. Arcane represents for me a purgatory position of not quite working at the level of a console. That said, if Dockhand is too feature-rich for you or you simply don't like the layout, you might give Arcane a try.

