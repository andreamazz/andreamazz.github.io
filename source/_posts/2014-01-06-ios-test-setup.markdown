---
layout: post
title: "iOS Test Setup"
date: 2014-01-06 14:39
comments: true
categories: ios specta kif objc
---
Yesterday I wrote an article with my Rails test setup, so it's right about the time to do the same for iOS. I'll be covering Specta, Expecta and KIF, plus OHHTTPStubs for... you know... stubs.
<!-- more -->
First thing first, as you can see from the previous post, I'm not really into Cucumber. I have more of a _dev-first_ approach for specs. That's the benefit of working on your own projects: you only have to explain things to yourself, who is usually a pretty cool guy and understands what you're saying. Most of the times at least.
That being said, I'm not a fan of the syntax that's _behind_ the nice english-like specs, i.e. lots of regex. Welp!
That brings me to my choice for UI Testing: KIF.
KIF (Keep It Functional) is a pretty neat iOS integration framework, it uses the accessibility attributes embedded in every UIKit object to streamline integration testing. This means that you write your tests in plain old Objective-C, which is a big selling point for me.
Tests won't read in fluid english-like sentences, like they do with Cucumber, but they still remain easy to write and easy to understand, even if your objc-fu is not strong.

####Configuration
I'll be using [Cocoapods](http://www.cocoapods.org/), of course.
First of all, you should already have a XCTest target ready. Make sure to add another one for the integration tests. I usually callit 'UI Tests'.
Here's the Podfile:

{% codeblock Podfile lang:ruby %}
	target "Project" do
		# your pods
	end

	target "Tests", :exclusive => true do
		pod 'Specta'
		pod 'Expecta'
		pod 'OHHTTPStubs', '~> 3.0.2'
	end

	target 'UI Tests', :exclusive => true do
		pod 'KIF', '~> 2.0', :inhibit_warnings => true
		pod 'OHHTTPStubs', '~> 3.0.2'	
	end
{% endcodeblock %}

#####Feature tests

After a ```pod install``` everything will be ready, so we can ```open Project.xcworkspace```
As for Specta we should already be set, so we can just head to our Tests target and start adding specs. Note that I added also Expecta, to have a RSpec 2 like syntax.
{% codeblock lang:objc%}
#import <Specta.h>
#define EXP_SHORTHAND
#import <Expecta.h>	

SpecBegin(Thing)

describe(@"Something cool", ^{
	it(@"does something amazing", ^{
		expect(1).to.equal(1);
	});
});

SpecEnd
{% endcodeblock %}

Hit ```⌘ + U``` and you should be green.

#####Integration tests
Now, Specta needs no further configurations, but KIF does.
[KIF documentation](https://github.com/kif-framework/KIF) suggest to make sure that the Bundle loader and Test Host are correctly configured, but I always found them already set by XCode to the proper values. 
Since now XCode ships with XCTest, I had to switch back to ```octest``` in the Wrapper Extension property, under Build Settings.
Everything should be ok, let's write a test.
{% codeblock UselessTest.h lang:objc%}
#import <KIF/KIF.h>

@interface UselessTest : KIFTestCase

@end
{% endcodeblock %}

{% codeblock UselessTest.m lang:objc%}
#import "UselessTest.h"

@implementation UselessTest

- (void)doSomething
{
	[tester tapScreenAtPoint:(CGPoint){0,0}];
}

@end

{% endcodeblock %}

Again ```⌘ + U``` and we are good.

###xctool
[xctool](https://github.com/facebook/xctool) is a great way to run your tests outside of xcode, and with much better output than xcodebuild. Using it is quite simple, install it via [homebrew](http://brew.sh/):
	brew install xctool

then run your test with it:
	xctool -workspace Project.xcworkspace -scheme Project -sdk iphonesimulator test
	
That's it.
Until next time.

Cheers

Andrea

