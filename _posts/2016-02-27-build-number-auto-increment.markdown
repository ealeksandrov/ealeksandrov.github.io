---
layout: post
title: "Build number auto-increment"
date: 2016-02-27 14:00:00 +0400
---

Many developers are using script to increment build number after each deployment. I don't like this approach because it provides no info about a builds other than ordering. Since Apple requires this to be increasing number - commits hash can't be used. But it is possible to use number of commits in current branch - this allows to quickly find a commit for any build deployed.

Also my approach only changes `plist` in build results, so nothing appear in `git status` for woking directory.

<!-- more -->

## Autoupdate script

Add script to new build phase: `ProjectName > TargetName > Build Phases > New Run Script Phase`.

Script can be just dropped as a text here, but I prefer to store it separately, so I am adding:

{% highlight shell %}
sh Scripts/autoupdate-revision.sh
{% endhighlight %}

![Adding Build Phase](/static/article-build-number/01.png)

And here is shell script:

{% highlight shell %}
#!/bin/sh
# autoupdate-revision.sh
#
# Evgeny Aleksandrov

git=`sh /etc/profile; which git`
branch=`${git} rev-parse --abbrev-ref HEAD`
commits_count=`${git} rev-list ${branch} | wc -l  | tr -d ' '`

filepath="${BUILT_PRODUCTS_DIR}/${INFOPLIST_PATH}"

echo "Updating ${filepath}"
echo "Current version build ${commits_count}"
/usr/libexec/PlistBuddy -c "Set :CFBundleVersion ${commits_count}" "${filepath}"
{% endhighlight %}

## Accessing app version and build number in runtime

For every my project I am adding short log of current bundle id + version + build number in `AppDelegate`.

{% highlight objc %}
NSString *bundleId = [[[NSBundle mainBundle] infoDictionary] objectForKey:@"CFBundleIdentifier"];
NSString *bundleShortVersion = [[[NSBundle mainBundle] infoDictionary] objectForKey:@"CFBundleShortVersionString"];
NSString *bundleVersion = [[[NSBundle mainBundle] infoDictionary] objectForKey:@"CFBundleVersion"];

NSLog(@"Starting %@ v%@ (%@)", bundleId, bundleShortVersion, bundleVersion);
{% endhighlight %}

{% highlight swift %}
lazy var bundleId: String = NSBundle.mainBundle().infoDictionary?["CFBundleIdentifier"] as! String
lazy var bundleShortVersion: String = NSBundle.mainBundle().infoDictionary?["CFBundleShortVersionString"] as! String
lazy var bundleVersion: String = NSBundle.mainBundle().infoDictionary?["CFBundleVersion"] as! String

print("Starting \(self.bundleId) v\(self.bundleShortVersion) (\(self.bundleVersion))")
{% endhighlight %}

## Finding commit for build number

If you want to get commit hash from build number, add this command to `[alias]` section of your `~/.gitconfig`:

{% highlight shell %}
show-rev-number = !sh -c 'git rev-list --reverse HEAD | nl | awk \"{ if(\\$1 == "$0") { print \\$2 }}\"'
{% endhighlight %}

And then use it in your working directory (don't forget to be on your correct release branch):

{% highlight shell %}
> git show-rev-number 13
3dd5cee5f623da80e3cb8e6417e8a31197ae75f6
{% endhighlight %}
