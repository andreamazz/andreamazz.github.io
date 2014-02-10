---
layout: post
title: "AMScrollingNavbar: creating a cocoapod"
date: 2014-02-01 12:07
comments: true
categories: ios objc cocoapods
---
This week [Matt Thompson](https://twitter.com/mattt) published an interesting article on NSHipster, about [stewardship](http://nshipster.com/stewardship/), which is basically _the duty and ethic of public service_. Since I've been trying to be more active in the open source community, I figured I could use this occasion to write an article with the process that I follow when I'm writing a new library for iOS. I'll be describing my work on [AMScrollingNavbar](https://github.com/andreamazz/AMScrollingNavbar) as an example.
<!-- More -->
```AMScrollingNavbar``` is a simple iOS library that lets you implement a ```UINavigationBar``` that scrolls out of the way when the user is scrolling the content of the app. You can observe this behavior in Google Chrome, Instagram or Facebook apps. I wanted to integrate something like that on one of the app I'm working on, over at [Fancy Pixel](http://www.fancypixel.it), but couldn't find anything that did the job as I intended. What better motivation to do some OSS?

###Writing the code
In cases like this, writing the code is more a series of trial and error, so instead of TDD I use a more _sample driven_ approach. This means that I just fire up XCode, create a new project and start fiddling around with the SDK. That's the best part of the job.  

Letting the ```UINavigationBar``` scroll out of the way was fairly easy, I just needed to add a ```UIGestureRecognizer``` to the ```UIScrollView``` with the content.
{% codeblock lang:objc %}
self.panGesture = [[UIPanGestureRecognizer alloc] initWithTarget:self action:@selector(handlePan:)];
[self.panGesture setMaximumNumberOfTouches:1];
[self.panGesture setDelegate:self];
[self.scrollableView addGestureRecognizer:self.panGesture];
{% endcodeblock %}

That alone won't do the trick though, we need to override a delegate method of the ```UIGestureRecognizerDelegate``` protocol:
{% codeblock lang:objc %}
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRecognizeSimultaneouslyWithGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer
{
	return YES;
}
{% endcodeblock %}
Returning ```YES``` in this method will allow both the scrollview and the gesture recognizer to work at the same time. Cool. 
Now it's time to stretch ```UINavigationBar```'s leg. Here's an abstract of the code that does that:
{% codeblock lang:objc %}
- (void)scrollWithDelta:(CGFloat)delta
{
	CGRect frame;
	
	frame = self.navigationController.navigationBar.frame;
	
	if (frame.origin.y - delta < -self.deltaLimit) {
		delta = frame.origin.y + self.deltaLimit;
	}
		
	frame.origin.y = MAX(-self.deltaLimit, frame.origin.y - delta);
	self.navigationController.navigationBar.frame = frame;
		
	[self updateSizingWithDelta:delta];
}
{% endcodeblock %}
As you can see the ```UINavigationBar``` has its own frame, that you can easily modify. Once I got the navigation bar following the scroll view, I needed to enlarge or reduce the scroll views' frame to leverage all the remaining screen estate. Big chunk of code coming up:
{% codeblock lang:objc %}
- (void)updateSizingWithDelta:(CGFloat)delta
{
	// At this point the navigation bar is already been placed in the right position, it'll be the reference point for the other views'sizing
	CGRect frame = self.navigationController.navigationBar.frame;

	[self updateNavbarAlpha:delta];

	// Move and expand (or shrink) the superview of the given scrollview
	frame = self.scrollableView.superview.frame;
	frame.origin.y -= delta;
	frame.size.height += delta;
	self.scrollableView.superview.frame = frame;

	frame = self.scrollableView.frame;
	frame.size.height = self.scrollableView.superview.frame.size.height - frame.origin.y;

	// if the scrolling view is a UIWebView, we need to adjust its scrollview's frame.
	if ([self.scrollableView isKindOfClass:[UIWebView class]]) {
		((UIWebView*)self.scrollableView).scrollView.frame = frame;
	} else {
		self.scrollableView.frame = frame;
	}

	// Keeps the view's scroll position steady until the navbar is gone
	if ([self.scrollableView isKindOfClass:[UIScrollView class]]) {
		[(UIScrollView*)self.scrollableView setContentOffset:CGPointMake(((UIScrollView*)self.scrollableView).contentOffset.x, ((UIScrollView*)self.scrollableView).contentOffset.y - delta)];
	} else if ([self.scrollableView isKindOfClass:[UIWebView class]]) {
		[((UIWebView*)self.scrollableView).scrollView setContentOffset:CGPointMake(((UIWebView*)self.scrollableView).scrollView.contentOffset.x, ((UIWebView*)self.scrollableView).scrollView.contentOffset.y - delta)];
	}
}
{% endcodeblock %}
Fairly easy to read, kinda tricky to write. The scrollview's height is increased by the scroll's delta, and its origin is shifted up or down of the same amount. With a little reflection I can check wether the scroll view is a normal ```UIScrollView``` (and this includes table views) or a ```UIWebView```. The later has its own scroll view, so I need to adjust that instead.  
With the view sizing complete, the last step was to fade out the navigation items. I used a hacky approach, since I wasn't able to set the alpha channel of every nav item, I figured I could just impose an overlay view with the same ```barTintColor```, and change its alpha back and forth:
{% codeblock lang:objc %}
- (void)updateNavbarAlpha:(CGFloat)delta
{
	CGRect frame = self.navigationController.navigationBar.frame;
	
	// Change the alpha channel of every item on the navbr. The overlay will appear, while the other objects will disappear, and vice versa
	float alpha = (frame.origin.y + self.deltaLimit) / frame.size.height;
	[self.overlay setAlpha:1 - alpha];
	[self.navigationItem.leftBarButtonItems enumerateObjectsUsingBlock:^(UIBarButtonItem* obj, NSUInteger idx, BOOL *stop) {
		obj.customView.alpha = alpha;
	}];
	[self.navigationItem.rightBarButtonItems enumerateObjectsUsingBlock:^(UIBarButtonItem* obj, NSUInteger idx, BOOL *stop) {
		obj.customView.alpha = alpha;
	}];
	self.navigationItem.titleView.alpha = alpha;
	self.navigationController.navigationBar.tintColor = [self.navigationController.navigationBar.tintColor colorWithAlphaComponent:alpha];
}
{% endcodeblock %}
And that did the trick.  

###Getting camera ready
Once the sample runs like I intended, I step in the refactoring phase. In this step I just take all the code and move it in a reusable object. In this case I opted for a ```UIViewController``` that can be subclassed by other developers. I valued the possibility of a category, but since I make an extensive use of instance variables, I preferred the first approach (not a fan of the ```associatedObject``` trick).  
The last item on the todo list is the refactoring of the sample project. And we're done.
###Writing the documentation
An open source project is as good as its documentation is. It helps the first implementation by other devs, and most of all helps them decide wether to use your library or not. In some way it's marketing, without the _market_ part.  
I like to spend some time to add comments following the [Appledoc notation](http://nshipster.com/documentation/). This will provide a comprehensive quick guide in XCode, and also will generate an Apple-like documentation in the [Cocoapods pages](http://cocoadocs.org/docsets/AMScrollingNavbar/0.5.4/Classes/AMScrollingCollectionViewController.html).
As you can see the syntax is easy to write and to understand:
{% codeblock lang:objc %}
/** Scrolling init method
 *
 * Enables the scrolling on a generic UIView.
 *
 * @param scrollableView The UIView where the scrolling is performed.
 */
- (void)followScrollView:(UIView*)scrollableView;
{% endcodeblock %}
  
Then it's time to push all the work on Github, and this calls for a good README file. The README should have a quick description of the library, the setup instructions, the library's documentation or a reference to it, and a screenshot, when possible.  
####Screenshot
Following the principle _"show, don't tell"_, screenshots work great, but GIFs are even better. Creating a GIF animation with your sample running is easy as pie, using [LICEcap](http://www.cockos.com/licecap/). Despite its weird name and lowres icon, LICEcap is an invaluable tool, easy to use and extremely versatile, you just fit the portion of the screen that you want to record inside its window, hit record, and you are done.

{% img center /images/posts/2014-02-01/licecap.png 'LICEcap' %}

You might want to resize the output GIF, if you are capturing a big part of the screen, or if you are working on a retina display. [ImageMagick](http://www.imagemagick.org/script/index.php) comes to the rescue:
{% codeblock %}
convert big.gif -coalesce temp.gif
convert -size 960x640 temp.gif -resize 480x320 small.gif
{% endcodeblock %}

###Travis CI
[Travis CI](https://travis-ci.org/) is a continuous integration tool, used to build and test projects hosted on Github. Having Travis enabled in an open source project is a great way to make sure that your project _compiles_ fine. This is awesome for your new releases (you know before hand if your last commit broke something) and even better for pull requests, since Travis tells you wether a pull request compiles without errors, directly in the github page of the PR.  
To start using Travis, head [here](https://travis-ci.org/), login with Github, and enable your project from the list in your dashboard.  
Enabling Travis with iOS is really easy, it only takes two steps:
####Configure build scheme
Open your sample project, or the project component that you want to build, and in the Manage Schemes panel, make sure that your project scheme is ```Shared```:
{% img center /images/posts/2014-02-01/schemes.png 'Schemes' %}

####Add .travis.yml
Add a ```.travis.yml``` file in your project root. You'll be writing a simple script that will compile your sample project:
{% codeblock %}
language: objective-c
install:
- cd ScrollingNavbarDemo
script: xctool -project ScrollingNavbarDemo.xcodeproj -scheme 'ScrollingNavbarDemo' -configuration Release -sdk iphonesimulator7.0 -arch i386 build
{% endcodeblock %}
The ```install``` phase just changes the directory to the one containing the project, then our trusty xctool compiles the project. Doesn't get much easier than that.  

Once you pushed the new release, the project will be added to the travis compile queue, and you'll receive an email with the result. You can also see the script running in real time, which is awesome.

###Cocoapods
There is no need to reiterate that [Cocoapods](http://cocoapods.org/) is _great_. The times when you spent precious time configuring a new library in your project are ([almost](/blog/2014/01/25/configuring-alljoyn-on-ios/)) over. Creating a pod is a matter of minutes, and at the same times provides great value for the community.  
You'll find a pretty extensive guide on the [cocoapods site](http://guides.cocoapods.org/making/making-a-cocoapod.html).
Here's what my podspec looks like:
{% codeblock lang:ruby %}
Pod::Spec.new do |s|
  s.name         = "AMScrollingNavbar"
  s.version      = "0.5"
  s.summary      = "Scrollable UINavigationBar that follows the scrolling of a UIScrollView. Similiar to Chrome for iOS7"
  s.homepage     = "https://github.com/andreamazz/AMScrollingNavbar"
  s.license      = { :type => 'MIT', :file => 'LICENSE' }
  s.author       = { "Andrea Mazzini" => "andrea.mazzini@gmail.com" }
  s.source       = { :git => "https://github.com/andreamazz/AMScrollingNavbar.git", :tag => '0.5' }
  s.platform     = :ios, '5.0'
  s.source_files = 'AMScrollingNavbar', '*.{h,m}'
  s.requires_arc = true
end
{% endcodeblock %}
Once the spec is merged in the Spec repo, a new pod is born, and with it the documentation page is also generated.

####Cocoacontrols
[Cocoacontrols](https://www.cocoacontrols.com/) is my favorite place to window shop for new libraries. As soon as my pod is ready and I feel that the README is comprehensive enough, I submit my control to cocoacontrols' review queue. It's a great way to get some visibility from fellow developers, and a great way to discover new controls. A big thanks to [Aaron, Marine and Bob](https://www.cocoacontrols.com/about).

###Maintaining the code
Once the code is in the wild, it's time to maintain the code. This means answering questions about its use, fixing possible issues and merging the always welcome pull requests. It can become a job of its own at times, but it's a high reward task, since it gives you the chance to experiment more and improve your code quite a lot.  
The only tool that I use is the ```Issue``` section of github. I tried once to get into the habit of tracking the issues with [Waffle](https://waffle.io/), but in the end I never really used it efficiently.  
AMScrollingNavbar was well received, and has now (what is for me) a fair amount of stargazers on Github. I'd like to thank all the contributors that helped to improve the library, introducing new features and fixing my missteps.  

That about wraps up this auto referential post.  
Until next time.  
Andrea