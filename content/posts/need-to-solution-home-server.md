---
author: ["Thariq Shah"]
title : 'From Need to Solution: Home server'
date : 2024-09-01T12:23:17+05:30
draft : false
ShowToc: true
cover:
  image:
categories:  ["home server"]
tags: ["home server"]
series: ["Home Server"]
searchHidden: true
---

## What is a Home Server?

Let’s start with the basics: what exactly is “the cloud”? It’s just someone else’s computer sitting in a data center somewhere—likely in a different country or even a different continent—managed by your friendly neighborhood cloud vendor (think Google, Amazon, Apple... you get the idea).

Now, about servers. The word “server” might sound intimidating, but it’s really just a computer that’s designed and dedicated to doing a specific job. And yes, your smartphone can technically be a server. Heck, if you’re really feeling adventurous, you could even turn your toaster into one (please don’t try this at home, though).

So, what’s a home server? It’s a computer you set up in your own space for a specific purpose, quietly working away in a corner of your house, rarely complaining (except maybe to send you an alert when something’s gone haywire).

People set up home servers for all sorts of reasons: managing security cameras, storing media, experimenting with Docker and Kubernetes, running virtual machines—the list goes on.

## What Could Be a Home Server?

If you’re just getting your feet wet, the best candidate for a home server is that old laptop gathering dust in your closet. Throw any Linux distro on there, set up SSH, and boom—you’ve got yourself a home server.

Once you’ve mastered the basics—Linux commands, terminal work, networking—then it’s time to start looking into more reliable, purpose-built hardware for your server. But remember: everyone’s gotta start somewhere, and an old laptop is a perfectly respectable place to begin your journey.

## Why Did I Need a Home Server?

I dabble in hobby projects because, well, boredom. When I get an itch to try something new, I need a place to experiment. Whether it's spinning up Discord bots or coding Telegram alerts to let me know when my favorite game map rotates, I need these things to run somewhere.

Sure, I could use the free tiers offered by cloud providers, but they never quite give me the flexibility I need. Plus, they usually require a credit card, and pricing models can change unexpectedly—just look at what happened with Heroku.

### - Deploy Hobby Projects

As software engineers, it’s crucial to truly understand our craft. At work, we’re often locked into specific requirements and environments, with limited opportunities to mess around with the latest and greatest tech. That’s why side projects are essential—they help us stay sharp, push the boundaries, and get hands-on with tech that we might not encounter in our day jobs. Go wild!

### - Learning

There’s always a new library, framework, or tool that’s begging to be explored. And the best way to learn is by doing—implementing it, breaking it, and figuring out how it behaves in the real world. This is where your home server shines.

### - Media Storage

Projects like [Jellyfin](https://jellyfin.org) or [Navidrome](https://www.navidrome.org/) can help you manage your media collection. Plus, with a home server, you can access your media from any device you own—TV, phones, tablets, you name it.

### - File Storage

One of my main use cases was keeping my files close, not with some faceless corporation. As my file sizes grew, Google kept asking for more money, and there’s always that lingering fear that one day they’ll change their policies or jack up the prices.

## How Do I Access My Home Server Remotely?

It quickly became clear that sooner or later, I’d need to access my home server remotely. But how could I do that without being at home?

Port forwarding is a non-starter if you’ve got a dynamic IP and you’re behind CGNAT. I started looking for free (or at least affordable) ways to make it happen.

- [Nord’s Mesh VPN](https://nordvpn.com/meshnet/?srsltid=AfmBOoodrZbnkUmdHgI17Ou7BrGLT3vFFtDj6pFsQ16Jm6dOEQhlTiHn)
- [ngrok](https://ngrok.com/product/secure-tunnels)
- [Cloudflare Zero Trust Tunnels](https://www.cloudflare.com/en-in/products/tunnel/)

I went with Cloudflare Zero Trust Tunnels because of its convenience and set up my domain with cloudflare to route traffic to my home server. Easy-peasy.
