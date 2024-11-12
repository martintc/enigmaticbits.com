---
layout: ../../layouts/post.astro
title: 'Adventures into Ruby on Rails'
author: 'Todd'
pubDate: 2024-11-11
---

# Introduction

About a little over a month ago, the video for DHH's [keynote talk](https://www.youtube.com/watch?v=-cEn_83zRFw&t=3329s) at the Ruby on Rails conference caught my eye. I don't really do a whole lot of web development. This blog I have been throwing together with static site generators and now the current iteration as of this writing is in the [Astro framework](https://astro.build). Ultimately, most of my 'web application' experience has been using static generators and a few tutorials every once in awhile when I get an itch with react. But DHH's talk caught my attention. First, I have always been a little curious of just seeing what Ruby on Rails is. I have heard it a lot, but have never really looked at the code. Second, it seems like a perfect match for a web development idiot like myself. I can learn web development, but I always run into an issue with, I read some documents and do some tutorials, but I never have a serious project to do with it.

Well, that is not entirely true. but for the most part it is. I technically did a little bit of a web development project at work where I had to modify an old ASP.NET MVC that probably had not been touched in 4 or 5 years to allow for a new feature. However, I did not have to do much with the UI, it was mostly adjusting the controllers and some of the logic within them. Still, I don't necessarily count that. 

# How I got here

As timing would have it, a project came along right around the time I listened to this talk. My father in law recently took over his dad's business doing carpet cleaning and with that, taking over the website and such. While out visiting, he said he had some struggles. He isn't very tech literate either. I told him I can take a look and see if I can help. After all, I am a software engineer. So he pointed me to it and well, it was Godaddy. By Godaddy, I mean the domain is registered through GoDaddy and his site was thrown together by another family member via their point and click UI to build a site. 

Now, I am not a huge fan of those interfaces. Maybe, its because I write software. Maybe, it is because I have been a user of Linux and other Unix system since when you had to custom compile kernels if you wanted your WiFi to work. I am much more comfortable dealing with text and not dragging and clicking. Also, he was paying somewhere northwards of $400 a year for a domain, hosting a pretty simple site, and an email plan I guess he got suckered into that he never plans to use. So, I told him I could play around with some stuff, but if I am going to do any work on it, the drag and drop website builder has to go.

So, now I had a project, and I just recently learned about a new framework.

# How it turned out

Before going too deep into details and talking about how I solved some problems, I will get to the conclusion on how this went. My father in law has a new website hosted on a server I rent for other projects and it does everything it does. The server I rent has way more capacity than needed, so it doesn't really cost me anything to host it for him. It also saves him some money. He offered, but as I explained to him, whether or not his site is on it, I'd be paying for it anyways. Plus, I considered his payment for my service being his giving me a problem and a project I have been on the look out for, for awhile.

# Give me the dirty details on working with Ruby on Rails

So far, I feel like DHH makes a promise and that for the most part is true. That promise is a framework that makes it easy to work in once you get the hang of it. Also allowing a person to be a one person team. Which was perfect for this project. Instead of starting with 7.2, I started his website using 8.0.0-beta and living with it through upgrading it from 8.0.0-RC1 to 8.0.0-RC2 and recently upgrading it to 8.0.0.

## Why start with Beta

The main attraction was the authentication generation. I didn't really want to have to deal with something like firebase's auth or an external auth and relying on a free teir plan. There is nothing wrong with utilizing the free tiers, but it is also nice that I never have to worry about a rug pull. Granted, for users, his site could have probably sat in a free teir for forever. That is because as of right now, the authentication just gives access to a dashboard with functionality for him and my mother in law. Also myself just to double check things in production. So, at most 3 users roughly? 

The scripts for authentication were as good as promised I feel. It pretty much took care of all of that auth without me having to tear too much into implementation and RFCs if I were going to roll my own or learning how to work with an auth provider. In about 5 minutes, I had a working authentication system.

The only part of auth that took me a little bit of reading to work with was how to specify which pages don't require auth. Now, I am still not an expert, but from what I have seen, when you use the generate authentication system, it automatically assumes that every page, or at least pages produced by controllers, require a valid session token. To have a controller not require a session token, a declaration needs to be made after the class definition.

```ruby
allow_unauthenticated_access
```

## A design I was somewhat familiar with

Ruby on Rails makes use of the MVC (Model-View-Controller) pattern which I was already familiar with from a little bit of work I did on an ASP.NET project at work, it is the one mentioned at the beginning of this article. 

Don't consider this an endorsement of MVC. I do not do enough web development to know if MVC is a superior patterns to other. I do know that Angular is based on an MVC pattern. In this case, it is just a pattern that I had seen before and technically worked with before when working on a website.

## Mailers

This project was going to need some functionality with email. As of right now, there are three main use cases for working with email. First, supporting the 'forgot password' functionality in the authentication system. Luckily, if the email server settings are configured, the auth generation provided by rails pretty much does this for you. Nothing to do, it just rather works without a thought. Next, I built the ability for my father-in-law to have a sort of newsletter. This he can leverage pretty well. He lives pretty far north so for winter, the business shuts down because the equipment will freeze. At the very least, he can have a sort of newsletter to notify people when he will be closing down or opening up depending on the season. People can subscribe on the homepage of his site by inputting in their email. An email is fired off to confirm subscription. Then, from the dashboard, I built an area where he can input a subject and email contents to send out to everyone on the list. Right now, it is pretty bare bones, but it is a start. I need to guide him through, and make he knows, he can make requests and I will work on them. Or in other words, coming up with requirements that will help him better.

Mailers is probably a killer feature of Ruby on Rails. It just makes it easy. I have some experience of the hell with working with email from work. There is a lot to be said about SMTP, but I'll try not to air those grievances here. Like auth, rails can generate quiet a bit for you and also splits it into an MVC like pattern. A controller can control when and how emails are sent. Views are present that are the template for the email to be sent. Then configuring the email server is pretty straight forward from looking at the documentation. There is also a lot of stack overflow, blog posts, and other resources to help with this.

# Conclusion

All around, I am pretty happy with it. For a solo developer working on a project and needing a simple authentication system and mailers, Ruby on Rails makes it easy and I would recommend it.