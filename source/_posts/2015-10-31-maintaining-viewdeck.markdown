---
layout: post
title: "Maintaining ViewDeck"
date: 2015-11-01 13:08:00 +0100
comments: true
categories: opensource
published: true

---

For a couple of weeks now I have taken over [ViewDeck](https://github.com/ViewDeck/ViewDeck) as the maintainer of the library. The original author and maintainer [Tom Adriaenssen](http://inferis.org) aka [@inferis](https://twitter.com/inferis) was able to get a Software Engineering job at Apple and therefore is not able to maintain the library anymore. Congratulations on the new job! Tom has been looking for a new owner [for some time now](http://blog.inferis.org/blog/2015/09/14/wanted-new-maintainer-for-viewdeck/). I want to thank Tom for his work on this great library and I hope I will be able to keep up his good work.

I am planning on changing a couple of things in this library in order to lift this very big project which technically still supports iOS 4 to a current set of technologies. My goal is to provide a future proof option to easily integrate a side bar into every iOS project.

<!--more-->

## tl;dr

If you are using ViewDeck, you might at least want to read 'Transition to 3.x'. But if you don't even want to read that, the message basically is: There will be a 2.4.0, after that there will be a 3.0.0 and unless you open an issue on GitHub, a couple of features will go away in 3.0.0.

## Hamburger menu

The hamburger menu has always been discussed very controversial. I myself am a very strong objector of the hamburger menu. So how comes I now maintain a library that does exactly that? Because I think the problem with the hamburger menu is that it is done wrong most of the time. Often a couple of people are discussing what the most important aspects of an application are and if they can't decide, they simply use a hamburger menu and show all of the features of an app, letting the user figure out what really is important. This is bad user experience, in my opinion, and if you have so many features in your app that are all equally important, you may want to reconsider your decision to put them all in one app. For these kind of menus there is the `UITabBarController` and there is a reason why it can only show a maximum of 5 elements at a time. This is the use case that I would refer to as hamburger menus.

However there are many other reasons where a side menu might make sense. One is Facebook's chat list on the right side. The right side of the screen isn't occupied by any native iOS gestures, so it is a great place to show stuff like a buddy list. This is not a list that shows different features but it is a selection where the user can choose from a list of equal items, in this case their buddies.

The left side of an iOS application is a bit more tricky. In every navigation controller based application the left side of the screen is occupied by the back gesture of the navigation controller. Putting a side menu there leads to a bad user experience. Unless you are an application that is not navigation controller based or you only show the menu in the lowest content controller, where no back gesture is available. A good example for this is Slack. It is an application that technically has a navigation controller but the vast majority of time you spend in the app will be with only one view controller of the navigation stack. So it makes sense to put a list of channels in a left side menu. Metaphorically they are below the main view controller that shows what is going on in a channel, however they are not and should not be the entry point of the application as you are always inside a channel. It makes sense to use a side menu in this case. And again, it shows a list of the same kind of items.

## Semantic versioning

Currently I am working on a stable 2.4.0 version that fixes the most common bugs that are currently in ViewDeck. This probably will not be the last 2.x version but after this I will start working on a 3.0.0 version which, according to my current plan, will remove a couple of features where I don't see a use case anymore.

I will maintain ViewDeck strickly under the Semantic Versioning system. In short this means, within a given major release (e.g. 2.x) you can be sure that you will not see any breaking changes in the API. There might be methods getting deprecated but they are guaranteed to at least function until the next major release.

## Transition to 3.x

With the release of 2.4.0 I will flag a lot of methods as deprecated. This does not necessarily mean that the methods and features will go away in 3.0.0. It is a great way for me to get a feeling of what features are currently in use. If you find one of your favorite features to be deprecated in 2.4.0 please create an [issue on GitHub](https://github.com/ViewDeck/ViewDeck/issues/new) and let me know.

One of the single biggest changes in 3.0.0 will probably be that it removes compatibility to old iOS releases. I am currently planning to make ViewDeck iOS 8+. As the 3.0.0 release will be at least a couple of month in the future, I think by then the adoption of iOS 8 and iOS 9 will be big enough to safely remove iOS 7 compatibility. This enables a lot of other cleanup work in the code. ViewDeck can then finally move to a fully child-controller enabled pattern where it doesn't need to call appearance methods (`viewWillAppear:`, `viewDidAppear:`, ...) manually anymore. Everybody who has written a container view controller in the past knows that this is a big step forward and probably eliminates a lot of code which always means less bugs.

Another thing that I am pretty sure can be removed is the support for top and bottom menues. Not only does this mean less code to maintain but I also think that these menues collide with the iOS notification center and control center, leading to a bad user experience.

There are other features where I want to change the API for them or, in some cases, remove API but not the feature. ViewDeck currently has too many options and it is very hard to get your head around all of them, especially if you are new to the framework and try to integrate this the first time.

## Documentation

The last thing that I find is very important for a new major release is documentation. For the 3.0.0 release my goal is to have a documentation for all public methods of ViewDeck in the header and in [cocoadocs.org](http://cocoadocs.org).

My goal is to make this a required thing for contributing to this project. In order for a pull request to get merged, the pull request needs to have full, extensive documentation of every newly added public method. Also in order to keep a clean API I will not merge pull requests that implement new features that are not requested by at least a couple of users.

I hope you see my thoughts as an improvement to the library and agree with them. If not, please let me know in the comments or open an issue on GitHub; after all this is an open source project and changes should be made in a democratic way.