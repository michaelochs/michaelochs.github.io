---
layout: post
title: "Custom error handling on iOS"
date: 2014-10-03 12:56:45 +0200
comments: true
categories: opensource
published: true

---

In the last weeks I reimplemented the error handling of the HRS app in a way that is painless and very fast for the developer to use, and gives the user a good, detailed and specific error while even allowing him to recover from certain error cases. The way I did this was basically by reimplementing a concept that is available on OS X for a long time now.

<!--more-->

## tl;dr

I have created a library that is open sourced and maintained by my company and me. It implements the workflow described below. You can use it under the Apache 2.0 License and it is available on [github as HRSCustomErrorHandling](https://github.com/Hotel-Reservation-Service/HRSCustomErrorHandling) and via [cocoapods](http://cocoapods.org/?q=HRSCustomErrorHandling). If you do so, I would be happy if you'd let me know.


## Common error handling

Error handling on iOS has always been very painful. In general you trigger an action on the users behalf in one of your view controllers and you might get an `NSError` in return. There are two common approaches to deal with this error:

0. Display a generic error message to the user that says 'Something went wrong. Please try again later.'
0. Have a huge block of code that checks the error domain and the error code every time an error can be returned and provide a custom error message to the most common error cases.

The latter is definitely the better one as it gives the user a better feedback than 'something went wrong'. However, it is a much bigger pain for the developer, so it is also the option that is rarely used.


## What is already out there?

The approach I took was to first look up existing concepts. There is an error handling that is already available in iOS's big brother, OS X, that reduces the code to display a custom error message to the user to a single line and also gives the user an option to recover from certain errors.

There already is a sample implementation of this error handling from the people over at [RealMac software](http://realmacsoftware.com/blog/cocoa-error-handling-and-recovery). However a couple of features are missing in their sample that I find very useful in the OS X version.

We wanted to have a way to handle specific errors in a custom and asynchronous way but at a central place in the app. Imagine a server connection that returns an error to you and tells you, that the user is not logged in to the server. You don't want to simply tell the user that this is the case and you also don't want to handle this with a specific dialog in every single part of your app that might be able to get this error. Instead there should be a single place in the application that checks for this error and displays your login dialog.

The error handling I am going to describe in this blog post can deal with this and a couple of other scenarios.


## Responsibilities

Let's first check which component of the app is responsible for what. Again, OS X is a good guideline for this.

### Creation

Let's imagine there is a method that creates an `NSError` instance and passes it to the caller. A method like `- (BOOL)save:(NSError **)error`. This method usually is somewhere in your data abstraction layer or in other frameworks. If this method is in your code, this is the place where you have as much context information available as possible. You know exactly why the error happened and what went wrong. You might even know what to do to make the error go away. This is the part of code that should put all these informations into the `NSError` when creating it.

### Handling

The caller of the `save:` method mentioned above is your view controller in most of the cases. It is the one that knows what lead to the save action and - if there are any parameters to the method it called - it also knows what parameters have been used and what to do if this action should be executed a second time. This layer does not need to have any information about what went wrong. The only information it is interested in is 'should I try this again or not?'.

There are a couple of rare cases where your view controller might also want to modify or skip error presentation. For example a login view controller might not want to display the general 'your credentials are invalid' error, instead it might want to show an inline error message close to the login screen's input text fields to show the user that something went wrong. So there needs to be a way to customize error handling in view controllers.

### Displaying

Displaying an error is a task that should be handled in general in an applcation. You do not want every error dialog to look different. You might want to show all your errors in a simple alert view or in a custom overlay that integrates nicely into your app's ui, but they should all look the same.


## NSError

How to create an `NSError` that has all the information mentioned above? The good thing is, `NSError` already has an interface to provide all this, and it is right in front of you: Have a look at the keys you can put into the `userInfo` dictionary of an `NSError`. There are a couple that are quite common when creating custom errors. The one that is used the most is probably `NSLocalizedDescriptionKey`. We are not going to use that - bummer.

Instead, we are going to use almost all of the others:

- `NSLocalizedFailureReasonErrorKey`  
This describes what went wrong. This is going to be presented in the title of an alert view, later on.

- `NSLocalizedRecoverySuggestionErrorKey`  
This describes what the user can do to fix the error. For example you can tell the user something like _'Try turning it off and on again'_, but also some more useful information like _'Do you want to overwrite the existing file with your new one? This will delete the existing file permanently.'_

- `NSLocalizedRecoveryOptionsErrorKey`  
This key should contain an `NSArray` with localized strings that describe the actions the user can take. Each element is used later on as the text of a button. You should make sure that this key contains at least one cancel operation that simply does nothing except dismissing the alert. A possible array could look like this: `@[ @"Overwrite", @"Cancel" ]`  
One important fact about this array is, that the first option in the array is the default option. This means that if you are going to present the error in an alert view, it is the option on the right most button that is displayed in bold text. In this case that would be the _'Overwrite'_ option, but be carefull when making a destructive operation the default one!

- `NSRecoveryAttempterErrorKey`  
Now this is the most interesting key. It contains an object that conforms to `NSErrorRecoveryAttempting` protocol. This protocol basically contains a method that can be called with the index of the option the user has choosen. The recovery attempter should then try to fullfil the action behind this option and report back whether it was able to recover from the error or not. If the recovery attempter was successful in recoverying from the error, it is safe for the presenter to retry the action that lead to the error.

With the above keys we are able to carry all information that is necessary to handle an error case properly inside the `NSError` object that describes the error. Besides the information shown to the user we can also add logic to the error that can try to recover from an error and communicate back whether we can safely retry the action or not.


## Responder chain

OS X is using the responder chain for handling and displaying the error. The responder chain is something that is rarely used in iOS by applications, yet it is used by the frameworks all the time. It is responsible for delivering actions like text input or motion gestures.

It is very useful for transporting information up to the application from every `UIResponder` object through out the app. The good think is: A lot of the objects you are dealing with on a daily basis are in fact inheriting from `UIResponder`. The one that is well known for being a responder is probably `UIView`. But all your `UIViewController`s are responder objects, as well. Even your `UIApplication` is a responder object, and in the case you are making a game, `SKNode` is one, too. This basically means that all your code that deals with presentation is a responder or has very easy access to a responder.

Another great feature about responders is, that they build a chain. Each responder object implements a method called `nextResponder` that returns the next responder in the chain. The chain is build from the deepest point in your application up to the application itself and also, since a couple of iOS releases, even up to your app delegate, as long as it inherits from `UIResponder`. This means you can dispatch a message to `nextResponder` recursively and it will end up in your app delegate eventually, which is a very great place to handle application wide behaviour such as presenting an error. And because of this, that is exactly what we are going to do.


### Handling

OS X implements a method on `NSResponder` that is called `-presentError:` which returns a bool telling you whether error recovery was successful or not. Due to the fact that showing modal dialogs on OS X are a blocking operation, this method is blocking, too. Obviously, if you would do this in an iOS application your UI would not be responding anymore and your app would be killed by the watch dog very fast. Luckily we do have blocks now, so this method could easily be changed to something like `-presentError:completionHandler:`.

The completion handler should then tell you whether error recovery was successful or not. If it was, you can simply call the method that lead to the error again. As blocks hold on to their context, this can be done very easily in most scenarios.

You need to implement the present error method yourself, as it is not available on iOS, but it is very simple. You could either use the github project I created or do this yourself. All the method does, is calling `-presentError:completionHandler:` on the object returned by `-nextResponder`. Well, not all. There are a couple of exceptions. Before it does forward the message up the responder chain it calls `willPresentError:` on itself.


#### `-willPresentError:`

This method gets the error passed in and returns an error. The base implementation of this method simply returns the error it was passed. You can now override this method to change the error you return or even return `nil`. If you return `nil`, the error is not forwarded up the responder chain by `presentError:completionHandler:`.

You can use this to solve the problem mentioned in the introduction. Your login view controller simply overrides `willPresentError:`, checks if the error passed in is a login error, and returns `nil` in this case. With this hook, every responder in the chain is capable of modifying the error or even stopping the error presentation.


#### `UIViewController`

Another exception in `-presentError:completionHandler:` is that it forwards the message to `-[UIApplication sharedApplication]` if calling `nextResponder` returns `nil`. This is because a `UIViewController` that is currently not participating in the view controller hierarchy will return `nil` when asked for its `nextResponder`.


#### App Delegate

There is one thing I simply dropped up to now: The errors you get from a framework you are using do not contain the informations you need for the custom error presentation. To upgrade these errors to enriched ones, you can once again override `-willPresentError:`, but this time do it in your app delegate. Now you have a place where you can add nice error messages to your most common framework errors. You should also add a fallback error message that probably simply tells the user that _'something went wrong'_. That is not a very good error message, but it is for sure better than showing the user a _'The operation couldn’t be completed. (Cocoa error 1570.)'_.


## Presenting

Now that we ensured that your error will get passed to your app delegate, all you have to do is to override `-presentError:completionHandler:` in your app delegate and configure a `UIAlertView` or any other custom view that can show the error to the user. In this single place you can easily customize your error UI to fit perfectly into your app design. No more ugly default alert views in your nicely crafted user interface!


## HRSCustomErrorHandling

I have open sourced the little library that is driving the error handling in the HRS iOS app I am working on during the day. It has a couple of nice little features that make your error creation even easier. For example you can simply create a recovery attempter with recovery blocks that are executed once the user taps on an option in the alert view. It is available under the Apache 2.0 License on [github as HRSCustomErrorHandling](https://github.com/Hotel-Reservation-Service/HRSCustomErrorHandling) and via [cocoapods](http://cocoapods.org/?q=HRSCustomErrorHandling). If you use it, I would be happy if you'd let me know.

I also gave a talk about custom error handling. The slides are available in the [talks](/talks/) section.
