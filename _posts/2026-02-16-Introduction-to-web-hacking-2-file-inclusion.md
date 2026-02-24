---
layout: post
title: Introduction to Web-Hacking 2 File Inclusion
date: 2026-02-16
author: Wandipa Marema
--- 

This topic deserves its own post within the web-hacking introduction, cause I was completely lost learning this the first time around. 
Removing all the fancy terminology — The essence of file inclusion, is to gain access to confidential content, aka files on a website. Using vulnerabilities commonly found and exploited in various programming languages for web apps, such as PHP. Input Validation is the core issue here, where applications don't thoroughly sanitise or validate user input. When user input is not examined and appropriately altered, the user can pass any input to the function causing the vulnerability. 

The technique branches out into 2 forms. Local File, and Remote File inclusion. The prior stems from a developer's lack of security awareness whilst working on the website. Referring to PHP, using functions such as, 'include', and 'require' to name a few, often assist in creating a vulnerable web app. The latter is to deliberately inject an external URL containing a harmful file/payload, that the target server retrieves via GET request. Executing the code on the device, exploiting the website's lack of sanitising user input. LFI lets you read, whilst RFI lets you execute. Big difference.

After recently brushing up on the forgotten content, it's pretty understandable. Specifically the terminology, the practical side of things... Not so much.  
  
I was given a set of labs. Each lab would practically build up on what you've recently read in a passage. "That's pretty straight forward" I said to myself. Yet I found myself struggling to the point of head pain. I couldn't decipher whether, I didn't understand the question or the content as a whole XD! 

The first lab, I had to read the content of a file inside the specified directory. Now, what I had to understand about file inclusion, which wasn't firmly branded into my brain the first time around. THE TECHNIQUE IS A URL EXPLOIT!!! A Web application's primary role is to request resources from servers. Despite compulsively reading through the content over and over again. I couldn't process that fact, and ended up googling answers. I'm certain ethical hackers google constantly, but the difference in knowing what to google and how to apply what they find is what I'm striving for. 

For some reason I did the entire section again. Moving on, and not understanding a fraction of the content didn't sit right with me. This time I changed my approach. Clearly I couldn't fully grasp what I was reading so why not find another form of content to learn this section from. Youtube! 

Going through the tutorial made the topic practically digestible, which the TryHackMe section made difficult to grasp. I was amused seeing others sharing the same opinion in the comments XD.

I can recall speaking about this on previous posts. When I was learning cryptography and networking, reading the content was the ideal. Now for practical sections, relying on just text isn't beneficial for my learning to understand. I had to use a different approach. 

As previously stated File Inclusion is a URL exploit, so in order for me to read the contents in the specified directory, I was gonna have to alter the URL. Through changing the parameter within the query string, specifying the exact file and the path to traverse through. For example, the normal URL was "http://blahblahblah/lab1.php?file=welcome.php". Now since the app doesn't validate, I can change it to,  "http://blahblahblah/lab1.php?file=../../../../etc/passwd".

 Now for the specific question I was working on. It didn't need me to traverse through directories with the use of '../'. I'm just demonstrating how I could traverse through the directories to reach the desired destination, whilst simultaneously searching for the target file.

This post is to show the power of having multiple learning methods, not saying that more is better, but having access to other learning resources, saves time and countless nonsensical effort.
  
Now modern day websites will obviously have better input validation, and URL sanitising mechanisms in place. So trying techniques like these probably won't go in my favour. In an abstract sense, knowing the primitive of such a technique will open doors in learning much more advanced methods.

I'm excited to tackle the next topic — SSRF, whatever that means.
