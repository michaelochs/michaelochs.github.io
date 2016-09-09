---
layout: post
title: "Connection handling in UIViewController"
date: 2014-10-11 13:24:12 +0200
comments: true
categories: [ UIKit, best practices ]

---

This is the second post in my `UIViewController` series. This time I want to explain what I think is the best practice to do connection handling in your custom view controller instances.

There are a couple of ways to implement a connection handling in your application and different reasons for an application to load data from the web. If you are for example developing an application like a todo application, you probably do not want to load data in the view controllers at all. A better way to do this would be to have a synchronization manager that does all the loading and the view controllers are only showing the locally cached data. If you are developing one of these apps, this post is not helping you at all. Connection handling is not part of your view controller, but updating your view according to changes in your model is, so you might want to have a look at my previous post about [model handling in UIViewController](/blog/2014/06/22/model-handling-in-uiviewcontroller/). This post is only about the how and when to load data in your view controllers and how to respond to error events.

<!--more-->

## Use AFNetworking

There is not much to say about it. Use `AFNetworking` when making connections! You have a very comfortable block based API when dealing with your requests and you do not need to deal with taming `NSURLConnection` in a background thread yourself. The framework is constantly improved and the development community is very active in this open source project. It is maintained by [Mattt Thompson](http://mattt.me), who is well known for [a lot of very great projects](https://github.com/mattt).


## Use the presenting view controller

So we are going to load data in our view controllers. My first point is something that you might not be able to do, but still: Do not do it! Well, that is not entirely true, you just should not load the data in the view controller that is also displaying it. In my experience it is a lot easier if the previous view controller starts the connection and presents the displaying view controller only if the connection was successful. In my opinion this also leads to a much better user experience with less effort.

One thing people tend to forget is: Your connection might fail. It is even very likely that your connection will fail. Do not make the mistake to think that everybody around the world has a LTE or 4G connection with speeds around 100MBit/s. Mobile networks are still very bad in a lot of places around the world! Make sure to design your app and your connection handling around a slow EDGE connection. This are the circumstances people will use your app the most in. If you have an app that has something to do with travel, chances are high that a large number of your customers don't have access to the internet at all, when they are using your app, as most people still have roaming disabled.

The point I am trying to make here is that people will see your error dialog way more often than you think. Design it wisely and make sure it interupts the user's workflow as less as possible. Empty pages with a text that tells the user that he or she needs an internet connection might look nice, but it forces the user to read the text to realize what went wrong and leave the screen again before he can continue to use your app. A popover telling the user that the request failed with a 'cancel' and a 'retry' button lets the user quickly decide how to preceed and if he taps 'cancel' he is right back where he was.

This is one of the reasons why I think the workflow should be as follows: The user taps on whatever makes the next view controller appear. The current view controller triggers the connection and shows a loading overlay. If an error occurs, show it to the user, either in an alert view or in a custom designed popover. You can do this very easily with the techniques described in my post about [custom error handling on iOS](/blog/2014/10/03/custom-error-handling-on-ios/). Make sure to give the user an option to retry, if it was an error from the `NSURLErrorDomain` error domain. If no error occurs, instantiate the target view controller, set the model on it and display the controller.

To make sure you do not need to setup the same response in a variety of view controllers that all have a transition to the same view controller, you can still let the target view controller create the connection that is needed. With blocks you can create a very easy to use pattern that can be implemented by all of your view controllers:

{% codeblock lang:objc %}
+ (NSURLConnection *)loadData:(void(^)(id data, NSError *error))completionHandler;
- (void)setData:(id)data;
{% endcodeblock %}

Make sure to implement the first method as a class method. This way you can load the data before you instantiate the view controller. If you use a framework like `AFNetworking` the implementation of this method is as simple as creating the request and enqueuing a new operation with it. You already have a success and a failure completion handler that you can use to call the completion handler from in there. If you like the approach of having two different completion handler, one for the success case and one for the failure case, you can easily adjust the method above to represent that as well. If you are using `AFNetworking` you should also change the return value of the method to `AFHTTPRequestOperation`. The return value of the operation is to be able to cancel the operation if you have to, e.g. when the view controller that started the connection is removed from the navigation stack.

The `setData:` method takes the `data` from the completion handler and configures the view controller.

So to display a view controller with prefetched data, all you need to do is:

{% codeblock lang:objc %}
- (IBAction)showDetails:(id)sender {
	[ViewController loadData:^(id data, NSError *error) {
		if (error) {
			// handle the error (e.g. present it!)
		} else {
			ViewController *viewController = [ViewController new];
			viewController.data = data;
			[self.navigationController pushViewController:viewController animated:YES];
		}
	}];
}
{% endcodeblock %}

If you use the [custom error handling](/blog/2014/10/03/custom-error-handling-on-ios/) I mentioned, your error handling is also only a couple of lines:

{% codeblock lang:objc %}
[self presentError:error completionHandler:^(BOOL didRecover) {
	if (didRecover) {
		[self showDetails:sender];
	}
}];
{% endcodeblock %}


## What about the target view controller?

If you cannot load your data in the presenting view controller for whatever reason, you can still use a very similar API design as above and add a couple of guards. The important part is to make sure the reload trigger can be called at every time, regardless of whether your view controller is currently visible or not. Your view controller does not know when its view will be displayed or hidden, so make sure that it does not matter.

If you do that, you can then load the data during initialization or when your view controller is displayed. Your connection layer does not have to care about the surrondings and so do you. The only thing you should ensure is to **not** enqueue the operation in your `viewDidLoad` method. `viewDidLoad` has a very random execution behavior. `UIKit` does not gurantee you that this method is only called once. This method could get called multiple times, but the time depends on the current memory situation. Since iOS6 I have not seen this happen anymore, but it is still the documented way for this method! Anyway, this method basically tells you that your view needs to be configured. As we are dealing with a connection here and a connection deals with your model, this simply has nothing to do with your view being loaded, so don't put it there!

Keep in mind: It is totally valid for a view controller to present another view controller several times instead of instantiating a new one each time! This is one of the reasons why your loading method should not care about the view state of your controlller. The fact that you are not doing this at the moment does not mean you - or anybody else, for that matter - will not do it in the future.

That being said, make sure to call your load method only once during the initial setup of your view controller. If you trigger a reload in `viewWillAppear:`, do not already start a request during initialization. That does not mean that you should only trigger your connection during the setup phase. It is totally valid to have a `reload` action the user can trigger or poll again after a certain timeout.

Make sure to keep a reference to the connection or the operation that you started to

0. cancel it when dealloc is called
0. Check in `viewDidLoad` if you are still loading and present a loading indicator in that case

In your completion handler you then check if your view has already been loaded and dismiss the loading indicator, if this was the case.

{% codeblock lang:objc %}
@property (nonatomic, weak, readwrite) AFHTTPRequestOperation *operation;

- (instancetype)initWithCoder:(NSCoder *)aDecoder {
	self = [super initWithCoder:aDecoder];
	if (self) {
		[self loadDataIfNeeded];
	}
	return self;
}

- (instancetype)initWithNibName:(NSString *)nibNameOrNil bundle:(NSBundle *)nibBundleOrNil {
	self = [super initWithNibName:nibNameOrNil bundle:nibBundleOrNil];
	if (self) {
		[self loadDataIfNeeded];
	}
	return self;
}

- (void)loadDataIfNeeded {
	if (self.connection || self.data) {
		return;
	}
	
	if (self.isViewLoaded) {
		// show loading indicator
	}
	self.operation = [self loadData:^(id data, NSError *error) {
		if (self.isViewLoaded) {
			// hide loading indicator
		}
		if (error) {
			// handle the error (e.g. present it)
		} else {
			self.data = data;
		}
	}];
}

- (void)viewDidLoad
{
    [super viewDidLoad];
	
	if (self.operation) {
		// present loading indicator
	}
}
{% endcodeblock %}

If you use this and make your data property's setter handle the reload as described in my post about [model handling in UIViewController](/blog/2014/06/22/model-handling-in-uiviewcontroller/), you should be good to go with this approach.

By making your connection property `weak` when using `AFNetworking`, you have the benefit that as soon as your connection finished, it will be deallocated as the queue is no longer retaining it. With this trick you do not need to `nil` your property and you can easily check if a connection is already running.


## Conclusion

No matter which of the described approaches you use, keep in mind that most of your users are in a bad network connection! Give them the ability to retry easily if the connection fails. Also try to make your connections state as easily detectable as possible. Your view controller should be able to handle a connection at every point in time, no matter if your UI is visible or not. If you follow this rule, you don't have to worry about when to reload your data. You can always call `reloadDataIfNeeded`.
