---
layout: post
title: "Transitioning"
date: 2014-04-19 19:30
comments: true
categories: ios objc
---
I like to kick the morning off by visiting a handful of sites that collect cool app designs. One of my favorite is [Capptivate](http://capptivate.co). I often find myself browsing the site thinking "I want one of that.". It's even better if the design triggers another question in my mind: "How did they do that?!".  
I had one of these moments last week, looking at the push transition in [Facebook's Paper](http://capptivate.co/2014/02/04/paper/) app. It was a good chance to take a stab at iOS7 custom transitions.
<!-- More -->
###Learning custom transitions
I won't be covering in depth the custom transition APIs in this post, just what I needed to learn to reproduce the aforementioned effect. The main reason is because there are already good sources that cover this matter, I learned a lot from NSScreencast's ["iOS7 View controller Transitions"](http://nsscreencast.com/episodes/86-ios-7-view-controller-transitions) (the subscription fee is well worth all the quality content) and objc.io's ["View controller transitions"](http://www.objc.io/issue-5/view-controller-transitions.html). The former covers modal transitions, while the later coverage of push transitions was really helpful.

###UINavigationController's delegate
iOS7 introduces a new set of delegate methods in UINavigationController's protocol:

{% codeblock lang:objc %}
– navigationController:animationControllerForOperation:fromViewController:toViewController:
– navigationController:interactionControllerForAnimationController:
– navigationControllerPreferredInterfaceOrientationForPresentation:
– navigationControllerSupportedInterfaceOrientations:
{% endcodeblock %}

The first one is the most interesting of the bunch, it's called when the navigation controller is ready to animate to a new state (or for better terms, when it's ready to perform a new `UINavigationControllerOperation`) and expects in return a new object that conforms to the `UIViewControllerAnimatedTransitioning` protocol. It also has a reference to the view controller that we are transitioning from and a reference to the controller that we are transitioning to:
{% codeblock lang:objc %}
- (id<UIViewControllerAnimatedTransitioning>)
                   navigationController:(UINavigationController *)navigationController
        animationControllerForOperation:(UINavigationControllerOperation)operation
                     fromViewController:(UIViewController*)fromVC
                       toViewController:(UIViewController*)toVC;
{% endcodeblock %}

###UIViewControllerAnimatedTransitioning
Cool, we get notified when the navigation controller is ready to make its magic. Let's take a look at what kind of object it expects from us, by inspecting `UIViewControllerAnimatedTransitioning` protocol:
{% codeblock lang:objc %}
@protocol UIViewControllerAnimatedTransitioning <NSObject>

- (NSTimeInterval)transitionDuration:(id <UIViewControllerContextTransitioning>)transitionContext;
- (void)animateTransition:(id <UIViewControllerContextTransitioning>)transitionContext;

@optional

- (void)animationEnded:(BOOL) transitionCompleted;

@end
{% endcodeblock %}

Easy enough... one call for the animation's duration and one call to perform said animation.  
Before diving into the code we might want to lay out our animation graphically. Let's fire up Sketch and draw something:

{% img center /images/posts/2014-04-19/sketch.png 'Sketch' %}

Ok, so we have two view controllers, they both have table views, and we want to present the new cells with a slight delay each, while kicking the old cells to the other side, with the same delay.  
Let's break it down.  

*From* ViewController:  

- Get the visible cells in the table view
- Move each cell to the left by the screen width
- Fade out each cell
- On completion set the cells back to their position (we'll need them on _pop_)  

*To* ViewController:  

- Set the controller's frame to match the starting controller
- Move the controller's frame to the side
- Get the visible cells in the table view
- Push the cells to the right _with no animation_
- Move each cell to the left

That looks pretty simple, but there are couple of caveats.  
The first thing that I noticed is that the new cells are not available at first. The second issue was that if I moved the destination frame to match the source one right away, the source cells would disappear instantly.  
To solve these issues I had to use a little trick: use a `0` duration animation where I move the frame of the destination controller to the point `(1,0)`. This allows the destination controller to load its cells, and the source controller will still be visible.  

To sum it up (I'll include only the _push_ code):
{% codeblock lang:objc %}
// Move the destination in place
toVC.view.frame = source;
// And kick it aside
toVC.view.transform = CGAffineTransformMakeTranslation(SCREEN_WIDTH, 0);

// First step is required to trigger the load of the visible cells.
[UIView animateWithDuration:0 delay:0 options:UIViewAnimationOptionCurveEaseIn animations:nil completion:^(BOOL finished) {
    
    // Plain animation that moves the destination controller in place. Once it's done it will notify the transition context
        [toVC.view setTransform:CGAffineTransformMakeTranslation(1, 0)];
                    [UIView animateWithDuration:self.duration + self.maxDelay delay:0 options:UIViewAnimationOptionCurveEaseIn animations:^{
                            [toVC.view setTransform:CGAffineTransformIdentity];
                    } completion:^(BOOL finished) {
                            [transitionContext completeTransition:YES];
                    }];

    
    // Animates the cells of the starting view controller
    if ([fromVC respondsToSelector:@selector(visibleCells)]) {
        [[fromVC visibleCells] enumerateObjectsWithOptions:NSEnumerationReverse usingBlock:^(UITableViewCell *obj, NSUInteger idx, BOOL *stop) {
            NSTimeInterval delay = ((float)idx / (float)[[fromVC visibleCells] count]) * self.maxDelay;
            void (^animation)() = ^{
                [obj setTransform:CGAffineTransformMakeTranslation(-delta, 0)];
                [obj setAlpha:0];
            };
            void (^completion)(BOOL) = ^(BOOL finished){
                [obj setTransform:CGAffineTransformIdentity];
            };
            [UIView animateWithDuration:self.duration delay:delay options:UIViewAnimationOptionCurveEaseIn animations:animation completion:completion];
        }];
    }
    
    if ([toVC respondsToSelector:@selector(visibleCells)]) {
        [[toVC visibleCells] enumerateObjectsWithOptions:NSEnumerationReverse usingBlock:^(UITableViewCell *obj, NSUInteger idx, BOOL *stop) {
            NSTimeInterval delay = ((float)idx / (float)[[fromVC visibleCells] count]) * self.maxDelay;
            [obj setTransform:CGAffineTransformMakeTranslation(delta, 0)];
            void (^animation)() = ^{
                [obj setTransform:CGAffineTransformIdentity];
                [obj setAlpha:1];
            };
            [UIView animateWithDuration:self.duration delay:delay options:UIViewAnimationOptionCurveEaseIn animations:animation completion:nil];
        }];
    }
}];
{% endcodeblock %}
This will get the cell animating back and forth, but the effect is far from impressive, we need some further tinkering.

###Setting up the view controller
If you take a look at Paper, the cells move back and forth on the same background image, you can't see a clear cut from a view controller to the other. That's easy to achieve, we just need to configure our navigation controller like this:
{% codeblock lang:objc %}
[self.navigationController.view setBackgroundColor:[UIColor colorWithPatternImage:[UIImage imageNamed:@"background"]]];
[self.navigationController.navigationBar setBackgroundImage:[UIImage imageNamed:@"navbar"] forBarMetrics:UIBarMetricsDefault];
[self.view setBackgroundColor:[UIColor clearColor]];
[self.tableView setBackgroundColor:[UIColor clearColor]];
[self.navigationController.navigationBar setTintColor:[UIColor whiteColor]];
{% endcodeblock %}

And our cells need to be transparent too:

{% codeblock lang:objc %}
- (UITableViewCell*)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    // ... 
    [cell setBackgroundColor:[UIColor clearColor]];
    [cell setSelectionStyle:UITableViewCellSelectionStyleNone];
    // ...
}
{% endcodeblock %}

Now we can take the new animator for a spin, by returning it in the `navigationController:animationControllerForOperation:fromViewController:toViewController:` method.  

Here's the result:
{% img center /images/posts/2014-04-19/screenshot.gif 'Result' %}
It's not perfect, but it can be easily tweaked.

###The code
I released all the code described above as a pod. It includes both the push and pop operations, and its fairly customizable.  
You can find the source [here](https://github.com/andreamazz/AMWaveTransition).  

Until next time.
