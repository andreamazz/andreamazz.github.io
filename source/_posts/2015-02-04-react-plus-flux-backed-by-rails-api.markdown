---
layout: post
title: "React + Flux backed by Rails API"
date: 2015-02-04 11:01
comments: true
categories: rails api react flux javascript
---
I’ve been working on a frontend for a project we are developing here at Fancy Pixel. We are embracing what looks like a good habit: slicing what would be a monolithic Rails app in a lightweight backend serving APIs and a frontend consuming them. We did this in the not so distant past using Angular.js. It was all fine and dandy, until it wasn’t. There’s something about it that doesn’t sit right with me, I wouldn’t go in detail, since many others already did, but let’s just say that there’s too much magic involved for my tastes (says the guy using Rails). Magic is fine as long as I can figure out how to tinker with the internals when things go down south. With Angular the effort seems too much, but that’s just personal taste really. Also I can’t deny that the major structural changes introduced in 2.0 were the last nail in the coffin.
I wanted to try something new, something that would enforce a solid architecture of our apps, letting me control the single cogs in the engine. React got a lot of good press in the past months, so I took the chance to dive in. In this three-part post you’ll find pretty much everything I learned by writing a frontend using React, with a vanilla Flux architecture, consuming an API written in Rails.  

Read the full article [here](http://fancypixel.github.io/blog/2015/01/28/react-plus-flux-backed-by-rails-api/)
