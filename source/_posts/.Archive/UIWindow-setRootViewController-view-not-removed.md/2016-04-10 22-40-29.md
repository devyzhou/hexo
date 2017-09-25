---
title: '[UIWindow setRootViewController] 导致的内存泄漏'
date: 2016-04-04 18:10:13
tags: [iOS]
---

在做退出登录的时候，直接修改 `window.rootViewController`，来导航到登录界面，发生了内存泄漏。<!--more-->

```objc
//"AppDelegate.m"
- (void)setupLoginViewController{
    UIStoryboard *storyboard = [UIStoryboard storyboardWithName:@"Main" bundle:nil];
    UINavigationController *ZYNav = [storyboard 	    instantiateViewControllerWithIdentifier:@"ZYNavigationController"];
    [UIView transitionWithView:self.window duration:0.5 options:UIViewAnimationOptionTransitionCrossDissolve animations:^{
        [self.window setRootViewController:ZYNav];
    } completion:nil];
}
```

但是在调试界面的时候发现，设置界面的 `view` 竟然还存在，这样在用户多次操作登录－退出登录时会导致 `view` 的无限叠加。

![](http://7xlcff.com1.z0.glb.clouddn.com/16-4-4/36242137.jpg)

搜索了一下stack overflow 发现也有同样的问题存在，[leaking-views-when-changing-rootviewcontroller-inside-transitionwithview](http://stackoverflow.com/questions/26763020/leaking-views-when-changing-rootviewcontroller-inside-transitionwithview) ，解决的方法不那么优雅。

```objc
- (void)setupLoginViewController{
    UIViewController *previousRootViewController = self.window.rootViewController;

    UIStoryboard *storyboard = [UIStoryboard storyboardWithName:@"Main" bundle:nil];
    UINavigationController *ZYNav = [storyboard instantiateViewControllerWithIdentifier:@"ZYNavigationController"];

    [UIView transitionWithView:self.window duration:0.5 options:UIViewAnimationOptionTransitionCrossDissolve animations:^{
        [self.window setRootViewController:ZYNav];
    } completion:nil];

    // The presenting view controllers view doesn't get removed from the window as its currently transistioning and presenting a view controller
    for (UIView *subview in self.window.subviews) {
        if ([subview isKindOfClass:NSClassFromString(@"UITransitionView")]) {
            [subview removeFromSuperview];
        }
    }
    // Allow the view controller to be deallocated
    [previousRootViewController dismissViewControllerAnimated:NO completion:^{
        // Remove the root view in case its still showing
        [previousRootViewController.view removeFromSuperview];
    }];
}
```

强行移除依然存在的 `UITransitionView`  。

Over