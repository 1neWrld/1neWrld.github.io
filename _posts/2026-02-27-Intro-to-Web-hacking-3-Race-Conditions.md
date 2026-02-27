---
layout: post
title: Introduction to Web-hacking 3 - Race Conditions
date: 2026-02-27
author: Wandipa Marema
---

Cheers to another vulnerability getting its very own long, yet fruitful blog post!

Quick Storytime! — A while back. 2022 I believe, my mates and I wanted to go watch the latest Batman movie, called The Batman. Directed by Matt Reeves, starring Robert Pattinson (Twilight guy) as Batman. Now our plan, obviously, was to all sit together as besties. To ensure we had the best experience possible, we wanted the most optimal seats. Not too close, leading to neck pain from tirelessly looking up to the screen, or too far back from it because... I actually don't have a reason for that one, we just didn't want to be at the back. Lastly sitting in the corner was beneficial, to avoid people walking across you when watching.

So I went on Numetro's website to make a reservation for the seat I wanted, the usual. It was there, it was available, I booked the seat and done. Though now the night came, we entered the cinema — someone was in my seat. We get into a verbal tussle, which led to the authorities getting involved. The word authorities is used pretty loosely here, as I'm referring to the cinema attendant XD. We show her our tickets and lo and behold — one of us didn't solely own the seat, cause both our tickets stated we reserved the seat. Now as two grown men, how were we going to decide who sits on whose lap XD! Though despite that interaction, this story is the core idea of a race condition.

A Race condition occurs when multiple requests access and modify data concurrently, and the outcome is based on the unpredictable timing of execution.

Programs, Processes, Threads, and Multi-threading are terms and procedures pivotal to The Race (Race conditions). A Program is a recipe of instructions to achieve a specific task. In terms of game dev. If I wanted to give my in-game character the ability to run, jump, aim, and shoot. I would create a script (the recipe) for this character, which contains all the necessary instructions to give my character the needed fundamentals for my game. That is the program!

When I run my game to test whether the script works, a process is created. A process is a program in execution. If the character script is the program, the process is executing that script.

Threads — A lightweight unit of execution in a process, sharing memory space and resources. This part is important for when I get into the challenge I did. As in many cases, one would need to replicate the same process repeatedly. Think of a battle royale, having to constantly connect players within a server into a game. I could adopt 2 approaches — Serial Processing, requests are dealt with one at a time. User A then User B, one thread at a time. Parallel processes, multiple requests are handled simultaneously using their own threads. Parallel processing is a part of what our final term is — Multi-threading! As parallel request handling is often implemented using multi-threading, which allows multiple execution paths to share the same application state.

Now with this newfound knowledge collected like infinity stones, how do I exploit it!? Well the challenge I completed explains just that — After going through a demo task, which required me to transfer $100 from just $5 on a web application dealing with phone credit transfer. I had to explore and study how the application receives HTTP requests and responds. How the hell am I going to do that!?

Burp Suite — A software used to intercept traffic, allowing us to analyse, and modify between a web browser and target server, to uncover vulnerabilities. Now with using Burp Suite, and utilising its bundled browser feature, allowing us to browse the target site and study how it processes our requests.

I logged into one of the two given accounts and made a transfer to the other. Within the Proxy tab in Burp Suite (which is the tab used to view intercepted traffic), This will help me understand how the system reacts to valid and invalid requests. I saw a 'POST' request, showing the target's details and the amount sent. Within the response I could see that the transfer was successful, and now that I've seen the internals of the application — Let the race commence!

Sending the single POST request to the repeater tab (A manual tool used to modify, resend, and analyze individual HTTP requests multiple times), I can exploit the application by sending 20 duplicated requests. Keep in mind, the challenge requires us to send $100+ to the other user with only $5. The balance check and deduction were not atomic, so each duplicated request passed validation before the account balance was updated. So duplicating the request, and unleashing all those minions at once to the server, will be a GTA money glitch XD! Due to the server processing requests concurrently, multiple requests were validated before the first request completed and updated the balance. Creating a race window for the account to be overdrawn.

So going back to the cinema fiasco. Just like the challenge application, the NuMetro website had an analogous situation. Two people are processed at the same time, which is fast and efficient. Though the lack of synchronisation of shared data initiated a race condition. Both booking requests were processed concurrently, and the server checked seat availability before either request completed the reservation, leading to our old town road showdown!

Now despite the immense knowledge dump, I want to discuss the immense trauma dump. The previous challenge was a guided demo. Which in itself was pretty easy to practically do, yet challenging to grasp theoretically. The next task had us do the exact same thing but without any guidance. Transfer $1000+ to another user. 

That's pretty simple! We're just more greedy the second time around XD. Intercept the traffic, duplicate it, and send the bait on its way to the server. No harm no foul. Yet it... Didn't go as planned.

The account I was using had $99 in it. 99 * 11 duplicates would give me just above $1000. So tell me why when I sent the requests, only $300 shows up in the account? Luckily the frustration wasn't immense yet. I used the account I just sent the $300 to. I sent the money back with duplicates of its own. I only duplicated the request 4x, cause that would get me $200 over the required amount. Though the issue still prevails, as $400 (Forgot the exact amount but between 500-400) was only in the account. 

A very significant quote for this moment — "Insanity is doing the same thing over and over again and expecting different results" I was just doing the same thing over and over again. Switching accounts, duplicating, sending — Nothing! The application crashed most of the time. Due to an overload... An Overload... That's it! 

I wish I could say my 'aha' moment stemmed solely from my head, but it also took a ton of third party thoughts from the internet. It was reassuring reading how people were going through the same issue as me. The overload was the issue to all the problems. At the start of the Introduction to Web-hacking path, it discusses the importance of analysing how the application responds and reacts to certain requests. The plethora of requests trying to access and modify the data concurrently caused the application to overload and crash.

So one needed to figure out a way to send the requests bit by bit to avoid that issue from occurring. So at first I planned to send each request one by one, which sounded a little tedious, but also. Wouldn't that completely disregard the entire point of race conditioning? Sending each request one by one would give the system time to efficiently validate the transactions, making it impossible to actually exploit the system. 

So the first set of requests I sent a small amount back to the server, then sent the rest in one big set. Hopefully that would prevent the system from overloading — It did, and it worked! 

Whilst I know I had a bit of assistance from other people struggling with the same challenge on the internet. That doesn't invalidate the importance of thoroughly analysing the target application. The overload error message gave me a clue on how the plethora of requests working concurrently, affected the application's performance. So I had to find a way to exploit the system without causing it to fail. That error was pivotal in finding a solution. Quite ironic how a message utilised to communicate with the user, can be used to assist exploiting your very own system.

In regards to the web application architecture. The real culprit was the application performing a check-then-act operation without wrapping it in an atomic transaction. An atomic operation is one that executes completely as a single irreducible unit, without any interference from other concurrent operations.   

Race conditions aren't just about sending a bunch of requests — they're about understanding the system behaviour under load. The overload message wasn't just for communicating failures. For an attacker like me, it's information! It told me the system couldn't handle 20+ concurrent requests so I adjusted. This is about observing, adapting, and unfortunately, iterating until a solution is found.

Phew! This is my longest post yet!
