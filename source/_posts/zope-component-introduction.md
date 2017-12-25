---
title: zope.component introduction
date: 2011-11-23 17:28:48
categories: python 
tags: [guide, python, tutorial, howto, zope, zope-components]
---

Recently I gave a talk about zope component architecture at a company in Paris where I started consulting on zope, plone, django and python in general. In these posts, aided by code examples, I will try to show what  zope components are capable of, what are the common use-cases and some of my personal views about the subject.

First lets identify the problem that we are trying to solve using a component architecture:

Applications tend to get big. Especially those applications that have a lot of users and have years of development. Users want different features/changes in different stages of the development of the product. Hence the key to a successful and long lasting application is to comply with the user needs as fast as possible with as little technical dept as possible. Finding a balance between the two is not easy.

Speed of development is important especially in the beginning of a project. Your clients want that software today and you can’t waste time on thinking a lot about long term benefits if your bank account is empty. Those benefits are nonexistent if you don’t finish in time so the shortest path usually gives you the best chances of success. This is the correct way to think about it. However when the project is getting bigger you may want to start thinking in different terms. Questions like these are usually good ones to ask:

- How much time does it take to create a feature?
- Refactoring is hard, but can it be made faster?
- Is it easy to extend existing features?
- How different parts of your application talk to each other?
- How about debugging?

These questions matter both to managers and coders. Even if the managers and developer are very good at what they do usually what happens is that layer after layer gets added on top of the existing product until it becomes a big pile of crap – new developers are screwed because they can’t figure out what is happening in the system and the time to train them is very expensive. New features take enormous amounts of time to develop and your technical dept gets bigger and bigger with every new modification.

Very little can be done about this once you’re big. There are no easy solutions and old applications sometimes just whither and die. Tough. But maybe components can help.

<!-- more -->

## Prelude

Python is a great language. It is easy to learn and there are many different frameworks that can get you started really fast. What I found very important about python is that it’s a good prototyping language – some complain that it’s too good. It’s no accident that it’s the language of choice for many successful startups (Dropbox, Disqus, Quora, etc.).

So once you start delivering and get some sort of income you try to improve the product to make your users happy. You provide missing features, improve stuff. More code, more complexity, more tests, more features, more developers and so on and we get back to our problem.

Zope components architecture promises to alleviate some of the problems of big applications by using adapters as a way to organize code. If this is true for your case you will have to decide for yourself.

#### What’s an adapter?

{% blockquote Wikipedia: https://en.wikipedia.org/wiki/Adapter_pattern Adapter pattern %}
In computer programming, the adapter pattern (often referred to as the wrapper pattern or simply a wrapper) is a design pattern that translates one interface for a class into a compatible interface. An adapter allows classes to work together that normally could not because of incompatible interfaces, by providing its interface to clients while using the original interface.
{% endblockquote %}

In other words, an adapter (usually a function or a class) allows two interfaces to talk to each other. Consider this example:

Say we have interfaces `iA` and `iB`, receptively `A` and `B` their implementations. At the beginning they don’t know about each other. But then you decide that you want to make them talk to each other. Let’s say `A` wants to list all Bs. At the same time you don’t want to change A or B because they are central to your application and a lot of stuff uses both `A` and `B` and changing them can be both ugly and difficult. This sucks!

The good news is that in some cases this can be avoided by using a third party that acts as a intermediary (the adapter) between `iA` and `iB` and provides functionality that did not exist before. Not only that, but it allows you to keep your `A` and `B` unchanged.

Let’s forget about interfaces and adapters for a while and let’s consider this simple scenario of apples and oranges in a basket:

{% gist 1388813 %}

It’s pretty clear that Apple doesn’t have slices and the last line will fail with an `AttributeError`.

So how do we make apple have slices? There are many possible solutions. But let’s impose these 2 restrictions (I will explain why later):

We are not allowed to modify the Apple class
We are not allowed to subclass the Apple class.
A possible solution is to create a wrapper class around the Apple class that will give us the functionality we need.

{% gist 1388877 %}

So let’s see what we did here.

We had an Apple class which didn’t have any slices
We created an AppleWrapper which takes an Apple object and provides a `slices()` function which can be used later with that apple object.
As far as `get_slices()` is concerned it just needs to call a function called `slices()` on basket’s items.
We also didn’t subclass the Apple class and most importantly Apple class remains untouched.
AppleWrapper is what we call an adapter and the apple object is the adaptee.

#### Why it is so important not to subclass?

For a longer explanation of why subclassing is considered harmful search for “inheritance is evil”. The short explanation is that we don’t really need to inherit all Apple’s properties and methods – we just need to make it have slices and that’s it.

#### Why just not modify the Apple class to give it slices?

Because we don’t really need the Apple to have slices all the time. Just the time when it’s in a basket. This way the Apple class will stay a clean and simple – and as a result easily extendable in the future.

Ok. Now that we understood how an adapter works we can switch to a more complicated example and this time we are going to use `zope.component` package.


### zope.component

`zope.component` is a reusable python package that usually comes with `zope.interface` and `zope.event`. We can split this package into two categories:

- Components: adapters, utilities, named adapters, multi-adapters and others
- Registries: global component registry, local component registry.

What we need to remember from all this is that there are registries that store adapters. The idea is to use these registries through out our application. Think of it as a global dictionary with values as adapters and keys as interfaces – because it’s exactly what it is. Without further ado we’ll jump directly to a simple example, but instead of adapters will use an utility (an adapter that adapts nothing).

A small utility

{% gist 1391210 %}

Let’s see what we did here:

We declared an interface and an utility class Hello.
We got the global registry and registered our utility for the ISpeach interface.
In the end we used our utility first by querying the register and then just used it as a Hello object.

#### The benefits

The nice thing about registering a utility like this is that you can change its location in the project, you can refactor it and give it a different name, you can override it with another utility at runtime – and most importantly is that the last 2 lines never change! 

Here is why:

All we really care about is the interface with which we are making the query – the implementation can change in the future.
The code is decoupled (no direct imports to the implementation) – just the interface is known to us.
A direct result to this is:  more control for the developer which can replace the implementation at will without worrying that he/she needs to refactor a lot of code. Looks like a lot of code just to register one simple utility isn’t it? I couldn’t agree more, but remember that we are talking here about big applications with tons of utilities and adapters and events and event handlers. In the end everything is a structured chaos. It’s structured because as long as we have a registry that has all the components we have an overview of our application and all its components.

But my application is already far in it’s 3rd year of development. I don’t have everything in one big component registry!

The good news is that you don’t need to have everything as components – I strongly advice against it actually. You can use this just for the parts you feel that your application will benefit the most. You can gradually use it where it makes sense.

Now let’s return to our apples and oranges example, but make it use adapters.

Oranges and Apples – again

{% gist 1391374 %}

It’s really not that different from our previous example with apples and oranges except the whole interfaces and registration stuff. The benefit however is that this is a lot more usable in the long run for the reasons I explained above. And the difference from the utility example is that it actually adapts something – `IApple` in our case. The key thing to remember is that all the registry queries are done using interfaces.

### In conclusion: a word of advice (this part was added after 6 years): 

Now that you understood how to use the adapter pattern and why it can be useful I must add that you'll rarely need it in real world development and when you do it's usualy a sign that you have to structure your application better.

The power of adapters is also their greatest weakness, the part about hiding implementation behind a registry can lead to some pretty nightmarish debug situations. No imports also means that your tools will have a hard time finding the right adapter.
