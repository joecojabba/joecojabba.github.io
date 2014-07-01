---
layout: post
title: An improved guide to the multicultural festival
tags: []
status: publish
type: post
published: true
meta:
  _publicize_pending: '1'
  _oembed_d4e2aa4760e30e1d88a889aa22d1d5b9: "{{unknown}}"
  _oembed_1eab74f54d629a37066aa44c0fa71dd1: "{{unknown}}"
author: 
---
The [national multicultural festival](http://www.multiculturalfestival.com.au/) is fantastic; It's probably the best thing Canberra does.

## tl;dr

1. I <3 festival especially the food :)
2. Played around with a new build / tech stack
  * [jQuery 2.0 beta](http://blog.jquery.com/2013/01/15/jquery-1-9-final-jquery-2-0-beta-migrate-final-released/)
  * [jQuery Mobile 1.3rc1](http://jquerymobile.com/) with [backbone.js](http://backbonejs.org/)
  * [yeoman.io](yeoman.io)
  * REST implementation on [ASP.NET Web API](http://www.asp.net/web-api/overview/getting-started-with-aspnet-web-api)

3. HTML5: Given the network congestion, perhaps native would be a better choice?

## Anyway, some boring background...

I'm a big fan of the food spectacular event that they have where they bring together hundreds of stalls representing various cultures and community groups and everyone puts on a massive feast.

The official guide however, doesn't&nbsp;really help out all that much.

I mainly wanted to know who would sell me something tasty to eat with a refreshing beverage to wash it down with and where among the action were they!

With the official guide there&nbsp;doesn't seem to be much rhyme or reason to stall locations and its extremely difficult to figure out where a vendor may be.

So I put together&nbsp;[http://cultfoodfest.net](http://cultfoodfest.net) (probably a bit too late for others from this blog post perspective :) ) &nbsp; to help me out and further .

I used the information available from the official site&nbsp;to gather event &nbsp;and stallholder details, and the general layout of the area.

I also built up some latitude,&nbsp;longitude&nbsp;estimates for the stall locations using google maps. I later hit the streets to confirm the co ordinates as best i could as the tents were going up and on the main day itself.

## Tech

### jQuery 2.0b

I went with the beta of jQuery 2.0 to see if there would be any immediate gotchas for even this tiny app (especially working with 3rd party libs)

jQuery 2.0 is supposed to be faster and smaller than 1.9 and a&nbsp;[jsPerf ](http://jsperf.com/jquery-1-7-2-vs-jquery-1-8/13)indicates that it is indeed mostly faster. Minified however there is only a few kb difference. I was hoping for a larger reduction, but it is a beta with more work to come.

One of the best things to come out of 1.9 &amp;amp; 2.0 imo is actually the jQuery-migrate plugin.&nbsp;The plugin restores deprecated features and behaviors so that older code will still run properly on jQuery 1.9 (no jQuery 2.0 tho it seems from my testing).

jQuery-migrate would log to console any usage of deprecated features just served as a &nbsp;healthy reminder to clean up my own coding practices.

### jQuery Mobile 1.3rc1

I really wanted to test our 1.3 for use of their new panels widget, but i just didnt get time to get the functionality that I was thinking of in before the festival.

1.3rc1 also isn't compatible with jQuery 2.0 since it had some uses of deprecated functionality. From the 1.3 source attrFn seemed only used for old IE so for my app, I just removed it. There were also a couple of usages of attr, but they were simple enough to change to use .prop instead. (Mental note, submit patch)

I also found I didnt get the greatest, responsive performance compared to other versions I had used elsewhere. It wasnt bad, it just could have been better imo.

It could have been a few things or combinations there of.

*   Perhaps it was because its 1.3rc1 and not final
*   maybe my above "hacks"
*   maybe the integration with backbone and "play nice" with routing that is needed to be done
*   In the other projects I messed with the click delay
*   maybe the default theme threw me off.

Eitherway, using the chrome dev tools didnt reveal any delays from the stuff that I had written and seemed to be buried among calls in jquery and JQM.

### Backbonejs

I just love the simple, understandable structure that it provides. Given the deployed functionality I probably could gotten away without it + the added dependency/download on underscore/lodash given I already had jQuery as well for a mobile site.

But whats 10kb between friends? Besides from little things, big things grow and I had plans for much more.

### yeoman.io

&gt; "Yeoman is a robust and opinionated client-side stack, comprising tools and frameworks that can help developers quickly build beautiful web applications"

The .NET space leaves a lot to be desired in the form of tooling targeting front end development. I'd be really interested to know what others (assuming anyone reads this :) ) have found works for them. From my experience its been a mix match of different tools and Visual studio plugins which, while helpful, has caused me pain for my CI.

I found yeoman removed a lot of that friction.

The "watch" auto builds, scaffolding, linting, minifying, testing support, image optimizations and more... a nice all in one workflow built on established tools and practices.

I'd definitely recommend checking it out.

## Final thoughts

For real world usage, I ran into a problem I hadn't immediately thought about. Network congestion.

With tens of thousands of people in such a small area it was difficult to even get a connection on the phone. Calling friends was impossible let alone loading google maps.

The site loaded and cached fine making it somewhat usable offline, even the REST calls to update and refresh data were ok. But it was that "where is the german beer tent" type of functionality that relied on google maps that failed. The HTML5 geo location didnt really help either. Sometimes it was spot on, but mostly it but me a block away from where i was.

Maybe native would have been best for this type of application. It probably would have alleviated most of the network congestion issues using the internal map controls. The GPS seemed to be more consistent as well.

Things to do for next year I guess. Phonegap or complete native and getting it in the app store would have been impossible given my 24 hour dev cycle this year anyway.