---
layout: post
title: "Sam's started a blog!"
description: ""
category: 
tags: []
---
#### Hi Folks,

So after attending and watching multiple conferences and talks I decided to create my own blog. Expect ramblings of a very passionate developer which go into detail about everyday pains we all feel in IT.

There were multiple contenders to what I can ramble on for my first post, there is the whole "Is TDD Dead" or shall I throw my hat at "Continuous Delivery vs Continuous Deployment" or maybe even give my thoughts on some Agile processes.

After much deliberation I decided it will be fitting to write about the issues I face everyday. Anyone who works in a team that has to collaborate with other teams, especially in a corporate environment will be able to relate.

Before I start my rant I feel like I should give all of you some background, up until a few months ago the process in our team was to write a feature then throw it over the wall to the QAs to test/automate. This means that code builds and deploys from TeamCity to a environment, afterwards the QAs come into play and run regression tests on it.

Everyday I argue over why we need to move the tests from Jenkins to TeamCity and why we need to fail fast! Passing a feature over to QAs only to wait days to have them push it back because it broke something that isn't related is not ideal.

As a developer I want to quickly know if a change I make works and more importantly if it breaks something else. Failing fast is one of the most important aspects of continuous deployment

After hours and hours of cleaning the tests and putting them through what my colleague called a "super lean diet program" I got the tests down from taking over 2 hours to about 25 mins. I will be doing another post about how I achieve that in the coming weeks

But for me it comes down to this, I want to fail fast and the term fast is integral to the whole point. If a test takes 6 hours to run then is it doing anything it doesn't need to do? like closing down and opening browsers for no reason, logging out and logging in with the same user and the same permissions

We need to understand what our tests do to understand our domain and vice versa. Any developer worth his salt will know that "Sleeps" are evil! a UI test should act as a user, what kind of user expects something to happen so goes to sleep for a minute or so then refreshes the page?

I am sure there are valid reasons for using sleeps but in my opinion these are few and far between.

If any of you have any QAs in your team who come from a manual background, the idea of automating tests might make them feel threatened and its easy to stick with what you know. However it is your mission - should you choose to accept it - to explain to them that automation is not meant to replace a QA, its to make their job much easier.

QAs can focus on what they do best which is exploratory testing. Humans are great at that! We think of strange and weird ways of using the application that would have never occurred to the original creators.

Basically by removing the repetitive element of manual tests you will make your team both more lean and efficient while also giving you far more confidence in what your shipping.

Remove the manual error prone testing and lets join our fellow evangelists into the wonderful word of autonomous testing!(TM)

Yeah that's right I just created my very on buzzword there ;)
