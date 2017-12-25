---
title: Near-future is for human-computer hybrids
date: 2015-01-02 10:45:20
categories: philosophy
tags: [ai, productivity, future]
---

![customer support oracle](/images/magician.jpg "customer support oracle") 

Most of tech startups today try to be scrappy, to have many users and/or customers while keeping a small team. For some, this is the only way to survive and eventually become successful. This is possible because today's technology is cheap, powerful and enables us to automate a lot of daily tasks that previously required many people. 

_The ideas behind this post are based on the premise that at least for the foreseeable future this trend is not going to change._

### Automation and it's limits

Software (SaaS or otherwise) companies are usually the first to embrace automation. Payments, customer communication (newsletters, drip e-mails, etc..), deployment, automated testing,  statistical analysis, abtesting and other techniques enable such companies to stay small yet create and sometimes capture a lot of value. 
Taking a product from the "production line" and putting it into your customers' hands is a big part of the software company advantage, but there still exist a few areas that are not fully automated. 

### The human monopoly on creativity

The 'creative' jobs are mostly human and even though there are a few bots that rehash news articles, there is still a long way towards bots that can write software, write a good blog post or create a good website layout. Despite being an interesting subject to discuss, I will try to focus on another part of non-automation: customer support. 

### Customer support. How startups do it?

![old dude typing](/images/typist.jpg "customer support today")

Some companies, such as Google, provide only partial customer or no user support, but most startups today try to have a close relationship with their customers: they send non-automated e-mails to potential clients and the founders answer each customer individually. Doing unscalable things is not only normal, but strongly encouraged, at least in the beginning.  
This rightly gives the customer the impression of being taken care of and appreciated, which historically doesn't happen at big corp. Since the main focus of startups should be growth, this type of customer support is not very scalable. Meaning that the company has to hire people as their customer/user base grows.  

### The scaling problem

![super-charged customer support](/images/pushing-keys.jpg "super-charged customer support")

How can startups keep the customer support quality they used to offer at the beginning and yet still keep scaling up. The answer would be: automate more things. However automatically answering e-mails is a difficult problem and I believe that it can be ascribed to [hard-AI problems](http://en.wikipedia.org/wiki/AI-complete) 

### A simple example

Let's imagine a theoretical scenario of a customer called Anna that sends an e-mail to [support@gorgias.io](mailto:support@gorgias.io) 

> Hi, 
> After the last update the keyboard completion functionality stopped 
> working on Gmail. 
> Can you help me out? 
> Thanks!

Let's see some of the steps that are needed to solve her problem: 

1. I'm doing customer support so I read the e-mail and then try to reproduce the problem.
2. Let's say I reproduced it.
3. I physically go to the developer and show her the issue (or describe it in an issue tracker).
4. Once I do that, I reply to Anna saying that I managed to reproduce the problem, apologise for the inconvenience and then wait for a fix.
5. Fortunately, it's quickly fixed and the developer publishes an update and notifies me that it's fixed.
6. I return to Anna and notify her that it should be fixed.
7. She replies that indeed it seems to work well now.

![keyboard overload](/images/drowning-in-keys.jpg "keyboard overload")

There are a lot of steps (some of them might be missing here), not to take into account the hard work that is involved in finding the code and fixing the bug. A lot of information is not recorded between these steps and thus lost. Later on, it would be really hard to figure out, what the lifecycle of an 
issue is, just by looking at the exchanged messages between me and the customer and, internally, between me and the developer. I only we could build a really good AI, like in the movies, that could automate at least part of those steps.  

### The oracle

![customer support oracle](/images/magician.jpg "customer support oracle")

We can imagine an agent that is aware of the internal working of a company, an oracle that knows each customer's situation at any given time, that could even answer some easy questions to customers and demand clarification from its coworkers. Alas, this is something of a sci-fi domain for now. 

In the near-future however I think it's much more likely that we're going to have semi-intelligent agents (insect-level intelligence) that would help us with the task of doing customer support. They could be displaying the relevant information at the right time and even writing some part of the answer for us. This would make the experience of doing customer support more like editing than writing.  
It's hard to say, what the future will be like and what the problems we might encounter are, but I hope we're **not** going to fix it by throwing more man-power at these problems. 

A primitive form of the above post that is pushing in that direction is an extension for Google Chrome we built to write faster messages on the web. You can use it with Gmail, Outlook.com, Yahoo mail and many other websites.  

You can [check it out](https://chrome.google.com/webstore/detail/gorgias/lmcngpkjkplipamgflhioabnhnopeabf) (it's free)!