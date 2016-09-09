---
layout: post
title: "Subclassing UIViewController"
date: 2014-06-21 14:44:32 +0200
comments: true
categories: 
published: false

---

Subclassing `UIViewController` is pretty simple, there is even a template in Xcode that helps you with the right hooks, and yet doing it right can be pretty challenging. I see lots of implementations that have problems with connections that are handled by the controller, increased memory footprint, layouting issues or even empty views after iOS decided to remove the view controller's view from memory.

---> robust! even though you have full control!

# Basics
First things first. Lets get a quick overview about views and view controllers.

## Hierarchies

In general there are 2 different hierarchies: The controller hierarchy and the view hierarchy. These two hierarchies have a lot in common, which sometimes lead to confusion. It is important to keep them separated from each others. The controller hierarchy will always be set up before the view hierarchy and the view hierarchy will always be deconstructed before the controller hierarchy, they are nested. However, this does not mean, that the controller hierarchy will always be deconstructed when the view hierarchy is deconstructed. Take a look at `UINavigationController`: If you push and pop a view controller multiple times, the pushing view controller's view is removed from the view hierarchy with every push of the new view controller and is readded to the view hierarchy with every pop event. The view controller hierarchy for this view controller stays intact; until you pop this view controller as well, calling `parentViewController` on this view controller will always return the instance of the navigation controller.

# Best practices
In the following chapter, I am going to list a number of common pitfalls. If you read about those pitfalls and think that the resulting problem is obvious for you, then good! My experience shows, that to many people those pitfalls don't seem to be that obvious and I hope the list below will clarify a couple of mistakes.

## Data flow
Because of the two different hierarchies described above, it often is not a good idea to call any methods that control data flow in view related methods.

### Connection handling
Many people seem to think that starting a connection in `viewWillAppear:` or `viewDidAppear:` is a good way to not load data you might not end up displaying. The idea in general might be a good one, however you have to be careful make sure that you start your connections only on the first call to this method. Remember that you most likely will get many calls to these appearance methods. Even if you are in a view controller that does not push or present any other view controllers, nobody guarantees you, that the pushing view controller does not keep you instance around and might push it again at a later time.

## View related methods

In general, there should be no model related calls in any of the view related messages:

- `loadView`
- `viewDidLoad`
- `viewWillAppear:`
- `viewDidAppear:`
- `viewWillDisappear:`
- `viewDidDisappear:`
- `viewWillLayoutSubviews`
- `viewDidLayoutSubviews`
- other methods with view related names

These methods belong to events that occure when certain things in the view hierarchy happen. There should be no calls to update a model or start a connection within these methods, as these methods should be state independent from your view controller and might get called several times, or not at all.

## Controller related methods

The same is true for the other methods. You should not add calls to view related behavior in methods that are controller related:

- `initWithNibName:bundle:`
- `initWithCoder:`
- `willMoveToParentViewController:`
- `didMoveToParentViewController:`
- other methods with controller related names

Calling a view related method in one of these methods might end up in some pretty bad and hard to trace bugs. In general, most of the time you will call `view` somewhere which eventually will trigger a `loadView` and will mess with the nested call hierarchy as you will trigger a `viewDidLoad` before the controller hierarchy is fully build up.

# Best practices for certain events

## viewDidLoad

Build your implementation so that this method can be called as often as the system thinks this is necessary. A call to `viewDidLoad` should therefore be able to configure the view with an already existing model as well as configure the view with a loading interface in the case that a connection or any other asynchronous task is still pending.

## returning connections

In any asynchronous model callbacks, like a connection that will return with an updated version of your model, check for the view beeing loaded before manipulating the view. Use `isViewLoaded` to do so. This, together with the best practice for `viewDidLoad` gives you a very robust setup of your view controller that is independent from any view or model state.

## setting a new model

The best practices for `viewDidLoad` and returning connections give you another advantage: As you already have the code that configures your view to a certain model, regardless of the point in time this event happens, it is very easy to use the same code in your model setters. This means that you no longer need methods like `initWithModel:` but simply configure your view (in the case that `isViewLoaded` returns `YES`) to show the new model. And if `isViewLoaded` is `NO`, simply set your model's ivar. The code you already have in `viewDidLoad` will handle your new model, once a view is requested from your controller.

## NSCoder

With all the above setup, you get another benefit: It doesn't matter anymore if you instantiate a view controller through code or via a storyboard. A storyboard will call `initWithCoder:` no matter what you would like it to call. Therefore a `initWithModel:` method wouldn't work, anyway. You have to set your model later on.

All you have to do is, trigger the same connections you trigger in `initWithNibName:bundle:` in `initWithCoder:` and you are good to go.

## State restoration

I haven't tried that, but in theory, state restoration should call `initWithCoder:` as well, so you will get this for free with the above code modifications!