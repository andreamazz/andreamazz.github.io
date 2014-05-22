---
layout: post
title: "UIKit Dynamics"
date: 2014-05-22 19:19
comments: true
categories: ios uikit 
---
I'm late to the party with this, but I finally had a chance to play around with UIKit Dynamics. I like what I see, I really like it... I'll spend the rest of my days implementing physics based UI elements... UIImageView slingshots, UISwitch trabuchets, UISlider catapults... you name it. Jokes aside, a Github user pointed out in the issue section of [AMWaveTransition](https://github.com/andreamazz/AMWaveTransition) that Facebook Paper's _wavy_ transition also implements the interactive pop gesture. "Pffftt... easy" I thought. I was wrong. This is how I found out about my new favorite framework. 
<!-- More -->
###UIPercentDrivenInteractiveTransition
I started doing some research on custom interactive transitions, and I stumbled upon `UIPercentDrivenInteractiveTransition`. The logic behind it is really clever, the developer just needs to provide a custom animation and use a UIPanGestureRecognizer to keep track of the completion percentage, and notify the system with such value. UIKit does its magic and the animation is performed in small steps. 
That sounds easy enough, but AMWaveTransition uses a couple of hacky ways to achieve its effect, and it's not based on a single animation block, but in different (and nested) animations. 
I was not surprised when I found out that my percent driven transition didn't work. At all.  
Besides that, if you look closely to the Paper's implementation, you can see that the cells react to the finger's Y coordinate, not just the X. This means that the cell below the finger is the one 'leading the charge', the other cells lag behind it. That smells a lot like a custom transition, not a percent driven one.

###UIScreenEdgePanGestureRecognizer
"Ok, no UIKit magic for me this time, I'll just write the animations from scratch.". A `UIScreenEdgePanGestureRecognizer` and some UIView's animations sounded like the right tools. I just had to figure out what cell to move first, and let the other lag behind, taking account of the swipe velocity. The gestures' tools make that easy enough, you can get the state and velocity of the finger, and apply different animations with different durations and/or delays. Running an animation for each time that the swipe gesture changes isn't a good idea though. A lot of animations get queued up, and even whith the option `UIViewAnimationOptionBeginFromCurrentState` the resulting effect just isn't right. Also, managing the possibility that the user might swipe back and forth before completing or cancelling increases the complexity. 

I didn't want that.

###Dynamics!
UIKit Dynamics, introduced with iOS7, is a physics engine that lets you apply physics traits to UIKit elements. Its implementation is brilliantly simple, there's an Animator object that handles the various behaviors associated to an interface object. The animator is attached to a reference view (basically the container of all the other views) and handles all the animations, defined by some predefined behaviors (and you can define your own): 

- UIGravityBehavior
- UICollisionBehavior
- UIAttachmentBehavior
- UISnapBehavior
- UIPushBehavior

Each behavior is configurable, and can be combined with the others. 

####UIAttachmentBehavior

From its [class reference](https://developer.apple.com/library/ios/documentation/uikit/reference/UIAttachmentBehavior_Class/Reference/Reference.html): _"An attachment behavior specifies a dynamic connection between two dynamic items, or between a dynamic item and an anchor point. By default, an item’s attachment point is at its center, but you can change that."_. I can change that... sounds good. I can attach a view to an anchorpoint, and update such point, say to follow the user's swipe. It also features a couple of nifty properties:
{% codeblock lang:objc %}
// The amount of damping to apply to the attachment behavior.
@property(readwrite, nonatomic) CGFloat damping
// The frequency of oscillation for the attachment behavior.
@property(readwrite, nonatomic) CGFloat frequency
{% endcodeblock %}
These property will make the attachment springy. Feeling confident.

###Interactive pop
Performing the interactive pop now should be just a matter of making some calculation, and figure out which view to move and where. I started by adding a `UIScreenEdgePanGestureRecognizer` to the UINavigationController provided by the user:
{% codeblock lang:objc %}
- (void)attachInteractiveGestureToNavigationController:(UINavigationController *)navigationController
{
    self.navigationController = navigationController;
    self.gesture = [[UIScreenEdgePanGestureRecognizer alloc] initWithTarget:self action:@selector(handlePan:)];
    [self.gesture setEdges:UIRectEdgeLeft];
    [navigationController.view addGestureRecognizer:self.gesture];
    
    self.animator = [[UIDynamicAnimator alloc] initWithReferenceView:navigationController.view];
    self.attachmentsFrom = [@[] mutableCopy];
    self.attachmentsTo = [@[] mutableCopy];
}
{% endcodeblock %}
As you can see I also created a `UIDynamicAnimator`, he's the good guy that'll do all the heavy lifting in a while.

####Handling the gesture
Once a swipe gesture from the left edge is performed, I need to get a hold of the cells from both the view controllers (the current visible controller, and the one we are swiping back to):

{% codeblock lang:objc %}
// Starting controller
UIViewController<AMWaveTransitioning> *fromVC;
fromVC = (UIViewController<AMWaveTransitioning> *)self.navigationController.topViewController;

// Controller that will be visible after the pop
UIViewController<AMWaveTransitioning> *toVC;
int index = (int)[self.navigationController.viewControllers indexOfObject:self.navigationController.topViewController];
toVC = (UIViewController<AMWaveTransitioning> *)self.navigationController.viewControllers[index-1];
{% endcodeblock %}

While I'm at it I also ask the gesture for the touch position and the swipe horizontal velocity:

{% codeblock lang:objc %}
float velocity = [gesture velocityInView:self.navigationController.view].x;
CGPoint touch = [gesture locationInView:self.navigationController.view];
{% endcodeblock %}

Now the fun part. When the gesture starts, I need to figure out which cell is below the touch point, and then wire up a `UIAttachmentBehavior` to each cell. I'll be storing the behavior in a mutable array, and add it to our trusty animator. The anchor point will be the touch's X coordinate, while the Y coordinate will be the center of the cell (playing around with this coordinate can be fun, it's worth a try). 

{% codeblock lang:objc %}
[[fromVC visibleCells] enumerateObjectsUsingBlock:^(UIView *view, NSUInteger idx, BOOL *stop) {
    // The 'selected' cell will be the one leading the other cells
    if (CGRectContainsPoint([view.superview convertRect:view.frame toView:nil], touch)) {
        self.selectionIndexFrom = (int)idx;
    }
    UIAttachmentBehavior *attachment = [[UIAttachmentBehavior alloc] initWithItem:view
                                                                 attachedToAnchor:(CGPoint){touch.x, [view.superview convertPoint:view.frame.origin toView:nil].y + view.frame.size.height / 2}];
    [attachment setDamping:0.4];
    [attachment setFrequency:1];
    [self.animator addBehavior:attachment];
    [self.attachmentsFrom addObject:attachment];
}];
{% endcodeblock %}

I need to do this for both `fromVC` and `toVC`. 

####Moving the cells
Believe it or not, the hard part is over. Thanks to UIKit Dynamics handling the gesture change is easy as updating the point:

{% codeblock lang:objc %}
if (gesture.state == UIGestureRecognizerStateChanged) {        
  [[fromVC visibleCells] enumerateObjectsUsingBlock:^(UIView *view, NSUInteger idx, BOOL *stop) {
      [self.attachmentsFrom[idx] setAnchorPoint:(CGPoint){touch.x, [view.superview convertPoint:view.frame.origin toView:nil].y + view.frame.size.height / 2}];
  }];
}
{% endcodeblock %}

The code above just moves the cells in the old fashioned way. You might point out that that's not _wavy_ at all. Well... it's not. 
To make it more interesting we need to play around with the velocity and the distance between cells.
{% codeblock lang:objc %}
float delta = touch.x - abs(self.selectionIndexFrom - (int)idx) * velocity / 50;
[self.attachmentsFrom[idx] setAnchorPoint:(CGPoint){delta, [view.superview convertPoint:view.frame.origin toView:nil].y + view.frame.size.height / 2}];
{% endcodeblock %}
This takes the touch value, subtracts an amount of pixel determined by the distance from the starting cell, multiplied by the velocity (that is toned down by the magic value 50, provided by wisdom. And by wisdom I mean trial and error). 
If you set this up and play around with it a bit you'll notice that it's easy to break stuff. If you swipe too fast, `delta` becomes huge, and the latch changes direction. Fixing issues like this can be a pain, fortunately the open source community comes to the rescue.

####DynamicXray
[DynamicXray](http://dynamicxray.net/) is a tool that provides a graphical runtime representation of the inner workings of a UIKit animator. You just add it to your animator like you would with a behavior, and all your bounds, constraints and forces are represented alongside your views. That's really handy to figure out why a view is acting weird.

{% img center /images/posts/2014-05-22/xray.png 'Result' %} 

With the debug wireframe on I was able to see that an excessive swipe caused the latch to change direction, causing the view to follow the touch coordinates from the wrong side of the screen. It's an easy fix, that involves another magic number (2):

{% codeblock lang:objc %}
if (delta > view.frame.origin.x + view.frame.size.width / 2) {
    delta = view.frame.origin.x + view.frame.size.width / 2 - 2;
}
{% endcodeblock %}

Needles to say, I'm not happy with this solution, but for now it would do.

With all this set up, the rest was just maintenance... I just need to clean everything up when the gesture reaches its end, and call `popViewControllerAnimated` on the navigation controller.

And there we go:

{% img center /images/posts/2014-05-22/screenshot.gif 'Result' %} 

As usual, you can find the code on [Github](https://github.com/andreamazz/AMWaveTransition).

Until next time.


