---
layout: post
title: "An improved way to integrate Reveal into your Xcode project"
date: 2013-11-23 15:06:00 +0200
comments: true
categories: tools

---

Oliver Jones had a great article at the Itty Bitty Apps Blog about how to '[Integrating Reveal without modifying your Xcode project](http://blog.ittybittyapps.com/blog/2013/11/07/integrating-reveal-without-modifying-your-xcode-project)' that was also featured in [iOS dev weekly](http://iosdevweekly.com/issues/120 "iOS Dev Weekly Issue #120"). Reveal is a great tool to debug layout issues. I highly encourage everybody to [check it out](http://revealapp.com "Reveal website"). It improves debugging UI related issues like `UIView` position and clipping or `CALayer` ordering and really makes your life easier. Oliver's blog post explained a great way to integrate Reveal into your Xcode project without linking it automatically and without having it automatically starting up during app start. There are a couple of benefits with this approach.

<!--more-->

As the library is not linked automatically it is not discovered by Apple's static analyzer. This is great in the case that you forget to remove it before submitting to the App Store. I don't know if the Reveal library contains private APIs, but as it is a library that is meant to be used during development, I would consider this a possibility, if not at the moment maybe in the future.

The other benefit is that, as the Reveal service is not automatically run at app launch, you can debug your app without interference of the library and only fire up the library when you really need it. In the case you forget to remove it before the App Store build, people that get your app through the App Store still are not able to analyze your view hierarchy with Reveal.

A third great improvement is very useful for all the people that are working in a big office with dozens of other developers. If you have the Reveal library enabled in every build everybody makes, it becomes quite a challenge to select the right device in Reveal. You only want the Reveal service up and running when you really want to debug something.

Oliver used the LLDB init file `~/.lldbinit` to add a couple of aliases to load and start the Reveal library from the debugger:

- `reveal_load_sim` - loads the Reveal library in the iOS simulator
- `reveal_load_dev` - loads the Reveal library on the device
- `reveal_start` - starts the Reveal library
- `reveal_stop` - stops the Reveal library

However there are a couple of issues with the described approach that bothered me and that don't work very well with the workflow we have in the company I work at:

You still need to add the library to the copy step of your project file. Chances are high that you forget to remove this before running the App Store build. Even though the library is neither linked nor started automatically you don't want a library that you don't need and that increases your app by almost 4MB in your App Store ipa. This might make the difference between being downloadable through WiFi only or not. Besides that, it looks unprofessional in the case someone discovers this and it might open some security threads you don't think of.

When adding the library to your copy step you have two options: Use the library that is located in the `Reveal.app` folder or copy it to your project. In the first case you break the build process for everybody that doesn't have Reveal installed as the file will not be in the place where Xcode is looking for it. With the second approach you will run in to problems when different engineers have different versions of Reveal on their machines. it is pretty likely anyway that you will forget to update the library in the first place and only notice this when you really need it. Updating the library requires you to stop the running application and rebuild it after you updated the library. This reduces the benefit of the automatic approach dramatically.

Last but not least I don't want to have two load commands, `reveal_load_sim` and `reveal_load_dev`. This is more complicated than it should be. The debugger should be able to figure that out by itself or at best use the exact same approach on the device and simulator.

## reveal_load

Let's try to fix that little inconvenience first. With Oliver's approach we use the following aliases:

{% codeblock ~/.lldbinit %}
command alias reveal_load_sim expr (void*)dlopen("/Applications/Reveal.app/Contents/SharedSupport/iOS-Libraries/libReveal.dylib", 0x2);
command alias reveal_load_dev expr (void*)dlopen([(NSString*)[(NSBundle*)[NSBundle mainBundle] pathForResource:@"libReveal" ofType:@"dylib"] cStringUsingEncoding:0x4], 0x2);
{% endcodeblock %}

The first alias is `reveal_load_sim`. It loads the Reveal library from the `Reveal.app` folder. This is a path that you obviously can only reach in the simulator. The second alias, `reveal_load_dev`, loads the `libReveal.dylib` from the application bundle. This only works when you add the library to your copy step before.

When you need to copy and update the library manually within your project it is quite helpful to be able to use Reveal in the simulator without copying the library first. As we try to improve this behavior in the next step anyway, let's simply remove the `reveal_load_sim` macro and rename `reveal_load_dev` to `reveal_load`. Our complete list of Reveal aliases in the `~/.lldbinit` now looks like this:

{% codeblock ~/.lldbinit %}
command alias reveal_load expr (void*)dlopen([(NSString*)[(NSBundle*)[NSBundle mainBundle] pathForResource:@"libReveal" ofType:@"dylib"] cStringUsingEncoding:0x4], 0x2);
command alias reveal_start expr (void)[(NSNotificationCenter*)[NSNotificationCenter defaultCenter] postNotificationName:@"IBARevealRequestStart" object:nil];
command alias reveal_stop expr (void)[(NSNotificationCenter*)[NSNotificationCenter defaultCenter] postNotificationName:@"IBARevealRequestStop" object:nil];
{% endcodeblock %}

## Using a script instead of the copy phase

Now that we optimized the load command let's make sure that we always have the library version that our currently installed version of Reveal needs and that Reveal is not included if it is not available on the machine that is performing the build. Instead of adding the framework to the copy step we simply use a 'run script' phase in our build progress to copy the framework into the .app folder. This script can check for the configuration you are currently building and wether the framework exists in the `Reveal.app` folder.

{% codeblock lang:sh %}
APPBUNDLEPATH="${TARGET_BUILD_DIR}/${EXECUTABLE_NAME}.app/"
REVEALFRAMEWORKPATH="/Applications/Reveal.app/Contents/SharedSupport/iOS-Libraries/libReveal.dylib"
if [ -f "${REVEALFRAMEWORKPATH}" ] && [ "${CONFIGURATION}" != "ReleaseAppStore" ]; then
 cp "${REVEALFRAMEWORKPATH}" "${APPBUNDLEPATH}"
fi
{% endcodeblock %}

The first line defines the path to the .app folder of the application we are building. The second line defines the location of the `libReveal.dylib` file if you have a standard Reveal installation. Now we only have to perform two checks:

- *Does the libReveal.dylib file exist?* This is the first part of the condition and makes sure the build is successful on machines that do not have Reveal installed or where Reveal is installed in a different location.
- *Is the configuration we are currently building NOT `ReleaseAppStore`?* This is the configuration we are using for App Store distribution. You might need to change this to meet your environment. Maybe you are simply using the default `Release` build for App Store distribution or maybe you have another configuration name. Simply replace `ReleaseAppStore` to whatever configuration you are using. This ensures that the library is only copied when you are not building for the App Store.

If you save this file somewhere in your project path, all you have to do is add a 'run script' phase to your project and call this project by inserting the following line in the phase:

{% codeblock lang:sh %}
bash "MyProject/Supporting Files/lazy-copy-reveal.sh"
{% endcodeblock %}

Adjust this path to meet your environment and you are ready to go.

With this script and the above LLDB aliases everybody that has Reveal installed can simply pause the application at any time in the debugger and simply type `reveal_load` followed by `reveal_start`. After that, don't forget to resume your application. You can now immediately attach Reveal to your device. The library that is used in your project will automatically stay up to date with your Reveal version and you don't have to change anything in your project before submitting to the App Store.

I am only using this approach for a couple of days now and it has already been very useful as you can fire up Reveal anytime you have a UI issue without the need of restarting the application and reproducing the bug first.

I uploaded the two files you need to gist in the case you want to simply copy them:

- `.lldbinit`: [https://gist.github.com/michaelochs/7614557](https://gist.github.com/michaelochs/7614557 ".lldbinit on gist")
- `lazy-copy-reveal.sh`: [https://gist.github.com/michaelochs/7614574](https://gist.github.com/michaelochs/7614574 "lazy-copy-reveal.sh on gist")
