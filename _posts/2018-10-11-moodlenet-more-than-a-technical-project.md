---
title: "MoodleNet: more than a technical project"
description: "How Moodle’s core values affect our development process in MoodleNet project"
tags: [MoodleNet]
header:
  teaser: /assets/images/moodlenet_more_technical_project/live_work_create.jpg
  image: /assets/images/moodlenet_more_technical_project/live_work_create.jpg
  overlay_image: /assets/images/moodlenet_more_technical_project/live_work_create.jpg
  overlay_filter: 0.5
last_modified_at: 11/10/2018

---

As many of you already know, [I have been hired by Moodle](https://blog.moodle.net/2018/introducing-alex/) to work on the MoodleNet project. Moodle is the world’s most popular learning platform with more than 120 million users around the world. It is used by many universities on every continent. In fact, I used it myself during my time as a student of Computer Engineering. And, of course, **it’s an open source project!**

{% include figure image_path="/assets/images/moodlenet_more_technical_project/live_work_create.jpg"
alt="Live, Work, Create"
caption="Live, Work, Create" %}

The MoodleNet project is a little bit different, but of course, related to the Moodle main project. Our mission is to empower communities of educators to share and learn from each other to improve the quality of education. We are building a new open social media platform for educators, focused on professional development and open content.

For now, I have only worked on the project for two weeks, but I have already noticed big differences from previous clients and jobs. So I’d like to use this post to explain these differences and how they influence the technical aspects of the project.

## "Make the world a better place" is more than a slogan

A lot of workplaces write these kind of slogans on their walls. They think it is motivating for their workers. But do you know what is really motivating? Actually doing it. It is easier to feel like that when you write open source code, but we are pushing this mantra even further.

{% include figure image_path="/assets/images/moodlenet_more_technical_project/do_something_great.jpg"
alt="Do something great"
caption="Do something great" %}

We must have a prototype of the MoodleNet project ready in January. For this very reason, I just wanted to start coding immediately to make that possible. You know, if there is something that you learn in the world of technology, it is "build fast, fail fast". This is applied almost everywhere for MVPs, proof of concepts, prototypes, etc. However, things are a little different with MoodleNet. We’re not just building for one company, but for a whole community.

I have been learning about ActivityPub, which is an open protocol developed by the W3C for social networks. We are not just implementing this protocol for MoodleNet, **we want to create an ActivityPub generic implementation for the community**. In this way, the future developers will be able to create their own open social network more easily. How cool is that?

This way, we cannot fail. Even if the MoodleNet project does not achieve all our goals, we’ll leave a legacy and we’ll collaborate with other open social networks. This is very important and brings us to the next point.

## Philosophy in practice

We can architect MoodleNet in many possible ways. We could just create a centralized service where people log in and share their content, just like Facebook or Twitter do. This is much easier to code and to maintain. We didn’t choose this path because of our philosophy.

**We are not building just a simple project, we are building a more democratic world. Learning is an essential value in any society. We want to make MoodleNet in the same way, and as  democratically as possible**.

This is the reason we bet on open and federated social networks, sometimes called just the "Fediverse" (Federation + Universe). This allows interacting with users that are hosted in other servers in the same way you can do it with users which are in the same server.

A great example is the email service. We all use it on a daily basis and it is a federated protocol. When you send an email from your ProtonMail address I can receive it in my Fastmail. And they are not the same server, but it works! Now, it is the same with the social networks, you can follow me from your ProtonSocial server, you can receive my messages from your ProtonSocial server and you can like my posts from your ProtonSocial server. You don’t have to worry about if I’m using FastSocial, or ProtonSocial or anything else.

**Our goal is you can set up your own server, so you own your data**. When you use Facebook you don’t own your data. **If tomorrow Google+ decided to close its social network, all its users would lose their data**. They would not be able to keep using even if they want it.

In an open federated social network like MoodleNet, you will have the code and you can install it on your own servers. So, you can always keep using it with all the other users which are using your server, and with all the other users in other servers around the world. You don’t depend on a big organization anymore! You are absolutely free! And all of this without the need to data-mine and sell your data :)

{% include figure image_path="/assets/images/moodlenet_more_technical_project/mic.jpg"
alt="Microphone"
caption="You have a voice" %}

**Another big problem for the democracy that is solved by the open social networks is censorship**. Corporate owners of social networks decide what should and what should not go inside of their platforms. Most of them motivated by economic reasons such as providing shareholder value and you cannot do anything about it.

Well, here, in the fediverse, the rules change. It is more, "your server your rules". So, if you don’t like a server you can move to another. If in the future they change their rules, you can move to another one again. Finally, if you really need it, you can create your own server only by yourself. You can make censorship a thing of the past.
A Pluriform Social Network

Right now, we go to Youtube if we want to see videos, to Instagram to see photos and Medium to read something. While there is nothing wrong with have multiple accounts in different platforms, it may be a fragmented and frustrating. experience.

Thanks to the ActivityPub protocol, we have many more possibilities – more than could have imagined before I started this project! ActivityPub defines several entities, ie: "actors", "follower", "likes", "notes", "article", "videos", "photos", etc. If two servers implement ActivityPub, they know how to speak each other even if they are from different platforms!

This can be explained better with an example. Imagine you have an account in Mastodon, which is best described as a kind of "Twitter clone" in the Fediverse and you love the shows of one guy on Peertube (a Youtube clone). **The ActivityPub protocol means you can just follow him from your Mastodon account, instead of having to create a separate PeerTube account**. When your Mastodon server receives a "video" from Peertube, it may not only notify you, but actually present the message with an embedded video! The same is true of PixelFed (an Instagram clone), if you follow people there, any new photos can appear in your Mastodon timeline. Or imagine a blog which implements ActivityPub. Blog posts can appear as a regular message in Mastodon with the introduction and a link to read more! The same data can be presented in so many different ways! I have given the example of everything appearing in Mastodon here, but of course the reverse is true.

**The great thing about Open Source is that you always have a multiple options**. Let’s say that the Mastodon interface is not for you. You can use another client, which implements ActivityPub, to read your data and updates. Even more, you can actually use different ones. You can have one that is very useful for community managers, another focus on a very nice UX for reading and the last one is just useful to listen to your favorite podcasts. If you are interested, you can find an alternative interface for Mastodon in Pinafore.

**MoodleNet wants to be part of this Fediverse and we are going to support it creating as many tools as possible to encourage people to join us**. For this very reason, we are doing the other way around, we are going to build a nice and generic platform for ActivityPub and later build, on top of it, our MoodleNet project.

## Work openly

This is something I liked from the very beginning. Even before I was hired! When I was in the selection process, I did a little research about the company and the project. I was surprised they had a blog, and actually, it was regularly updated. So I saw many videos about the UX for the future prototypes. Then I could read about why they chose Elixir for the project and the next steps for the future backend developer, including the open source project they wanted to reuse. In addition,  there were details about frontend technologies, **why some decisions were made**, diagrams, a lot of interesting stuff.

{% include figure image_path="/assets/images/moodlenet_more_technical_project/together_we_create.jpg"
alt="Together, We Create"
caption="Together, We Create" %}

I didn’t realize but I spent like 3 hours reading content about the project. I feel like I knew everything about it!  It was funny then, in the interview, because I had to explain that I didn’t have questions, not because I wasn’t interested in the job, but because I had already done super research as a result of the team working openly.

What I mean is I felt like I was in the team and I understood why some decisions were made. This is a really important thing and I’ve missed in many open source projects that don’t share this knowledge with the rest of the world. Sometimes I don’t understand why some projects took one direction instead of another. **After all, we can learn a lot from other project experiences, technical and not technical things**.

Now that I’m on the other side, I see it is not easy at all. For example, I don’t know a lot of stuff about the Fediverse but I have to share with the world that I actually don’t know what I’m doing! Or maybe, I changed my opinion about something and I have to publish in the public issue tracker that my first decision was completely wrong.

It’s not easy to work so openly, especially at the beginning. You’re exposing yourself to the world. On the other hand, I see how the people actually understand your process and they empathize with you. It’s also cool to see how some people, which don’t work in Moodle, are interested in the project and collaborates with you. They give us their perspective and it is very enlightening, to see their perspective at the beginning of building a new project. So it is hard but it’s worth.

## Conclusion

So, as you can see, **Moodle’s core values affect our development process deeply**. It is more difficult to code a federated system that a centralized one, but at the same time, makes it more exciting, and more importantly, **we can have a broader impact**. We are creating a tool to empowering teachers to share content, impossible a better goal. Besides this, we are collaborating with open source community exposing our code and creating new open source libraries. Equally important, we are promoting technologies and protocols to make the world a better and more democratic place. All of this in just one project. And this is one of the reason because I love this job.

If you are interested in our project you can also [collaborate with us](https://docs.moodle.org/dev/MoodleNet/Contributing), although a simple comment is more than enough.

---

This post was also published in [MoodleNet blog](https://blog.moodle.net/2018/moodlenet-more-than-a-technical-project/).

Photos by [Jon Tyson](https://unsplash.com/photos/QL0FAxaq2z0),
[Clark Tibbs](https://unsplash.com/photos/oqStl2L5oxI),
[Ilyass SEDDOUG](https://unsplash.com/photos/c4lXkCHuaXY) and
[My Life Through A Lens](https://unsplash.com/photos/bq31L0jQAjU)
