---
layout: post
title: "Why Internal RESTful API clients are evil"
description: ""
category:
tags: []
---
#### RESTful APIs and shared clients

This post was inspired by this [tweet](https://twitter.com/mbazydlo/status/722787637719920640). 

Over the years I have had the pleasure and pain of working with a variety of distributed systems. The current trend is to have a website or app with multiple APIs being called to retreive or take an action on data and I started to notice an anti pattern developing in our industry. 

Picture this, we have a team developing and maintaing an API, over time their consumers say 

"Its really hard knowing what urls we should call or how the model keeps changing." 

So the team suggests a solution 

"How about we publish a client for it that we reference and everytime we change the functionality we publish a new version, that way you can just update the package and viola! you have al the latest and greatest!'

On the surface that seems fine, so say Team A publishes 3 or 4 clients/packages for each API, then Team B does the same. Weeks - or even monts on some cases - go by and some developers say "we keep rewriting these clients and we have lot of duplication lets create a base client that all new clients derive from, you know, DRY and all that. Again this in principle sounds fine

#### All is not what is seems

 
 So now we have all teams using the same base client that implements all the required headers (if you are so inclined) and we have consistency throughout the platform right?
 
 Wrong! Because of the inherit way of how distributed systems work, you will have one or more APIs or even the website calling multiple APIs, and what happens if all of them are using a base client? you will have multiple APIs using various different versions of the base client. Not fun! 
  
 The next problem - while not directly caused by these base clients themselves - is that developers just say use this method on the client, we end up mixing and matching resource or in other words we eventually will have a culture of methods over HTTP instead of resources. It goes back to old school SOAP calls.
  
 For example, lets say you are calling Order details based on an order ID, so you have an IOrder Resource, if you have a base client it will be very easy to return a payment or a restaurant or even a user resource masking it in the Order Interface because it users order ID as its identifier
 
 While these are different bounded contexts, they can be different resources in the same bounded context being returned by the same API but since we focus on methods over HTTP that means we would return a resource from somewhere it shouldnt be called. This is especially true when working towards deadlines where we are more likely to take shortcuts, "We can refactor it later!"
 
 These sort of problems becomes very clear when you try and write tests for your client!
 
 This happens quite alot and can be easily avoided if the resource is correctly identified from the start.
 
 So thats it, my last bit of advice is if your designing an api make it easy to use, make it so easy that you can call it via CURL! 
 
 If you think about all the tutorials for the big company APIs out there, think Facebook or Amazon then a simple GET should be easy to make.
 
 Writing these base clients and packages is just a hack to avoiding making the API more accessible
 
 Thats it for now folks, comments and feedback are always appreciated!
 
