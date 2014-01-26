---
layout: post
title: "Configuring Alljoyn on iOS"
date: 2014-01-25 11:29
comments: true
categories: ios alljoyn objc
---
Have you ever played [Spaceteam](http://www.sleepingbeastgames.com/spaceteam/) on [Android](https://play.google.com/store/apps/details?id=com.sleepingbeastgames.spaceteam) or [iOS](https://itunes.apple.com/it/app/spaceteam/id570510529?mt=8)? It's a local (cross-platform) multiplayer only game, based on a really cool concept. The players are... _you guessed it..._ a space team, and they need to cooperate to keep the spaceship they are traveling on intact through space. This means that you'll be [shouting at your friends](http://penny-arcade.com/comic/2012/12/31) commands, and activating various weirdly named knobs and switches.  
While I was researching the tech behind it, I stumbled upon [Alljoyn](https://www.alljoyn.org/), by Qualcomm.
<!-- More -->
Spaceteam uses [HHServices](https://github.com/tolo/HHServices) and [AsyncSocket](https://github.com/roustem/AsyncSocket), so the networking protocol was built almost from the ground up by [its creator](https://twitter.com/hengineer). Alljoyn on the other hand promises an opensource _"universal (cross-platform) software framework that enables interoperability among connected products and software applications across manufacturers to create dynamic proximal networks."_ And I'm all for that.  
The cool thing about it is its ability to seamlessly operate over wifi or over bluetooth (losing the cross-platform ability).
If you login on the Alljoyn site you'll come across the free, opensource, SDK and a bunch of samples. I downloaded the iOS SDK, and started working on my sample.  
What I found out is that the walkthroughs and guides are either not up to date or just wrong. So I figured I could write down what I did to make everything work.  
In this day and age, where cocoapods eases our mind when we need to integrate a third party library, it was a harsh throwback in the world of missing headers and wrong architecture builds.  

####Compiling openssl
If you take a look at Alljoyn's SDK readme you'll notice that the first thing you need to do is compile Openssl for iOS. That's quite easy, and fairly well documented, just head to [Github and clone the openssl-xcode project](https://github.com/sqlcipher/openssl-xcode).  
As stated in the readme you need to download the [openssl sources](http://www.openssl.org/source/) and place them in your openssl-xcode project.  
That's how the folder structure should look:  

{% img center /images/posts/2014-01-25/openssl.png 760 550 'openssl' %}

To compile it just open the .xcodeproj file and use XCode's Product -> Build For -> Release, or stay in your terminal window and compile it with xctool or xcodebuild, either:
{% codeblock %}
xcodebuild -project openssl.xcodeproj -scheme crypto
{% endcodeblock %}
or:
{% codeblock %}
xctool -project openssl.xcodeproj -scheme crypto
{% endcodeblock %}

You'll find the built library in your derived data:

{% img center /images/posts/2014-01-25/derived-data.png 'built openssl' %}

Just grab ```Debug-iphoneos``` and ```Debug-iphonesimulator```, and place them under a ```build``` folder, inside your openssl sources. It should look something like this:
{% codeblock %}
☺   ~/openssl/build ll
drwxr-xr-x  5 Andrea  staff  170 21 Gen 23:26 Debug
drwxr-xr-x  5 Andrea  staff  170 21 Gen 23:28 Debug-iphoneos
drwxr-xr-x  5 Andrea  staff  170 21 Gen 23:28 Debug-iphonesimulator
{% endcodeblock %}

Cool, now download and unzip the Alljoyn's SDK.

####New Alljoyn project
With that out of the way it's time to start working on our project. Fire up Xcode and create a new project.  

#####Object Model
We'll start by creating the object that will travel the bus. Alljoyn uses its own XML format to define the object structure. I'm not a fan of this format, but that's what we have to work with:

{% codeblock lang:xml Object model %}
<?xml version="1.0"?>
<xml>
	<node name="org/alljoyn/Bus/sample">
		<annotation name="org.alljoyn.lang.objc" value="SampleObject"/>
		<interface name="org.alljoyn.bus.sample">
			<annotation name="org.alljoyn.lang.objc" value="SampleObjectDelegate"/>
			<method name="Concatentate">
				<arg name="str1" type="s" direction="in">
					<annotation name="org.alljoyn.lang.objc" value="concatenateString:"/>
				</arg>
				<arg name="str2" type="s" direction="in">
					<annotation name="org.alljoyn.lang.objc" value="withString:"/>
				</arg>
				<arg name="outStr" type="s" direction="out"/>
			</method>
		</interface>
	</node>
</xml>
{% endcodeblock %}

This is the basic object that you'll find in the official guide, it defines a simple method that returns two input strings concatenated:
{% codeblock %}
- (NSString*)concatenateString:(NSString*)str1 withString:(NSString*)str2;
{% endcodeblock %}

Cool, now we should run the code generator bundled in the Alljoyn's SDK to generate the Objective-c code. 
The generator needs to be compiled, so we'll do that in a couple of clicks.

######Compiling the generator
Building the generator is pretty straightforward, just head to ```<AllJoyn SDK Root>/alljoyn_objc/AllJoynCodeGenerator``` and build with XCode the AllJoynCodeGenerator.xcproj. This will create a /bin folder with the generator excecutable file.

Now the good Alljoyn folks suggest to setup a new target that will allow us to generate with XCode the code. Since I like to keep my targets tidy, and since we have a perfectly fine console, I just skip this step and go the old fashioned way. Just copy the bin/AllJoynCodeGenerator wherever you like, as well with our trusty XML object, and run:
{% codeblock %}
./AllJoynCodeGenerator sample_object.xml
{% endcodeblock %}

This will generate:
	AJNSampleObject.h
	AJNSampleObject.mm
	SampleObject.h
	SampleObject.m
Copy these file to your project and add them to your target.

#####Build settings
Now the fun part. We'll need to configure the build settings of our project. What I really like to do is move the AllJoyn's SDK bins and openssl build in the project's root. Two reasons for that:  

- I don't have to configure environment variables and link the relative paths  

- The project is self contained, so everyone can just clone the project and compile it without firther hassles

So, here's my folder structure:
	☺   ~/code/git/alljoynsample ll
	drwxr-xr-x  16 Andrea  staff  544 25 Gen 15:14 AlljoynSample
	drwxr-xr-x   5 Andrea  staff  170 25 Gen 15:14 AlljoynSample.xcodeproj
	drwxr-xr-x   5 Andrea  staff  170 25 Gen 15:14 AlljoynSampleTests
	drwxr-xr-x   5 Andrea  staff  170 25 Gen 15:14 alljoyn-sdk
	drwxr-xr-x   4 Andrea  staff  136 25 Gen 15:14 openssl

Remmeber to remove the unused samples and openssl sources.  

Next step, head to your project's build settings tab, and set ```Build Active Architecture Only``` to YES.  

Below that field, look for ```Other Linker Flags``` and set it to ```-lalljoyn -lajdaemon -lBundledDaemon.o -lssl -lcrypto```.  

That's where I needed to stop following the official guide, since the path described are wrong. 
Head to the ```Header Search Paths``` field and enter these:
	$(inherited)
	/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include
	"$(SRCROOT)/alljoyn-sdk/alljoyn_core/build/darwin/arm/$(PLATFORM_NAME)/$(CONFIGURATION)/dist/cpp/inc"
	"$(SRCROOT)/alljoyn-sdk/alljoyn_core/build/darwin/arm/$(PLATFORM_NAME)/$(CONFIGURATION)/dist/cpp/inc/alljoyn"

Then in ```Library Search Paths```:
	$(inherited)
	"$(SRCROOT)/openssl/build/$(CONFIGURATION)-$(PLATFORM_NAME)"
	"$(SRCROOT)/alljoyn-sdk/alljoyn_core/build/darwin/$(CURRENT_ARCH)/$(PLATFORM_NAME)/$(CONFIGURATION)/dist/cpp/lib"

The $(CURRENT_ARCH) var was key to make the project compile on armv7s devices. That took a while to figure out.

We can now get back to the official guide, setting ```Enable C++ Exceptions``` and ```Enable C++ Runtime Types``` to NO.

Add ```-DQCC_OS_GROUP_POSIX -DQCC_OS_DARWIN``` to _Other C Flags_ Debug and ```-DNS_BLOCK_ASSERTIONS=1 -DQCC_OS_GROUP_POSIX -DQCC_OS_DARWIN``` to _Other C Flags_ Release.

######Frameworks
Almost done. We need to take care of the frameworks. Just copy the ```AllJoynFramework``` folder from the Alljoyn's SDK in your folder and your project, and make sure to remove the ```AJNPasswordManager``` class or you won't be able to compile.
Finish up by linking the static frameworks ```SystemConfiguration``` and ```libstdc++.dylib``` and we are done.

Hit the trusty ```⌘B``` and you should see a succesful build.

Well, that wasn't fun at all, and it sure made me appreciate Cocoapods even more.
Hope this helps anyway.  

Until next time.  
Andrea