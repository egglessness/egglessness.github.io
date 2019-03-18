---
layout: post
title: "Abusing Facebook prefetching to leak users IP address and user agent"
description: "Could ever a magical feature that improves the user experience by reducing load time be exploited to get some useful data on a targeted user? C'mon it's 2018, obviously yes!"
date: 2018-12-14
tags: [responsible disclosure, facebook, feature abuse]
comments: true
share: true
---

In 2016 Facebook, aiming to improve final users experience, introduced **link prefetching** as a way to reduce the load time of external webpages. 
As the [related Facebook help page](https://www.facebook.com/business/help/1514372351922333) states:
> Prefetching allows Facebook to download mobile content before someone clicks a link. [...] When someone clicks the link or call to action on your ad, a portion of the web page it's linked to may already have been prefetched and will appear more quickly.

Essentially, when you share on Facebook a rich link (i.e., a link that features title, thumbnail, and description), every time someone comes across to that post, **without even clicking it**, his device loads the page and stores it for a certain amount of time: if the user decides to click on link, the page is already loaded.

Cool, right? User experience can benefit from this, and page load times are slightly faster.  
But, as always, cool features can be used by attackers to achieve completely different results.

Since every time the device prefetches a webpage it fires a `GET` request to the server in which the page is located, I decided to do a simple experiment within my own local network.

I set up a simple server using `Flask` to log full headers of each incoming request; then I created a new post on Facebook, sharing a rich link to my server.

Then, I logged in using a secondary Facebook account on my iPhone and,ì as soon as the news feed was loaded and the shared post became visible, without even touching anything, my server log started to talk very eloquently:

```
Referer: http://m.facebook.com
User-Agent: Mozilla/5.0 (iPhone; CPU iPhone OS 11_4_1 like Mac OS X) AppleWebKit/605.1.15 \
(KHTML, like Gecko) Mobile/15G77 [FBAN/FBIOS;FBDV/iPhone7,1;FBMD/iPhone;FBSN/iOS;FBSV/11.4.1;\
FBSS/3;FBCR/TIM;FBID/phone;FBLC/en_GB;FBOP/5;FBRV/127173516]
Connection: keep-alive
X-Purpose: preview
Host: 192.168.0.24
X-Fb-Http-Engine: Liger
X-Fb-Connection-Type: wifi
Accept-Encoding: gzip, deflate


192.168.0.23 - - [11/Oct/2018 18:41:02] "GET / HTTP/1.1" 200 -
```

We can figure out that this request was fired for prefetching purposes by looking at these specific fields:

```
X-Purpose: preview
X-Fb-Http-Engine: Liger
```

There is plenty of useful information about our target:
- Device model
- iOS version
- Carrier operator (FBCR)
- Device locale
- IP address (so very coarse location information if the target isn't using proxies/VPNs)

Obviously, this works also outside the local network: a simple IP logger service (like [Grabify](https://grabify.link/)) was enough to generate a URL that can stealthily log request headers and redirect the user wherever I want.  
The result? I was able to collect the same data with no effort.

### Affected users
At the time of the discovery, only iOS users having the build `192.0.0.61.85` installed were affected. 

I tried to repeat the same steps using an Android device, but without success (prefetching appears to be disabled at all).

### Unaware users, unaware attackers
Since link prefetching was a feature enabled by default, potentially every person that shared rich links could spy on users behaviour, using a profile, a group or a page as an attack vector.
Moreover, by using this trick and leveraging on privacy controls to shared posts, making them visible to a specific person, it was possible to target an individual strictly.

One can argue: "IP addresses and user agents are not such a big deal! Every time a user browses a website, he leaves the same kind of tracks behind his back".  
The fact is that, in this particular situation, **people didn't know anything about this behaviour**, because:

- the only reference given by Facebook can be found in the help section for businesses
- even if a user read that passage, there aren't any details on the particularly abundant amount of data that is sent just for the sake of previewing.

I just want to stress more on the second point.
An [experiment](https://www.eff.org/it/deeplinks/2010/01/tracking-by-user-agent) carried by the EFF (Electronic Frontier Foundation) in 2010 showed that:
> Browsers usually convey between 5 and 15 bits of identifying information, about 10.5 bits on average. 
> 10 bits of identifying information would allow you to be picked out of a crowd of 2^10, or 1024 people. 

The fun fact is that in this experiment analysed user agents were like:

```
Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US) AppleWebKit/532.0 (KHTML, like Gecko) Chrome/3.0.195.27 Safari/532.0
```

which is nothing compared with the huge amount of data carried by the FB in-app browser:

```
Mozilla/5.0 (iPhone; CPU iPhone OS 11_4_1 like Mac OS X) AppleWebKit/605.1.15 \
(KHTML, like Gecko) Mobile/15G77 [FBAN/FBIOS;FBDV/iPhone7,1;FBMD/iPhone;FBSN/iOS;FBSV/11.4.1;\
FBSS/3;FBCR/TIM;FBID/phone;FBLC/en_GB;FBOP/5;FBRV/127173516]
```

For this reason, even if I didn't run any actual experiment, I believe that the number of bits of entropy on the latter kind of user agent can be far higher, thus leading to a unique identification within an even larger crowd of people.


### Timetable
- 11 October 2018: vulnerability found and reported to Facebook
- 22 October 2018: investigation started by the appropriate product team
- 5 November 2018 (ish): prefetching appears to be disabled, thus closing the vulnerability 
- 14 December 2018: official notification by the Facebook security team, and $1000 bounty awarded
