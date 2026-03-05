---
layout: post
title: Introduction to Web-hacking 4
date: 2026-03-04
author: Wandipa Marema
---

Well! I did promise this web-hacking post a reflection ago and never got to it. Today I'm going to discuss — Server-Side Request Forgery and Cross-site Scripting.

SSRF (Server-Side Request Forgery) — The ability to create HTTP request to any resource of my choice, that the client server responds to. XSS (Cross-site Scripting) — Please don't mistake the abbreviation for the boring style sheet language! This is an injection attack where malicious JavaScript is injected into the web application with the sole intention of being executed by other users. This can give the hacker control of a user's session, or gain access to confidential data.

A SSRF challenge I completed during a content discovery exercise against the given web application. The first endpoint was /private giving me an error message explaining how this content couldn't be viewed from my IP. The second endpoint contains the newer version of the customer account page, allowing one to choose an avatar for their account.

So after creating an account, I visited the new avatar selection feature. I viewed the page source of the avatar form, and the form had a field value containing the path to the image. When the avatar is updated, I'll see the form change and above it would show my newly chosen avatar. Viewing the page source again I would see the avatar is displayed using the data URI scheme with the image content in base64 format.

Now the goal in mind is to change the avatar value to private, hoping that the server accesses the resource getting past the IP address block. How I executed this — Each avatar has a radio button that you can interact with in order to choose the correlated avatar. By inspecting the radio button of my chosen avatar and modifying its value to private and updating the edited avatar — it didn't go as planned as an error message popped up telling me a URL cannot start with "/private". Meaning the website has a deny list in place blocking me from the resource I'm prying on.

Now a deny list accepts all requests except from resources specified or matching a particular pattern. So this deny list has been authorised to restrict any request starting with private, and that's a problem for me so what now!?

Directory Traversal! As the path cannot start with private, utilising path traversal allows me to start from a specified location and move up to the desired location. The trick I used — The value for the radio button, which was private, I did this "x/../private". So when the server sees this "../", it moves up a directory translating to? /private.

Viewing the page source of the edited avatar. It contained contents from the /private directory in base64 encoding. So using a base64 encoding website I was able to capture the flag and complete the challenge.

Now after going through this challenge preventing this vulnerability is possible. The website did have a deny list in-place yet through path traversal the feature was easily bypassed. So possibly improving one's URL sanitation and deny list would be very beneficial.
 
Now, The XSS Challenges. As the main goal of the vulnerability, is to execute on another user's browser. Perfecting the payload is the way to go. The payload is the JavaScript we want to execute on another users browser. So executing a proof of concept is essentially determining if the exploit works, and viewing how the payload is reflected in the target website's code. This will inform me on which payload to use.  

The aim of this challenge for each level was to execute the JavaScript alert function with the string THM. 

Level 1 — I was presented with a form asking me to enter my name, when entered it would be presented on a line below, "Hello, Wandipa".
Viewing the page source, my name was reflected in the code "<h2>Hello, Wandipa</h2>", so instead of entering my name I tried entering a JavaScript payload "<script>alert('THM');</script>". Now when I clicked the enter button, I received a popup with the string "THM" and the page source showed the following: "<h2>Hello, <script>alert('THM');</script></h2>". This is known as a reflected xss. This is when the user-issued data is included within the webpage's source code without any validation. With entering the payload through the form, our exploit is injected into the webpage's source code.

As there are 6 levels to this challenge, I'm just going to talk about the easiest and the hardest Levels.

Level 6 — This level, I had to escape from the value attribute of an input tag, I tried ("><script>alert('THM');</script>"). Where the "> is the important part of the payload, allowing me to escape the input tag by properly closing it in order for the payload to run properly, to no avail. 

The issue was the < and > symbols were getting filtered out from the payload, stopping me from escaping the img tag. To get around this issue I took advantage of what was recently learnt from the passage. The additional attributes of the IMG tag, such as the onload event. Which executes the code of my choosing once the image in the source attribute has fully loaded onto the webpage. Changing my payload to this "/images/cat.jpg" onload="alert('THM');". When I click the enter button, I got the alert with the 'THM' string, and a message informing me that my payload was successful and the needed flag to complete the challenge.

Both SSRF and XSS are exploits that thrive on websites that lack or don't have sufficient URL validation, and input sanitation. To refer to the challenge completed for SSRF — the Deny List was implemented but lacked the necessary knowledge to block other tricks, patterns to bypass the security mechanism. 

Adding these exploits to my arsenal will increase my knowledge in becoming an ethical hacker. A deny list, Input filters. These security mechanisms were built by people who didn't anticipate for someone like me. Now don't get me wrong, this environment was specifically built to be broken, though that doesn't diminish the value of the lesson here — Through path traversal and event handler abuse, their security fell. The security is only as strong as the attacker it's designed against. So to stay triumphant I intend to always be the attacker the system wasn't designed for     
