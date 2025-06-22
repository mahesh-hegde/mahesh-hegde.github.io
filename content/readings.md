---
title: "Reading List"
date: 2023-08-27T22:37:36+05:30
draft: false
type: page
showTableOfContents: true
---

This page includes an infrequently-updated list of interesting articles I have read recently and think to be significantly impactful.

### Computer Science - General
These are generic - useful and educational software engineers regardless of what domain they work in.

* [__"Worse is Better"__](https://www.dreamsongs.com/RiseOfWorseIsBetter.html) - An essay which describes how the "perfectly designed" software loses to "worse-designed" counterparts. I think every software engineer should read this once.

    > The New Jersey guy said that the Unix solution was right because the design philosophy of Unix was simplicity and that the right thing was too complex. Besides, programmers could easily insert this extra test and loop. The MIT guy pointed out that the implementation was simple but the interface to the functionality was complex. The New Jersey guy said that the right tradeoff has been selected in Unix -- namely, implementation simplicity was more important than interface simplicity.

    > The MIT guy then muttered that sometimes it takes a tough man to make a tender chicken, but the New Jersey guy didn’t understand (I’m not sure I do either).

* [__"Use boring technology"__](https://boringtechnology.club/) - A very compelling argument that you should minimize your technology surface and stick to solutions which are well-understood. Even if a tool is not locally optimal for a given use case, it might be still globally optimal choice simply because it's well-understood.

    >  If you think about innovation as a scarce resource, it starts to make less sense to also be on the front lines of innovating on databases. Or on programming paradigms. The point isn’t that these things can’t work. Of course they can work. And there are many examples of them actually working. But software that’s been around longer tends to need less care and feeding than software that just came out.

* [__Go at Google: Language Design in the Service of Software Engineering__](https://go.dev/talks/2012/splash.article) - A talk by Rob Pike on design decisions involved in Go language.

    >  Note that these tools allow us to update code even if the old code still works. As a result, Go repositories are easy to keep up to date as libraries evolve. Old APIs can be deprecated quickly and automatically so only one version of the API needs to be maintained. For example, we recently changed Go's protocol buffer implementation to use "getter" functions, which were not in the interface before. We ran gofix on all of Google's Go code to update all programs that use protocol buffers, and now there is only one version of the API in use. Similar sweeping changes to the C++ or Java libraries are almost infeasible at the scale of Google's code base. 


* [__Command-line Tools can be 235x Faster than your Hadoop Cluster__](https://adamdrake.com/command-line-tools-can-be-235x-faster-than-your-hadoop-cluster.html) - Understanding execution models and operating systems is as important as building distributed systems that scale horizontally.

    > If you have a huge amount of data or really need distributed processing, then tools like Hadoop may be required, but more often than not these days I see Hadoop used where a traditional relational database or other solutions would be far better in terms of performance, cost of implementation, and ongoing maintenance.

* [__The grug-brained developer__](https://grugbrain.dev): A legendary article about simplicity and practicality in software development.

    > early on in project everything very abstract and like water: very little solid holds for grug's struggling brain to hang on to. take time to develop "shape" of system and learn what even doing. grug try not to factor in early part of project and then, at some point, good cut-points emerge from code base

* [__Towards Modern Development of Cloud Applications__](https://sigops.org/s/conferences/hotos/2023/papers/ghemawat.pdf) - This paper goes back to basics and looks at the ways we can reverse some of the complexity of kubernetes. It's written by the legendary Sanjay Ghemawat.

    > Fundamentally, this is because microservices conflate logical boundaries (how code is written) with physical boundaries (how code is deployed). In this paper, we propose a different programming methodology that decouples the two in order to solve these challenges. With our approach, developers write their applications as logical monoliths, offload the decisions of how to distribute and run applications to an automated runtime, and deploy applications atomically.

* [__A Eulogy for DevOps__](https://matduggan.com/a-eulogy-for-devops/) - or, how to not do DevOps.

    > This meant someone going through and doing all the boring stuff. Upgrading Kubernetes, upgrading the host operating system, making firewall rules, setting up service meshes, enforcing network policies, running the bastion host, configuring the SSH keys, etc. What organizations quickly discovered was that this stuff was very time consuming to do and often required more specialization than the roles they had previously gotten rid of.

* [__Algorithms interviews: theory vs. practice__](https://danluu.com/algorithms-interviews/) - Essay by Dan Luu, a bit preachy, but captures the point. Leetcode tests for very small part of DS&A knowledge, so next time you hear "We ask hard algorithm questions because we can't allow inefficiencies at our scale", you will know it's crap.

    > At the start of this post, we noted that people at big tech companies commonly claim that they have to do algorithms interviews since it's so costly to have inefficiencies at scale. My experience is that these examples are legion at every company I've worked for that does algorithms interviews. Trying to get people to solve algorithms problems on the job by asking algorithms questions in interviews doesn't work.

* [__In Defense of Not-Invented-Here Syndrome__](https://www.joelonsoftware.com/2001/10/14/in-defense-of-not-invented-here-syndrome/) - Adds some nuance to "Buy vs Build" decision. Here's a quote

    > Not so fast, big boy! The Excel team’s ruggedly independent mentality also meant that they always shipped on time, their code was of uniformly high quality, and they had a compiler which, back in the 1980s, generated pcode and could therefore run unmodified on Macintosh’s 68000 chip as well as Intel PCs. The pcode also made the executable file about half the size that Intel binaries would have been, which loaded faster from floppy disks and required less RAM.

### Computer Science - Specific topics
* [[Book] __"Java Concurrency in Practice"__](https://jcip.net/) - It's a well-known and classic book, worth reading even if you are not going to write concurrent java code every day.

* [[Book / OCW] __Mathematics for Computer Science__](https://ocw.mit.edu/courses/6-042j-mathematics-for-computer-science-spring-2015/pages/readings/) - I would say the proofs section is the most important for someone in CS. Proofs and Discrete math not emphasized enough in Indian schooling.

* [[Book] __Beej's Guide to Network Programming__](https://beej.us/guide/bgnet/) - Perhaps the best accessible resource on sockets programming.

* [__Git from the bottom up__](https://jwiegley.github.io/git-from-the-bottom-up/) - Git internals.

* [Crafting Interpreters](https://craftinginterpreters.com/) and [Architecture of Open Source Applications](https://aosabook.org/en/) are good too, but I never managed to complete even half of these books!


### Lifestyle, Productivity
* [__"Why do Ivy League students self-sabotage?"__](https://movingthelimit.com/why-do-ivy-league-students-self-sabotage/).

    > We're sheep, selected for our ability to jump through contrived academic hoops.

* [__"Deep Work" (Book)__](https://books.google.co.in/books/about/Deep_Work_Rules_for_Focused_Success_in_a.html?id=Uc6RzgEACAAJ&source=kp_book_description&redir_esc=y) - This book explains how much productive work we lose due to distractions and shallow, fungible work, and also motivates the reader to adapt a lifestyle of deep, uninterrupted work.

    >  To simply wait and be bored has become a novel experience in modern life, but from the perspective of concentration training, it’s incredibly valuable.

### Humour
* [__The Hustle__](https://www.youtube.com/watch?v=_o7qjN3KF8U&vl=en) - A satire of busy techie lifestyle.
* [__A Brief, Incomplete, and Mostly Wrong History of Programming Languages__](http://james-iry.blogspot.com/2009/05/brief-incomplete-and-mostly-wrong.html)
* [__Senior Engineer__](https://www.youtube.com/watch?v=eSqexFg74F8) - Another satire (or maybe not)
* [__n-gate__](http://n-gate.com/) - Satire about hacker news threads
