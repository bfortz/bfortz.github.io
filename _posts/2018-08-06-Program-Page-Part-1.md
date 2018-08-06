---
layout: post
title: "A new conference program page: what do we need?"
tags: EURO conferences OR2018 clojure 
---

*Some thoughts on current conference programs usage and requirements for a better (?) solution.* 

One of the major next steps in the organisation of the [Operations Research 2018](https://www.or2018.be) is to make the program available to participants in a practical and convenient way. As of today, the program is browsable [in the euro-online.org conference system](https://www.euro-online.org/conf/or2018/program) and soon, as a full pdf program with author and session chair indices. Until recently, this (huge) pdf was printed for each participant, or put on a USB stick offered to each participant.

I don't like any of these options: as I mentioned [in my first post]({%
post_url 2018-08-05-2-Many-Facets %}), the program page is currently heavy, and
definitely not mobile friendly. Moreover, it requires to be online. Printing
the program for all participants is definitely a waste of resources and not
environmentally friendly. A USB stick requires each participant to have a
computer to browse the program, and again, it's not very responsible to produce
so many USB sticks (do you really need an additional one?). 

![Do you need an additional USB stick?](/images/usb.jpg)

In addition, these last two solutions are static and will not contain the inevitable last minute changes.

Of course, there is an solution: **a mobile web app**:

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">📲 The *official mobile app* for the <a href="https://twitter.com/hashtag/euro2018valencia?src=hash&amp;ref_src=twsrc%5Etfw">#euro2018valencia</a> conference is already available!<br><br>You can have access to all the schedule and program online, updated info in your pocket! ⬇️ Just scan the QR code attached to this post. <a href="https://t.co/iOx7htOr5u">https://t.co/iOx7htOr5u</a> <a href="https://t.co/z4CnUcMao5">pic.twitter.com/z4CnUcMao5</a></p>&mdash; EURO 2018 Valencia (@euro2018vlc) <a href="https://twitter.com/euro2018vlc/status/1009862172606910464?ref_src=twsrc%5Etfw">June 21, 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

The one used for the last two EURO conferences worked quite well, allows you to make your personalised program and we have already all the tools to extract the information in the right format for the app. And once downloaded, you can use it offline as well.

## The dark side of mobile apps

Well, I would not write this post if I thought it was so simple. First of all, this application relies on third party software that we have to pay for, but most importantly we have no control on it. Some parts are really not best suited for the format of our conferences, and we have no way to change this. It would be better to have something integrated to our conference system, that we own and can improve and reuse freely.

So why not to develop our own app? Here are a few reasons for which I think that, despite their popularity, mobile apps are not a great choice.

A first reason is that the development of a native application would require to
make adaptation to fit the different platforms (Android, iOS) and publish the
app in their application store, according to their rules. 

The web, as imagined by [Tim Berners-Lee](https://en.wikipedia.org/wiki/Tim_Berners-Lee),
was developed to address the problem that [there were all these computing
systems and platforms that existed separately in isolation](https://venturebeat.com/2014/12/10/is-the-web-dead-no-says-tim-berners-lee-but-native-apps-are-boring/). Mobile apps are moving in the exact opposite direction.

Moreover, do we really want to clobber our phones with a new app each time we attend a conference?

## Technological choices and requirements

What would be a good solution then? Here are some elements that will serve as guideline as I move on in the development in the coming weeks:

### Develop once for all platforms 

A web application is definitely the way to go. The interface and functionalities should be similar independently of the fact that users are on a computer, a tablet or a mobile phone, but it should follow [responsive design principles](https://en.wikipedia.org/wiki/Responsive_web_design).

### Fast and responsive

The application should be much more responsive than today. Currently, the program page is dynamically generated each time a user consults it, which represents a high load on the server with many databases access. 

However, even if the program is likely to change, at these points the number of changes should be quite limited. Each network connection introduces a delay. Using cache technologies, the data of the program could be downloaded only once each time there is an update, and processed locally with a [fat client](https://en.wikipedia.org/wiki/Fat_client) technology. Our site is now using [Cloudflare](https://www.cloudflare.com/), making network usage much more efficient. Cloudflare has support for very efficient caching technologies, that makes such technologies very efficient.

### Offline use

The web application still needs to be online. But recently, [Progressive Web Apps](https://en.wikipedia.org/wiki/Progressive_Web_Apps) have changed this. I think this approach is great because it is not tied to a platform, and with a minimal overhead in the development, it leads to a web app that is usable offline. This technology was used for the [ISMP2018](https://ismp2018.sciencesconf.org/) conference program and it seems a really good alternative to a native app.

### Language choice

The current system was mostly developed in PHP, recent features were developed in Python in a [Django](https://www.djangoproject.com/) framework, backed with a MariaDB database. I'm not happy anymore with these two technologies. At some point I might make a longer post explaining why, but the major grief against PHP is that it is so easy to make things (really) wrong and insecure, while Django is a pain each time they update the library (they break the API too often from version to version).

Recently, I learned a lot about functional programming and in particular  [Clojure](https://www.clojure.org). Clojure is a LISP language running on the JVM, but it has also a dialect called Clojurescript that compiles to javascript. The beauty of it is that you can reuse the same code and data structures on the back-end and the front-end.
I experimented developing the new [EURO forum](https://www.euro-online.org/forums/#/) platform with it, and I liked the experience, with a very fast and responsive result.

Even though Clojure is quite a niche language (and probably will stay as such), I think this is a good choice for this project, so let's go...

## Next steps

Now that all this is in place, I can describe a roadmap of these developments:

1. Create a Clojure program to make a static [EDN](https://github.com/edn-format/edn) file 
each time the program changes.

2. Create a one-page app in Clojurescript, with a responsive design, that 
	- downloads the program data;
	- allows to browse it:
		- by time;
		- by stream;
		- by author;
		- by keyword.

3. Ensure things are properly cached with Cloudflare.

4. Adapt the app for offline usage (turning it into a progressive web app).

5. Add static local information for the conference (room maps, ...)

6. Adapt the app to work on past conferences to replace the existing legacy pages, then move on to cleanup the EURO databases.

All this will be documented here, and I plan to share the code as I move on. Keep tuned!

If you have suggestions for improvements or if you think there are bad choices
made here, send me your thoughts by [by email](mailto:bernard.fortz@gmail.com),
on [Twitter](https://www.twitter.com/djkrazyben) or by leaving comments below.
