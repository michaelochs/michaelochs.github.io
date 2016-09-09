---
layout: post
title: "Drawing buttons with blurred background"
date: 2013-05-09 17:25:00 +0200
comments: true
categories: opensource

---

A glossy looking button is a well known and widely used type of button that everybody knows if he or she hasn't been on the moon the last couple of years. Apple uses these buttons excessive. In OS X as well as on iOS. Those buttons look nice and they work very well when trying to achieve a polished, three dimensional look. However, they are always used with a background color that differs from the surrounding background or the surrounding background doesn't have much detail. This is because if you use this button on a background that has a lot of detail, like a photo, they don't look like glass anymore, at all. The button looks very flat if you can see, that the background isn't distorted by the 'glass' in any kind.

<!-- more -->

## CoreImage

The solution to this problem sounds quite easy at first: As soon as you simply blur the background just a little, the whole thing starts looking great again. This is where CoreImage comes in handy. CoreImage has a ton of useful filters, one of them is a gaussian blur filter. Using this filter is pretty easy. All you need to do is specify you input image and the blur radius you would like to use.

{% codeblock lang:objc %}
CIFilter* blur = [CIFilter filterWithName:@"CIGaussianBlur"];
[blur setValue:@(blurRadius) forKey:@"inputRadius"];
[blur setValue:inputImage forKey:@"inputImage"];
CIImage* outputImage = [blur outputImage];
{% endcodeblock %}

*Note that you are dealing with `CIImage`'s instead of `UIImage`'s. So `inputImage` has to be a `CIImage*`.*

## What to blur?

Now that we know how to blur, the question would be: What to blur? And here is the point where things are getting difficult. There are three methods on CALayer that sound promising:

- `setBackgroundFilters:` *"An array of Core Image filters to apply to the content immediately behind the layer"*
- `setFilters`: *"An array of Core Image filters to apply to the contents of the layer and its sublayers"*
- `setCompositingFilter`: *"A CoreImage filter used to composite the layer and the content behind it"*

All three methods are available on iOS, you can call them, the filters are set and they will be returned if you call the getter. But that's pretty much all these methods do, at least up to iOS6. Under *"special considerations"* Apple says *"This property is not supported on layers in iOS"*. Note that there was an error in the documentation on iOS 6.0, where it said *"This property relies on the presence of Core Image and its filters. In iOS, Core Image is available in iOS 5 and later only"*. They changed this in the iOS 6.1 documentation, so don't wonder if you see the second text, you selected the *old* version of the `CALayer` documentation.

So using one of these properties won't work on iOS. The only other approach I am aware of on iOS, is to render the underlaying layers into a context and blur it yourself. You can do this with `CALayer` again. The method you are looking for is `-[CALayer renderInContext:]`

This method renders the receiver and it's sublayers to the context you specify. 'But I want to render the views!' you might think. That's correct, but `UIView` is just the delegate of the underlying layer. Each `UIView` has a layer it is responsible for and the layer hierarchy is the same as the view hierarchy - unless you change something about this by yourself. The visible part of a `UIView` is the `CALayer`! If a view has five subviews, its layer has five sublayers and each of these sublayers belong to one of the subviews.

## Drawing

To render the background of the button we want to blur, we simply walk up the view hierarchy until we find the first view that is opaque. If the view is opaque we can safely render this view and its subviews without worrying about transparent parts from underlaying views.

To achieve this, we create a new graphics context in the size of the button that should get the blurred background. We don't need to render the whole superview we just found, as we only need to blur the part that is behind our button. The rendered image is what we will use as our input image for the blur filter. Don't forget to hide the button during rendering, otherwise the button itself will be contained in the button's background.

{% codeblock lang:objc %}
self.hidden = YES;
	
CGRect renderRect = [viewToRender convertRect:rect fromView:self];

UIGraphicsBeginImageContext(rect.size);

CGContextTranslateCTM(UIGraphicsGetCurrentContext(), -CGRectGetMinX(renderRect), CGRectGetMaxY(renderRect));
CGContextScaleCTM(UIGraphicsGetCurrentContext(), 1.0f, -1.0f);

[viewToRender.layer renderInContext:UIGraphicsGetCurrentContext()];

CGImageRef blurInputReference = [UIGraphicsGetImageFromCurrentImageContext() CGImage];

UIGraphicsEndImageContext();

self.hidden = NO;
{% endcodeblock %}

After rendering the view into a `CGImage` we convert this to a `CIImage` and set it as the input image of our blur filter. If you put all this code into your button's `drawInRect:` method, you can then draw the output image into the current context for displaying. After that, you can call all the other drawing code, if any, you need for your button.

{% codeblock lang:objc %}
CIImage* inputImage = [CIImage imageWithCGImage:blurInputReference];
CGImageRef blurOutput = [context createCGImage:outputImage fromRect:outputImage.extent];
CGContextDrawImage(UIGraphicsGetCurrentContext(), outputImage.extent, blurOutput);
{% endcodeblock %}

*Not that we render the output image with the extent of the `CIImage` as the blur filter will return an image that is larger than your input image. The extent property of you output image has a negative origin so that rendering the image with this extent will place your image exactly above your input image, with a border.*

## BCBlurView on GitHub

I have set up a [GitHub project](https://github.com/michaelochs/BCBlurView "BCBlurView on github") that adds a method to `UIView` that renders the view's background with a given blur radius into the current context. You can call this method on any UIView in it's `drawRect:` method to get a blurred background. The static library also contains two subclasses, one of UIView and one of UIButton. Those adopt this category and, aside from others, make the blur radius available as a property for easier use.

## Special considerations

This approach only works with still backgrounds. If you have any kind of animation in the background, this will not work. If the background changes, you have to call `setNeedsDisplay` on the view that has the blurred background. Note that blurring takes time, it is not possible to e.g. put a `UIScrollView` in the back of the button and call `setNeedsDisplay` on every scroll position change of the scroll view.