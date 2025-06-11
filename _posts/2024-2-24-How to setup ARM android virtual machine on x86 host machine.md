---
title: "Bypassing Pro Tier Paywall with a Business Logic Flaw"
classes: wide
header:
ribbon: ForestGreen
description: ""
categories:
  - Broken Access Control 
toc: true


---

# First, what are business logic errors?
Business logic errors are flaws in an application’s design that allow attackers to manipulate functionalities by interacting with the system in unintended ways. These vulnerabilities can disrupt workflows, bypass security controls, and ultimately lead to financial and reputational losses for the business.

# The story
I was browsing bugcrowd to look for a program to hunt on until I found a target. The report is not disclosed yet so let's name the target: target.com

target.com is a platform designed to help users create a landing page that houses multiple links. Content creators commonly use it to share various links from their social media bios

our program has 2 tiers free and pro memberships, so the first thing that came to my mind is let's try business logic errors!

Fortunately, the program had a 30-day trial on the pro membership, so I made 2 accounts one with the free tier and another one with the pro tier, and started mapping the application and noting every function in it

Several features were restricted to Pro users, including changing the page appearance and accessing premium themes. I tested these features, but all attempts returned 401 Unauthorized responses.

I spent about two days testing various functions, I thought the application was secure. However, nothing is ever completely secure

I found a function that allows only pro-tier users to invite additional admins to manage their accounts I captured the request in my pro account and it looked like this

```
POST /graphql HTTP/2
Host: graph.target.com
User-Agent: Mozilla/5.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/json
Authorization: "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6Ikh1UmRwOUd6dTY2ZXJpN05SSS1LSSJ9..."
Content-Length: 262
Origin: https://target.com
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-site
Priority: u=0
Te: trailers

{"variables":{"identifier":"invited+gmail@gmail.com","accountId":"95868158"},"query":"mutation ($identifier: String!, $accountId: String!) {\n inviteAdmin(identifier: $identifier, accountId: $accountId) {\n result\n message\n __typename\n }\n}"}
```
Let's break down the request …

The **identifier** value carries the invited admin Gmail

The **accountId** value carries the user ID

- First, i invited an admin using my Pro account. The invited user received a notification.
- Then, I replaced the authorization header with my Free account’s token and changed the accountId to my Free user ID.
- I sent the invite, and it returned 200 OK instead of 401

but I opened the invited account and didn't receive a notification!

I attempted the exploit multiple times, but it didn’t seem to work — until I checked my Gmail. The invitation was sent via email, not through the application itself!

![](/assets/images/site-images/1_ihJGosw4MpnylcXaf6DwVg.jpg)

I accepted the invite, logged into my Free account, and to my surprise — I had successfully added an admin to my Free-tier account!
![](/assets/images/site-images/1_ihJGosw4MpnylcXaf6DwVg.jpg)

Thanks For Reading :)
