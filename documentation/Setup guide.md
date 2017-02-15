# Setup Guide

Welcome to the Hub Framework setup guide! This guide aims to help you get set up with the framework in either a new or existing application.

Note that you only have to follow this guide when setting up the framework for the first time in your application. To learn how to use the framework in terms of building UIs and features using it - refer to the [getting started guide](https://spotify.github.io/HubFramework/getting-started-guide.html).

Before you proceed with this guide, it’s recommended that you read all of the programming guides, as to familiarize yourself with the various aspects of the framework. You can find links to all the programming guides [in the README](https://github.com/spotify/HubFramework#getting-started).

**Table of contents**

- [Introduction](#introduction)
- [Linking with `SystemConfiguration`](#linking-with-system-configuration)
- [Creating a `HUBManager` instance](#creating-a-hubmanager-instance)
- [Setting up the navigation system](#setting-up-the-navigation-system)
- [Further customization](#further-customization)

## Introduction

The Hub Framework is built to be easily integrated into apps of any size, with minimal impact on existing code. You don’t need to rewrite your app to start using it, and it has a very modular design that gives you a high degree of flexibility.

The recommendation is to start with as many defaults as you can, and then pick and choose what to customize after you’re set up.

## Linking with `SystemConfiguration`

The Hub Framework requires you to link with Apple's `SystemConfiguration` framework in order for its connectivity state resolver to work. To do so, simply add `SystemConfiguration` to your app target's "Linked Frameworks and Libraries" under the "General" tab in your project settings in Xcode.

## Creating a `HUBManager` instance

Each application using the Hub Framework will have one instance of the `HUBManager` class, which manages an instance of the framework. It’s recommended that you create this object early in your application’s lifecycle and keep it somewhere accessible - for example in your `AppDelegate`.

There are a few different ways to setup `HUBManager`, each providing different levels of customizability. The simplest way is to use the defaults, and just supply the following two parameters:

- `componentMargin`: What margin that should be used in between components.
- `componentFallbackBlock/Closure`: A block/closure that returns a component to use for fallback, in case a default one couldn't be matched.

For an example of how to set up the above, see the [demo app](https://github.com/spotify/HubFramework/tree/master/demo) which uses this way of setting up the Hub Framework.

There are also several optional parameters that you can choose to supply in case you want to further customize how the Hub Framework behaves. For more information about those, see the documentation for `HUBManager`, or the [Further customization section](#further-customization).

## Setting up the navigation system

The Hub Framework encourages the use of a view URI-based navigation system. This enables views to be completely decoupled, as they are no longer creating instances of each other but rather opening a URI, and having the system performing the actual navigation.

### Using navigation by opening URLs

By default, the framework calls `[UIApplication openURL:]` for any `target.URI` that a component’s model has, once that component was selected by the user (through a tap).

To make iOS route some/all of those URIs to your app, define what URL types and schemes that your app supports in your `Info.plist` file. Here's an example where we add support for the `hub-demo` scheme:

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLName</key>
        <string>com.spotify.hubFrameworkDemo</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>hub-demo</string>
        </array>
    </dict>
</array>
```

Once you've added the appropriate URL information to your `Info.plist`, any URL matching that information will be routed to your app delegate's `application:openURL:options:` method. This is an ideal place to perform the actual navigation.

To perform navigation, the first thing you’d want to do is to create a view controller to navigate to. To do that, use `HUBViewControllerFactory`, available on `HUBManager`. Then, push any view controller that was created onto your navigation controller, like this:

```objective-c
- (BOOL)application:(UIApplication *)app 
            openURL:(NSURL *)url 
            options:(NSDictionary<UIApplicationOpenURLOptionsKey,id> *)options
{
    id<HUBViewControllerFactory> factory = self.hubManager.viewControllerFactory;
    UIViewController *viewController = [factory createViewControllerForViewURI:url];
    
    if (viewController != nil) {
        [self.navigationController pushViewController:viewController animated:YES];
    }
}
```

### Using custom navigation

In case you either can’t or don’t want to implement navigation like above, you can supply a default `HUBActionHandler` when setting up `HUBManager`, which will give you full control to handle all types of actions that are performed from within the framework.

In this example we use a `HUBActionHandler` to create view controllers, without opening the view URI through the system:

```objective-c
@interface SPTDefaultActionHandler: NSObject <HUBActionHandler>

@property (nonatomic, weak) HUBManager *hubManager;

@end
```

```objective-c
@implementation SPTDefaultActionHandler

- (BOOL)handleActionWithContext:(id<HUBActionContext>)context
{
    // Don't prevent custom actions from being executed
    if (context.customActionIdentifier != nil) {
        return NO;
    }
    
    NSURL *viewURI = context.componentModel.target.URI;
    
    if (viewURI != nil) {
        id<HUBViewControllerFactory> factory = self.hubManager.viewControllerFactory;
        UIViewController *viewController = [factory createViewControllerForViewURI:url];
    
        if (viewController != nil) {
            [self.navigationController pushViewController:viewController animated:YES];
        }
    }
    
    return YES;
}

@end
```

The above is only one of many possible implementations. You could, for example, also use actions (`HUBAction`) to navigate. However, it’s really recommended to go with the default way of opening URLs through the system APIs. That solution also gives you the additional benefit of making it really easy to add deep linking support to you application.

## Further customization

You’re now ready to start building UIs using the Hub Framework. Head over to the [getting started guide](https://spotify.github.io/HubFramework/getting-started-guide.html) to learn how to register features and how to build content operations & components.

In case you need it, you can also continue customizing the framework, adding support for additional features.

### Building a custom component layout manager

In case you want more fine-grained control over how components are laid out, you can supply your own layout manager by conforming to the `HUBComponentLayoutManager` protocol. A custom layout manager enables you to calculate margins for your components, given a set of layout traits.

For more information about how layout works for components, see the [Layout programming guide](https://spotify.github.io/HubFramework/layout-programming-guide.html).

### Adding icon support

The Hub Framework enables you to easily convert string representations of icon identifiers to `UIImages` in components, through the `HUBIcon` API. However, before this API can be used, you need to implement a `HUBIconImageResolver`, which the framework will use to perform this conversion. The image resolver is then passed as `iconImageResolver` when setting up `HUBManager`.

Here is an example implementation:

```objective-c
@interface SPTIconImageResolver: NSObject <HUBIconImageResolver>

@end
```

```objective-c
@implementation SPTIconImageResolver

- (nullable UIImage *)imageForComponentIconWithIdentifier:(NSString *)iconIdentifier
                                                     size:(CGSize)size
                                                    color:(UIColor *)color
{
    NSString *imageName = [@"componentImage-" stringByAppendingString:iconIdentifier];
    UIImage *image = [UIImage imageNamed:imageName];
    return [image spt_colorizeWithColor:color];
}

- (nullable UIImage *)imageForPlaceholderIconWithIdentifier:(NSString *)iconIdentifier
                                                       size:(CGSize)size
                                                      color:(UIColor *)color
{
    NSString *imageName = [@"placeholderImage-" stringByAppendingString:iconIdentifier];
    UIImage *image = [UIImage imageNamed:imageName];
    return [image spt_colorizeWithColor:color];
}

@end
```

### Defining a global reload policy

A `HUBContentReloadPolicy` is used to determine whether a given view should be reloaded when it is about to re-appear on the screen. They can be defined on a per-feature basis, but you also have the option to supply a global one that is used in case a feature didn’t supply its own. You pass a default content reload policy as `defaultContentReloadPolicy` when setting up `HUBManager`.

Here is an example reload policy implementation:

```objective-c
@interface SPTDefaultReloadPolicy: NSObject <HUBContentReloadPolicy>

@end
```

```objective-c
@implementation SPTDefaultReloadPolicy

- (BOOL)shouldReloadContentForViewURI:(NSURL *)viewURI currentViewModel:(id<HUBViewModel>)currentViewModel
{
    // This will make a view reload once 1000 seconds have passed
    NSTimeInterval timeSinceReload = [[NSDate date] timeIntervalSinceDate:currentViewModel.buildDate];
    return timeSinceReload > 1000;
}

@end
```

### Prepended and appended content operations

You can also define system-wide content operations that are either prepended or appended to all views’ content loading chains.

To use this functionality, pass a `HUBContentOperationFactory` as either `prependedContentOperationFactory` or `appendedContentOperationFactory` when setting up `HUBManager`.

To learn more about the content loading chain and content operations, refer to the [content programming guide](https://spotify.github.io/HubFramework/content-programming-guide.html).
