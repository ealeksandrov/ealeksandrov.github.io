---
layout: post
title: "Xcode project configuration"
date: 2015-02-27 20:10:00 +0400
---

In every iOS or OS X app there are many configurations like API base url, keychain service name, keys for various 3rd party services. Some of them are different for Debug and Production, which leads to numerous `#ifdef`'s across all the project and so on.

Here I will show quick and convenient way to manage project compile-time dependencies in `.xcconfig`'s.

<!-- more -->

## Configs structure

Add new configs in Xcode: `File > New > File… > Other > Configuration Settings File`.

![Configs folder in Xcode](/static/article-config/01.png)

Most configuration values will be stored in `shared.xcconfig`:

{% highlight objc %}
WF_APPLICATION_ID = @"827118463"

WF_BUNDLE_VERSION = 2.0.0
WF_BUNDLE_IDENTIFIER = com.company.ExampleApp
WF_BUNDLE_DISPLAYNAME = Winterfell

WF_API_BASEPROTOCOL = @"https"
WF_API_BASEURL = @"api.example.com"
WF_API_VERSION = @"v1"

WF_KEYCHAIN_SERVICE = @"WinterfellService"

WF_FLURRY_KEY = @"Y6V7V868T7B42QPFTBVZ"

WF_ALLOW_INVALID_SSL = NO
{% endhighlight %}

And only ones that different to build configuration will be overridden, like in `staging.xcconfig`:

{% highlight objc %}
#include "shared.xcconfig"

WF_API_BASEURL = @"api.example-staging.com"
WF_ALLOW_INVALID_SSL = YES

#include "generator.xcconfig"
{% endhighlight %}

Translation in preprocessor macro will happen in `generator.xcconfig`:

{% highlight objc %}
// This is the config which will generate GCC_Preprocessor macroses. It is icluded from debug/release/production configs
GCC_PREPROCESSOR_DEFINITIONS = $(inherited) WF_APPLICATION_ID='$(WF_APPLICATION_ID)' WF_BUNDLE_VERSION='$(WF_BUNDLE_VERSION)' WF_BUNDLE_IDENTIFIER='$(WF_BUNDLE_IDENTIFIER)' WF_BUNDLE_DISPLAYNAME='$(WF_BUNDLE_DISPLAYNAME)' WF_API_BASEPROTOCOL='$(WF_API_BASEPROTOCOL)' WF_API_BASEURL='$(WF_API_BASEURL)' WF_API_VERSION='$(WF_API_VERSION)' WF_FLURRY_KEY='$(WF_FLURRY_KEY)' WF_KEYCHAIN_SERVICE='$(WF_KEYCHAIN_SERVICE)' WF_ALLOW_INVALID_SSL='$(WF_ALLOW_INVALID_SSL)'
{% endhighlight %}

You can see, for each value we adding `VALUE='$(VALUE)'` to `GCC_PREPROCESSOR_DEFINITIONS` string.

## Swift

Bridging header is needed to access preprocessor definitions from Swift code. It can be empty if you don't use any obj-c sources.

## Project settings

![Configurations in Xcode project settings](/static/article-config/02.png)

Open `Project > Info > Configurations` and select there corresponding configs. I have CocoaPods project, so I am adding this configurations on project-level, not target-level.

## Usage

You can use macros everywhere in your code, you don't have to import anything:

{% highlight objc %}
static NSString * const kPromoKeychainServiceString = WF_KEYCHAIN_SERVICE;
static NSString * const kBaseURLString = WF_API_BASEPROTOCOL @"://" WF_API_BASEURL @"/" WF_API_VERSION @"/";

self.manager.securityPolicy.allowInvalidCertificates = WF_ALLOW_INVALID_SSL;
{% endhighlight %}

Also, I like to move some values from Info.plist (like bundle version and display name), so all my configs stored in one place:

![Configurations in Info.plist](/static/article-config/03.png)

## Acknowledgments

Thanks to [Paul Taykalo](https://github.com/PaulTaykalo) for sharing this way of managing configurations.
