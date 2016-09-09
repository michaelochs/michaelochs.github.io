---
layout: post
title: "Making progress on ViewDeck 3.0"
date: 2016-09-08 10:38:00 +0200
comments: true
categories: opensource
published: true

---

I took over ViewDeck as a maintainer from Tom about a year ago. Since then not much has happened, it looks like. In this post I want to talk about what I have done in the last year and where I clearly have underestimated the amount of work maintaining an Open Source project like this takes. I'll then talk a bit about what the next steps will be.

<!--more-->

## Taking Over

Tom was looking for someone to take over the project because he was unable to maintain it anymore as he joined Apple. Even though I never used ViewDeck myself I always thought it was a very nice and helpful project and I did not want to see it die.

Back then I was working at HRS and already pushed forward the open sourcing of some of the components we developed there. I had some spare time and wanted to get more involved in the Open Source community to give something back after years of benefitting from it. So I contacted Tom and asked him if he already had found a maintainer and if not, I'd love to take over – he had not, so I did take over.

Due to the fact that Tom did not had any time to maintain it for the last couple of years, I was confronted with a huge amount of issues that were lacking a response when I first looked into the repository. My first task was to create a couple of labels and start tagging all the things to get a better overview about what were questions from people that had issues, what might be bugs, and feature requests. I then started answering them as good as possible. There were a couple of issues that required me to dig deep into the code and while doing so I already realized that there are many, many lines of code in ViewDeck that are from the ages where child view controllers were not supported by UIKit – an inconvenient state when you are maintaining a view controller whose purpose is to manage child view controllers. ViewDeck contained, and still contains, a ton of legacy code. The other part of ViewDeck is a ton of code that makes this legacy code work on current iOS versions. And while it was a very noble approach from Tom to keep that project working on all the old iOS versions as well, I quickly decided that I don't want to go down that route, but I will come back to this in a minute.

While answering a lot of questions and bug reports, I fixed a bunch of bugs and was working on getting the first version of ViewDeck out that fixed the most critical issues that were reported in the current versions of iOS. This is what version 2.4.0 was about. Another thing I did in this version was, I deprecated a lot of methods in `IIViewDeckController`. In fact I probably deprecated half of its features. My main reasoning behind this was to cut down the complexity of the controller and see what was actually used in the wild. Each deprecation had a warning on it to file an issue if people would still want to use this feature – the results surprised me, but we come to that later. I did two more bug fix releases and at the time I am writing this the latest release is 2.4.2.

## Setting the Course for 3.0

As I said above, I did not want to maintain the library in a way that would still keep it compatible to old iOS versions. It requires a huge amount of work as in my eyes the only way to do this without piling up a huge amount of patches in the code is to always refactor the stuff you are working on to the latest technologies and then make it also work on old versions. If you do not do it this way, you inadvertently end up with code that has the architecture and the features of the minimum iOS Version you are supporting and then a bunch of patches that try to get this code working with the latest iOS versions. This results in a UI that doesn't feel right anymore as it doesn't take advantage of all the new features available, while other apps do.

I don't have the time to do it the right way, the benefit is very little, and I could build a ton of features while instead trying to make a version working almost nobody is using anymore. My goal here is clear: I want to maintain the last two major iOS releases. If stuff works on older versions, fine, but if I need to write a single line of code to get it working on anything that is older than that, it's not going to happen as it would be code that someone would need to clean up again at some point.

## The Current State of ViewDeck 3.0

Because of the amount of outdated code and the huge number of features, I started reimplementing ViewDeck 3.0 from scratch after I tried to remove all the outdated stuff piece by piece. There is simply no point in removing a single piece and make the newly written code work with the other outdated parts, just to then start removing the next outdated part. By now I have an implementation that restores the main functionality of showing and hiding side view controllers via buttons and gesture recognizers. I also have put some effort in making the presentation of side view controllers customizable. As I am not entirely settled on the API yet, my plan is to have most of that customization methods still private in 3.0 and then open this up more and more through out the minor releases of ViewDeck 3.

There are a couple of other, smaller things on my todo list for ViewDeck 3.0 but I plan to release a minimal set of features shortly after iOS 10 launches in order to get a new release out as soon as possible. I want to enable people to move to a new version as soon as possible as the current release still has a couple of issues when it comes to rotation and resizing. So I decided to release with the feature set that probably is enough for 90% of the apps using ViewDeck. The other 10% might need to wait a little longer but I see no point in holding back code that works for most people just to take care of a bunch of rarely used features.

I deprecated a lot of features and currently I don't plan to reintroduce most of them because I did not get a lot of responses after deprecating all these methods. This either means nobody is using them or nobody is updating to the most recent version of ViewDeck. The update behavior may be worse than on other projects as at some point there was a broken release version and the solution was to use an old version. Therefore I think many people have pinned their version of ViewDeck to an old one, not even noticing that there are updates available. In either case it doesn't really matter if the old features are still in the next major version and the less features there are, the easier it is to maintain everything. If, of course, there are feature requests coming in and enough people would love to see the same feature, I am more than happy to add them. I just want to try and keep things small and simple so that the code base is easy to understand.

Let me know what you think about my approach and check out the [v3-reintegration branch][1] on GitHub. I will publish another post about versioning in ViewDeck shortly, so stay tuned.

[1]:	https://github.com/ViewDeck/ViewDeck/tree/feature/v3-reintegration