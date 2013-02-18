---
layout: post
title: "Transaction Processing   Key concepts and considerations"
description: ""
category: 
tags: []
---
<strong>Introduction</strong>



Given the extreme requirements on business solutions these days, the enterprise software ecosystem has become quite large and complex. I am always interested to see how technology divisions make their technology choices, from very strict process all the way through to pure emotion. There are roles that are supposed to be able to guide these kinds of processes (such as Chief Technology Officer) and there are many good commercial stacks available like Oracle and IBM. As a technologist sitting on my Mac at home, I don't have the money or infrastructure to even evaluate most (if not all) of these stacks. And so, I can simply hope to gain key insight into building enterprise solutions in my day to day or I need to take another approach. Now, obviously I get exposed to these kinds of stacks in my day to day, but that isn't really the best place to learn and experiment and grow beyond the constraints of the current technology choices and project demands.



Unlike a few years ago, however, I can still experiment and learn and grow, thanks to open source and readily available cloud infrastructure such as Amazon EC2. I am not pushing Amazon, or open source. The economics of open source versus not is well understood and widely documented, and there are many drivers that many push you in either direction and I believe that choice should be made in a mature structured manner for your organization. But for my personal purposes I need a complete open source Enterprise stack to play with.



With that out the way, what is an enterprise stack? Well, that really depends on the particular industry, company and requirements. So for the purposes of this discussion I will constrain things to financial services, particularly a traditional bank. And we can then suppose that, at the very least, this list of open source solutions would be required to be provisioned, integrated and managed in order to deliver some basic capabilities to the business units and customers that need them, for the purposes of this article we will also ignore COTS systems(Pre-Built payments systems for example). We will ignore COTS because the purpose of the article is to discuss building, not configuring and secondly there may just be quite a trend in the future towards build and away from buy (despite the current commonly accepted knowledge). Ok, so the open source capabilities:

<ul>

	<li>Core transaction processing</li>

	<li>Reporting and analytics</li>

	<li>Human workflow</li>

	<li>Case management</li>

	<li>Data wharehousing</li>

	<li>Cloud infrastructure and associated management and monitoring tools for that layer (You don't need a cloud, but it is useful so work with me here)</li>

	<li>Data integration</li>

	<li>System integration/ESB</li>

	<li>Master data management</li>

	<li>Single sign on</li>

	<li>Portal to present web layers associated with particular functions</li>

	<li>Log management</li>

	<li>Business activity monitoring</li>

	<li>Monitoring and management</li>

</ul>

And the list could go on for much longer.



What has always struck me is the that they key element within this stack is always the least understood, being core transaction processing. Everything hinges of a strong core transactional capability, everything else is there to provide operational support. But the key element is the ability to process transactions deterministically, at scale and enable core value propositions.



The purpose of this blog entry is to put forward my thoughts on what is required for core transaction engine(s) and what concepts need to be considered before defining its requirements or design. The bulk of the post will talk to technology agnostic concepts, but I will put forward some open source options to consider later, of course.



<strong>Definitions</strong>



Ok, before we dive into the details, lets just clear up the most important and most overloaded term...



<strong>What is a transaction?</strong>



Put simply, a transaction is a unit of related work. This is very vague, which is fine because it lets us tailor the meaning, depending on what we are doing. So, here are some transactions:

<ul>

	<li>In the banking or payment channel world a transaction consists of receiving a payment instruction and processing the funds across both accounts listed in the instruction.</li>

	<li>In the securities world, a transaction is the complete execution of a deal against a listed security, including the moving of funds and the change of ownership of the instrument.</li>

	<li>In the social networking world a transaction is the posting of a new tweet or status update, the associated billing and analysis gathering and the finally making the update available to all associated and subscribers.</li>

	<li>In the database world a transaction means a sequence of information exchange and related work (such as updating) that is treated as a unit for the purposes of satisfying a request and for ensuring database integrity</li>

</ul>

What is important to note here, is that a transactions boundaries vary from case to case and many transactions can make up a transaction at a higher level of abstraction E.g. in the case of a payment, the business level transaction is potentially made up of the following "sub" transactions: the posting to the debit account, the posting to credit account and finally the acknowledgement across whatever channel instructed the payment.



So for the purposes of this post I will use the following convention:

<ul>

	<li>A transaction means the business level transaction, the key domain concept that we are trying to process</li>

	<li>All other usages of the term will be explicitly qualified E.g: the database transaction.</li>

</ul>

<strong>Concepts to Consider</strong>



There are many things that one needs to consider when approaching transaction processing. The first and obvious one is a key understanding of the particular problem domain you are partaking in. I don't want to harp on about requirements management, customer affinity, agility and domain knowledge here, suffice is to say that if you don't understand what you are trying to solve you will land up with the wrong solution.



With a clear understanding in mind the key issues to consider next are the functional requirements, data consistency and the non-functional requirements. The non functional requirements are often overlooked, but here we need to understand the actual throughput, latency and availability requirements of the system (among other non functionals).



<strong>Fact: The free ride is over</strong>



Despite Moor's law, the humble processor and hard disk can't keep up with the ever increasing demand on systems. We can no longer think of solution design in terms of single systems, we have to think in terms of distributed systems and massive concurrency. This is not only a factual constraint, it is also an economic consideration. Is Oracle RAC really required or would a cluster of low cost hardware achieve more?



<strong>The CAP Theorem</strong>



The CAP theorem states that it is impossible for a distributed computer system to simultaneously provide all three of the following guarantees

<ol start="1">

	<li>Consistency: All nodes have all the same data at the same time (implies ACID type of semantics)</li>

	<li>Availability: grantees that every request is serviced</li>

	<li>Partition Tolerance: The system continues to function in the event of partial system failure or message loss</li>

</ol>

Popular approaches up until now have favored consistency, in fact they have made it an absolute focus, extending to try and guarantee that no data can be lost or inconsistent. In practice such guarantees are impossible.



Essentially what the theorem means is that we have to sacrifice something if we want to achieve real scale and high availability.



<strong>ACID vs BASE</strong>



Ok, time for a little bit of technical transaction theory.



ACID stands for Atomic, Consistent, Isolated and Durable. This means that a transaction must be fully rolled back or committed, it can't be anywhere inbetween. It also means that transactions run independently of each other, but once the technical transaction has committed then all copies of the data are consistent. This is the standard model we are used to as software developers, especially those who use relational databases and technologies like Java. Concepts like XA and two phase commit are ACID implementations of transaction management.



XA and related technologies make our lives quite easy at design time, largely because of the amount of mature framework to achieve these things (the implementations are mostly not simple at all, in fact quite complex). There are however a few inherent issues with these kinds of approaches:

<ul>

	<li>They can never guarantee to be atomic or consistent, but I will admit that they get extremely close</li>

	<li>They consume huge system resources in achieving 2 phase semantics. This has an obvious performance impact</li>

	<li>ACID technical transactions aren't necessarily associated with relational databases, but most of the tooling is built for relational databases, and the moment relations and consistency are in the picture then locking becomes necessary in order to synchronize the various actors in the systems, be they threads or separate nodes in a cluster.  AND LOCKING IS EVIL! It kills scale and performance.</li>

</ul>

It is for largely these reasons that typical enterprise stacks achieve only hundreds of transactions per second, despite massive investments in hardware. There are ways to achieve ACID semantics on technical transactions where the throughput is in the thousands and millions of transactions per second, but more on that later on.



BASE is a nice term that was coined to give us the old ACID or BASE comparison from science. BASE in this context stands for Basically Available, Soft state, Eventually consistent. This represents a mind set change on a few fronts. Firstly there is no concept of a rollback, it simply doesn't exist, which means if it didn't happen we don't know about it immediately in the same way as we would in say XA. This means that we have to compensate or failures in other ways, like compensating transactions. These are therefore specifically created for business scenarios of importance. This sounds onerous, but in practice it isn't because the technologies are made to be highly available and so these cases are rare, but more importantly we only deal with a few failures pessimistically. Where as XA takes a pessimistic approach on each transaction (causing all that overhead), we are forced in this way to only do that when required, giving us huge performance gains. The second mindset change is that of eventual consistency. But before we dive into the consistency discussion lets just talk about state quickly...



<strong>State</strong>



<strong>What is state?</strong>



I often interview candidates for development positions and try to phase questions around state to try and gauge their conceptual understanding in this space. There are so many different perspectives, largely because developers have come across the term in so many different contexts. In the old EJB days people discussed stateless and stateful session beans, people discuss state at a POJO level (and terms like immutability get thrown in), functional programmers love talking about immutable object exchange and the resulting gains, when we start talking about long running concerns we have put the state into a file or a queue or a database.



So what is state? State is all of those things, or more correctly all of those things are state. State is essentially a piece of information that is left behind after a transaction has completed, with the intention that this information gets used in later transactions or other functionality (we could argue that all other functionality is a type of transaction, but I won't do that here).



It is the class member variable.

It is the database table entry

It is the momento on the disk

Etc...



<strong>So?</strong>



So, why do we have to be so concerned about state. Well, firstly from a design principles perspective encapsulation is about hiding implementation and state, so we have to understand what state is before we can achieve encapsulation and without that we can't achieve low coupling. Secondly we need to understand how state is accessed and updated, because this effects how we scale and finally we need to consider the consistency of the state. When changes occur in our state, who needs them by when?



<strong>Is consistency really required?</strong>



The term eventual consistency usually engenders some level of fear among technical and business users alike. How can different parts of our system have different versions of the data? Lets go through a use case and apply some critical thinking to each.



Lets start with user interactions, specifically when using a think or rich client application in which the presentation is performed on the client machine or browser (AJAX, Flex/Flash, etc…). There are typically 2 interactions patterns between the client and server. Namely synchronous and asynchronous. I would like to focus on asynchronous given that synchronous isn’t the best approach, but the point is applicable in either case. Consider the following diagram:



<a href="http://quintona.blog.com/files/2012/09/AsynchUserInteraction1.jpg"><img class="alignnone size-full wp-image-6" src="http://quintona.blog.com/files/2012/09/AsynchUserInteraction1.jpg" alt="" width="934" height="687" /></a>



This is an extremely simplified example of how data might flow through some potential layers. The data will only exist on a single data node until the replication of said data has been completed to the other data nodes, meaning that the system is in an un-consistent state for some period of time. Lets now understand the impact of that. The customer, being the important player in this scenario isn’t affected at all because consistency is in place within the context of the session.  The data is then going to be used by the Management Information System (MIS) for management reporting and the FI system to post to the ledger. But neither of these tasks requires that the data becomes consistent immediately, when the data becomes available to them on their data nodes then it will be visible in the books and management reports. So, consistency doesn’t really matter there.



There are of course some use cases where eventual consistency technical can’t be tolerated, such as trading platforms or stock exchanges, but it must be noted again that the consistency is only required within the stateful component making the decisions. Other consumers of the data such as billing and reporting can consume the data eventually in an appropriate structure.



<strong><em>Command Query Responsibility Segregation</em></strong>



I won’t go into too much detail here, because people like Martin Fowler have done a much better job than I could at explaining (http://martinfowler.com/bliki/CQRS.html), however the key point for me is that we really need to consider the purpose of the model we are busy building and don’t assume that we have to have one large shared model, especially when we take the eventually consistency considerations into account.



<strong>Just the right amount of consistency</strong>



Obviously you can get varying levels of consistency. Consistency in this landscape is also a very subjective thing, depending on where I am and what I am doing. Most modern cluster technologies will allow you to choose the level of consistency you would like to achieve for a given action:

<ul>

	<li>Read local: meaning that I want whatever is on my visible node and if it is stale, that is fine</li>

	<li>Read 2 or more: read a certain number of nodes and make the data consistent between them before giving it to me</li>

	<li>Read quorum/local quorum: make the data consistent on the majority or nodes, either globally or in the current data center (assumes topology knowledge)</li>

	<li>Read ALL: make all nodes consistent first</li>

</ul>

Similar concepts can be applied to the writes. The importance here is that we can tweak our consistency, but whether we should or not depends on our problem domain and a key understanding of it and our architecture.



<strong>Concurrency considerations</strong>



<strong>How can we achieve scale?</strong>



Scale is either achieved through vertical or horizontal scaling. Vertical scaling involves deploying more process on a single node, more threads or instances of a service, often this essentially means buying bigger hardware. Horizontal scaling involves deploying processes across many nodes in a cluster. This is also achieved in 2 ways, depending on the state and the transactional semantics applied to it. Firstly we can do a functional segregation, meaning that we divide the problem domain into portions and assign different portions to different nodes. Think partitioning in the database world. This of course doesn't allow us to scale linearly (as load increases just add nodes, 2 nodes process twice as much as 1 node). The second approach is to deploy the same functionality across many nodes, but this brings it own set of problems, and this is where we need to start being clever.



<strong>Commutative Vs Associative</strong>



These terms come from mathematics and usually refer to the relationship and effect of operators on groups of numbers. So, communtative can be described as ab == ba, meaning the order of the numbers doesn’t matter, so multiplication of integers is said to be commutative. Associative can be described as a/b != b/a, meaning that the order of the numbers matter when doing division.



Simple enough, but what does this mean for us in the computer science world? When processing transactions we have to understand weather we are required to process them in an associative manner or a commutative manner. This understanding is borne out of an understanding of the particular problem domain, and like most things there isn’t only a black and white, there is some middle ground. I will try explain with some examples.



The first example is that of a stock exchange. A stock exchange involves holding a number of securities within groupings of financial instruments which are subject to different regulatory constraints and are generally traded differently. Despite all the differences the key issue is that everything in the market can potentially effect everything else in the market because all the values of the shares are based on perception of their value (mostly based on demand, given certain drivers in the market). Perception can easily be influenced between securities, meaning that the security itself influences its value based on its fundamentals, but traders will also evaluate it relative to other securities. There are event some instruments that are based on other instruments.  The point here is that order is vitally important, at a global scale, for all transactions. The order in which bids are received is the order in which they must be processed and all bids and orders can act on a central set of state, being the share’s value. These transactions are therefore said to be associative.



The second example is that of account opening. A core banking system must be able to accept account opening transactions. These contain all the required information to create a new account of some type. Requests will be received from many individuals via the online banking system. One request has no material impact on the other, there is no relation of any kind between them and as a result the order in which they are processed does not matter. These transactions are said to be commutative.



Many people mistakenly believe that certain transactions are associative when they really aren’t. Lets take the example of a bank account. A bank account is simply a “bucket” which knows its balance and limits and maybe some interest concerns. Debits and credits to the account affect is balance, and because of this shared state many people assume that the deposits and withdrawals on that account are associative, when they many not actually be. The key issue with an account is that the balance is known at the end of the financial day so that accounting can be performed. Lets take the example where an account starts with a balance of 100. Two transactions are received simultaneously at the bank for processing, a deposit of 10 and a withdrawal of 30. That means that the balance for some small period of time could either be either 110 or 70, but it would eventually become 80. And we need to understand that “eventually” is also some very small period of time. So in their very nature these transactions are not necessarily associative.



The middle ground example is that currently trading where the values of the currencies have no real bearing on each other for the given trade. A currently trading platform could be receiving many simultaneous trade requests, however only the trades for the dollar must be ordered, and like wise the trades for the pound must be ordered, but they don’t have to be ordered relative to each other. These transactions are said to be locally associative, and provide us with an opportunity for domain based segregation and scaling.



<strong>Locking is evil</strong>



Locking involved protecting a shared piece of data or resource to ensure that multiple threads or nodes can’t effect it simultaneously, because if they do then the behavior is undefined (all those who have ever chased a race condition down 50 different allies will appreciate what that means). Generally multiple write is problematic, but multiple read is fine provided of course that no one is currently writing. So common knowledge is that locks are a good approach to dealing with the concurrency problem of locking and many good frameworks have been created to deal with this, containing many good primitives to help make the implementation easier and more performant.



All that being said, locks are actually evil for a few reasons. Firstly they kill performance. Secondly if a process dies while holding a lock it is typically quite difficult to recover the system without any downtime. Finally, and again related to performance, over and above the locking itself being slow, if the process holding the lock is slow then everything slows down.



For me one of the best illustrations of a core understanding of concurrency concerns is the LMAX team that invented the disruptor. They did some really impressive optimization of concurrency through this understanding and I will go into their architecture a bit more later on. They did some detailed analysis of the performance impact of locking on performance and I would like to quote them to illustrate the point:



“The Disruptor paper talks about an experiment we did.  The test calls a function incrementing a 64-bit counter in a loop 500 million times.  For a single thread with no locking, the test takes 300ms.  If you add a lock (and this is for a single thread, no contention, and no additional complexity other than the lock) the test takes 10,000ms.  That's, like, two orders of magnitude slower.  Even more astounding, if you add a second thread (which logic suggests should take maybe half the time of the single thread with a lock) it takes 224,000ms.  Incrementing a counter 500 million times takes nearly a thousand times longer when you split it over two threads instead of running it on one with no lock. “



<strong>Immutability</strong>



The final, but vital consideration in concurrency is immutability. An immutable object/component is one who’s state can’t be modified after it has been created. This is important because it is therefore guaranteed to be safe to use in a concurrent environment, and this is a staple of functional languages. <strong></strong>



<strong>Summary</strong>



So, in this post I have taken a look at just some of the concepts that one needs to consider when choosing or implementing a core transaction engine for an enterprise. My next post will include some conceptual approaches to implementing such a system and I will also identify some useful open source technologies to use.



<strong>References</strong>

<ul>

	<li><a href="http://www.gotw.ca/publications/concurrency-ddj.htm">http://www.gotw.ca/publications/concurrency-ddj.htm</a></li>

	<li><a href="http://disruptor.googlecode.com/files/Disruptor-1.0.pdf">http://disruptor.googlecode.com/files/Disruptor-1.0.pdf</a></li>

	<li><a href="http://martinfowler.com/articles/lmax.html">http://martinfowler.com/articles/lmax.html</a></li>

	<li><a href="http://mechanitis.blogspot.com/2011/07/dissecting-disruptor-why-its-so-fast.html">http://mechanitis.blogspot.com/2011/07/dissecting-disruptor-why-its-so-fast.html</a></li>

	<li><a href="http://martinfowler.com/bliki/CQRS.html">http://martinfowler.com/bliki/CQRS.html</a></li>

	<li><a href="http://cassandra.apache.org/">http://cassandra.apache.org/</a></li>

	<li><a href="http://doc.akka.io/docs/akka/2.0.3/general/actor-systems.html#actor-systems">http://doc.akka.io/docs/akka/2.0.3/general/actor-systems.html#actor-systems</a></li>

</ul>
