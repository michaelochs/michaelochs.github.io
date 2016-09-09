---
layout: post
title: "Model handling in UIViewController"
date: 2014-06-22 12:56:25 +0200
comments: true
categories: [ UIKit, best practices ]

---

This is the first post in a series of posts about `UIViewController` subclassing best practices I got used to. This series is about how to write a robust view controller that is as dynamic as possible and can be easily adopted to new technices you might use in your app in the future for whatever reason. E.g. you might have handled you view controller instantiation in code in the past but then decided to do it with storyboards; or you might change your underlying connection handling; or you might change from an `NSObject` based model structure to Core Data. These are all cases that will end up in a big refactoring of your view controllers if you don't be careful when implementing them. I did all these things with view controllers that where build with the best practices I am going to describe in this series, and most of the time I didn't even had to change a single line of code in my view controllers.

<!--more-->

In my experience, developers tend to be sloppy when implementing view controllers from time to time. It seems like many people are thinking about view controllers as the passive part that only needs to forward certain events to various layers in your app. The truth is, view controllers are actually one of the most active parts in your app. They need to decide what to do in certain situations and always need to have a proper overview of the state of the application. Therefore it is very important to have a clean structure in your controllers. Once I got used to this structure, I even got faster in implementing my view controllers.

In today's post I want to talk about proper model handling and how to keep your view controllers clean and reusable, even if you decide to refactor one of the more fundamental parts of your app, like the initialization of your view controllers.


## Controller initialization

tldr; Do not write custom init methods.

It is as simple as that! There are lots of reasons why not to do this. The most obvious reason is, you loose the option to initialize your view controllers from a xib or a storyboard. Even though this is nothing you might consider at the moment, this is something that could change completely in the future. Apple is very good in making up new things you end up not knowing how you could have ever lived without them. If there will be a new feature you want to use, and it is implemented in Interface Builder, you have to use `initWithCoder:`. So make sure, your view controllers work with all designated initializers!

Most custom initialization methods I came across are used to hand over the model that should be displayed to the view controller. Sometimes you might want to pass a couple of parameters to the controller. In general, these models or parameters end up in a property at some point. Expose these properties or add a configuration method to your view controllers. Your view controller does not need a model to exist; really! It might need a model from a product point of view, but not from a technical. All the labels, buttons, text fields and other controls you have in your controller can render without the model. A label might be empty, a text field might not change a text inside your model, but your view controller does not crash. Put an assert inside `viewDidLoad` if you are afraid you might end up with a broken view controller, but in general: Handle these kind of guards in your data flow. Do not enforce things technically that are not technically necessary!


## View initialization

tldr; Do not override `loadView`; create an update method for your models and call it in `viewDidLoad`.

You are loosing the option to initialize your views from a xib file. Interface Builder is the best WYSIWYG UI editor I have worked with, so far. And it is getting better every year. I am pretty confident, that you will end up using it one day, even if you don't at the moment.

In my eyes, the only reason to override `loadView` is to have a custom view controller whoes view property is something else than `UIView` or `UITableView`. You could write a `MyScrollViewController` class, whoes main view is a `UIScrollView`. Of course you could also add a scroll view as a subview to the main view in `viewDidLoad`, but if you want to implement your view controller in analogy to `UITableViewController` this is a valid argument to me, but be aware of the ramifications.

The more common case is, that you only want to setup the subviews. Setting your UI state to match your model can be done in `viewDidLoad` without any negative impact vs. doing it in `loadView`. In fact you should split up your view creation code from your view configuration code. Build your view hierarchy in `viewDidLoad` if you are not using interface builder. This will make sure that every time the main view is instanciated, the view hierarchy is set up accordingly.

Add a method `updateViewWithCurrentModel`, put your configuration code in there and call this method from `viewDidLoad` after setting up the view hierarchy. With this approach you have a stateless method that always configures your view according to your current model, regardless of when you call it. We will make use of this benefit in the next step.

Do not setup your view hierarchy in `viewWillAppear:`! I have seen this a couple of times; in the best case you will create new views even though this is not needed. In the worst case you will end up with a view hierarchy where every subview exists multiple times because you did not remove the old views properly.


## Custom setter

tldr; implement a custom setter for your model and call your update method in it.

You can do the next step either by implementing KVO or through a custom setter. I tend to use a custom setter for mainly two reasons. First, there is not very much you can do wrong in a setter since the existence of ARC and second, I don't like the fact that it is not save to call super in `observeValueForKeyPath:ofObject:change:context:` as you need an inside of your superclass that - in my opinion - is non of your bussiness. However if you like KVO better, do it with KVO.

Implement the setter of the property you exposed in the first step yourself (or watch it with KVO). Write your own `setMyModel:` method, update the ivar and call your `updateViewWithCurrentModel` method in there if your view is loaded. If your view is not loaded, only set your ivar. As you are calling the `updateViewWithCurrentModel` in `viewDidLoad` as well, you don't have to worry about this when your view is not loaded. Your setter should look like this:

{% codeblock lang:objc %}
- (void)setMyModel:(MyModel *)myModel
{
	_myModel = myModel;
	if (self.isViewLoaded) {
		[self updateViewWithCurrentModel];
	}
}
{% endcodeblock %}

In case you are in a `UITableViewController` you propably don't need an update method. Simply call `reloadData` on your table view instead, but still make sure, the main view is loaded!

It is important to check whether your view is loaded or not. As the setter can be called by anyone at any time, a call to `self.view` could cause the view to load even though it is not displayed, which is a memory overhead that is completely useless. A view controller should only load its view when told so by its parent view controller. You should always check if a view is loaded in every method that is exposed in your public controller api before calling `self.view` or any method that does this internally.


## Asynchronous loading

With the above view controller design you will get another benefit: If you are loading data asynchronously, you do not need a complex state machine anymore. All you need to do is set your model after the loading is done and your view controller will do the rest. I would recommend loading the data in the previous view controller anyway, but that is a topic I will handle in the next post about `UIViewController` subclassing.


## Resources

I uploaded a basic template for a new view controller as [MyViewController.h/m](https://gist.github.com/michaelochs/a034f7dfddf1e6f3e1fa) to gist. You could easily create an Xcode template from this or generate custom Xcode snippets if you like.