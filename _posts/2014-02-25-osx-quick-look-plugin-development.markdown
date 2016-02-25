---
layout: post
title: "OSX Quick Look plugin development"
date: 2014-02-25 16:40:00 +0400
---

Quick Look — OS X service that creates thumbnails and previews for files in Finder. It supports a number of standard file types, for others there are QL plugins — custom thumbnails and preview generators. They have .qlgenerator extension and can be placed in `~/Library/QuickLook` or `/Library/QuickLook`.

In this article, I will tell about main stages of creating custom QL plugins.

<!-- more -->

I am iOS and OSX app developer. First time I ever encountered QL plugin is when I saw Craig Hockenberry's [Provisioning](https://github.com/chockenberry/Provisioning) preview generator for .mobileprovision files (`.mobileprovision`/`.provisionprofile` - profile containing certificates, device ids and other parameters needed for iOS & OSX app deployment).

This is how profile folder looks without any custom Quick Look plugins:

![Default appearance](/static/article-ql/01.png)

Manual profile selection may be sometimes required. In TestFlight autouploading script, for example. Choosing right profile from this pile of identical icons is quite a problem.

As a first solution, I started using the open-source [Provisioning](https://github.com/chockenberry/Provisioning) project, then closed-source (but more beautiful and detailed) [ipaql](http://ipaql.com/). Need in own open solution arisen after the `ipaql` maintainer added OS X Mavericks compatibility only six months after the release and still repairing apps' icons generation.

Here's what I got - [ProvisionQL](https://github.com/ealeksandrov/ProvisionQL). 
Supported file types for thumbnails and previews generation:

* `ipa` - iOS packaged application (both Xcode and AppStore supported)
* `app` - iOS application bundle 
* `mobileprovision` - iOS provisioning profile 
* `provisionprofile` - OSX provisioning profile

And here is the result of QL plugin work:

![ProvisionQL appearance](/static/article-ql/02.png)

## Project settings

Create a new project in Xcode: `File > New > Project… OS X > System plug-in > Quick Look Plug-in`. In base template go straight to `Info.plist`:

![Info.plist settings](/static/article-ql/03.png)

Unfold `CFBundleDocumentTypes` and add desired file types in `LSItemContentTypes` array. If you want to generate small icons in lists and tables, change `QLThumbnailMinimumSize` from 17 to 16. Note `QLPreviewHeight` and `QLPreviewWidth` properties — they are used when generator takes too long to generate a preview. In my case – with `ipa` – extraction of multiple files from a zip file is required, which is quite a long task (0,06 – 0,12 s), so system always uses these plist properties. If your generator will generate a file quickly — qlmanage will use dimensions of an image or a HTML you return.

Next, if you prefer obj-c and Foundation classes — rename `GenerateThumbnailForURL.c` and `GeneratePreviewForURL.c` to `GenerateThumbnailForURL.m` and `GeneratePreviewForURL.m` and add Foundation headers there:

{% highlight objc %}
#import <Foundation/Foundation.h>
#import <Cocoa/Cocoa.h>
{% endhighlight %}

I need to generate both thumbnails (GenerateThumbnailForURL) and preview (GeneratePreviewForURL), so I defined common imports and functions in `Shared.h/m`. Here is my `Shared.h`:

{% highlight objc %}
#include <CoreFoundation/CoreFoundation.h>
#include <CoreServices/CoreServices.h>
#include <QuickLook/QuickLook.h>

#import <Foundation/Foundation.h>
#import <Cocoa/Cocoa.h>
#import <Security/Security.h>

#import <NSBezierPath+IOS7RoundedRect.h>

static NSString * const kPluginBundleId = @"com.FerretSyndicate.ProvisionQL";
static NSString * const kDataType_ipa               = @"com.apple.itunes.ipa";
static NSString * const kDataType_app               = @"com.apple.application-bundle";
static NSString * const kDataType_ios_provision     = @"com.apple.mobileprovision";
static NSString * const kDataType_ios_provision_old = @"com.apple.iphone.mobileprovision";
static NSString * const kDataType_osx_provision     = @"com.apple.provisionprofile";

#define SIGNED_CODE 0

NSImage *roundCorners(NSImage *image);
NSImage *imageFromApp(NSURL *URL, NSString *dataType, NSString *fileName);
NSString *mainIconNameForApp(NSDictionary *appPropertyList);
int expirationStatus(NSDate *date, NSCalendar *calendar);
{% endhighlight %}

Final ProvisionQL project structure:

![Project structure](/static/article-ql/04.png)

`NSBezierPath+IOS7RoundedRect` — function for masking icon with iOS7-style rounded corners. `Install.sh` — autoinstall script for generator:

{% highlight sh %}
#!/bin/sh

PRODUCT="${PRODUCT_NAME}.qlgenerator"
QL_PATH=~/Library/QuickLook/

rm -rf "$QL_PATH/$PRODUCT"
test -d "$QL_PATH" || mkdir -p "$QL_PATH" && cp -R "$BUILT_PRODUCTS_DIR/$PRODUCT" "$QL_PATH"
qlmanage -r

echo "$PRODUCT installed in $QL_PATH"
{% endhighlight %}

To run it, go to Target settings, select `Editor > Add Build Phase > Add Run Script Build Phase` and enter the path to the script in the project folder:

![Adding script build phase](/static/article-ql/05.png)

You may need to debug the plugin. Because it is not an executable itself, you must go to project scheme settings – Edit Scheme... > Run > Info > Executable > Other > press `Cmd + Shft + G` > `/usr/bin/` > Go > `qlmanage`:

![Adding executable to run](/static/article-ql/06.png)

Then, in the Arguments tab, specify the arguments: starting with `-t` (for thumbnails debugging) or `-p` (for preview debugging), following with full path to the test file (in example case I'm testing thumbnail generation for `.ipa`):

![Adding executable args](/static/article-ql/07.png)

## Thumbnails generation

In this example I will show how to display a dummy icon ([defaultIcon.png](https://raw2.github.com/ealeksandrov/ProvisionQL/master/ProvisionQL/Resources/defaultIcon.png)). In ProvisionQL, you can check out an implementation for `ipa` file extraction, as well as displaying the number of devices and the status of the provision.

This is `GenerateThumbnailForURL.m`:

{% highlight objc %}
#import "Shared.h"

OSStatus GenerateThumbnailForURL(void *thisInterface, QLThumbnailRequestRef thumbnail, CFURLRef url, CFStringRef contentTypeUTI, CFDictionaryRef options, CGSize maxSize);
void CancelThumbnailGeneration(void *thisInterface, QLThumbnailRequestRef thumbnail);

/* -----------------------------------------------------------------------------
    Generate a thumbnail for file

   This function's job is to create thumbnail for designated file as fast as 
   possible
   ----------------------------------------------------------------------------- */

OSStatus GenerateThumbnailForURL(void *thisInterface, QLThumbnailRequestRef thumbnail, CFURLRef url, CFStringRef contentTypeUTI, CFDictionaryRef options, CGSize maxSize) {
    @autoreleasepool {
        NSString *dataType = (__bridge NSString *)contentTypeUTI;
        NSImage *appIcon;
        
        if([dataType isEqualToString:kDataType_app] || [dataType isEqualToString:kDataType_ipa]) {
            NSURL *iconURL = [[NSBundle bundleWithIdentifier:kPluginBundleId] URLForResource:@"defaultIcon" withExtension:@"png"];
            appIcon = [[NSImage alloc] initWithContentsOfURL:iconURL];
        } else {
            return noErr;
        }
        
        if (QLThumbnailRequestIsCancelled(thumbnail)) {
            return noErr;
        }
        
        NSSize canvasSize = appIcon.size;
        NSRect renderRect = NSMakeRect(0.0, 0.0, appIcon.size.width, appIcon.size.height);
        
        CGContextRef _context = QLThumbnailRequestCreateContext(thumbnail, canvasSize, false, NULL);
        if (_context) {
            NSGraphicsContext* _graphicsContext = [NSGraphicsContext graphicsContextWithGraphicsPort:(void *)_context flipped:NO];
            
            [NSGraphicsContext setCurrentContext:_graphicsContext];
            [appIcon drawInRect:renderRect];
            //draw anything you want here

            
            QLThumbnailRequestFlushContext(thumbnail, _context);
            CFRelease(_context);
        }
    }
    
    return noErr;
}

void CancelThumbnailGeneration(void *thisInterface, QLThumbnailRequestRef thumbnail) {
    // Implement only if supported
}
{% endhighlight %}

Note some things:

* you can't use `NSImage imageNamed:` - this method will look for the resource in qlmanage's (executable file) bundle, not in our plugin 
* check `QLThumbnailRequestIsCancelled(thumbnail)` before any time-consuming operation

## Preview generation

In the example, we will fill and return an HTML file as a preview. First, prepare your `template.html` (you can also include styles there).

{% highlight html %}
<!DOCTYPE html>
<html lang="en">
    <body>
        <div>
            <h1>App info</h1>
            Name: <strong>__CFBundleDisplayName__</strong><br />
            Version: __CFBundleShortVersionString__ (__CFBundleVersion__)<br />
            BundleId: __CFBundleIdentifier__<br />
        </div>
    </body>
</html>
{% endhighlight %}

Everything inside the `__KEY__` will be filled with the generator.
Here is our final `GeneratePreviewForURL.m`:

{% highlight objc %}
#import "Shared.h"

OSStatus GeneratePreviewForURL(void *thisInterface, QLPreviewRequestRef preview, CFURLRef url, CFStringRef contentTypeUTI, CFDictionaryRef options);
void CancelPreviewGeneration(void *thisInterface, QLPreviewRequestRef preview);

/* -----------------------------------------------------------------------------
 Generate a preview for file
 
 This
 function's job is to create preview for designated file
 ----------------------------------------------------------------------------- */

OSStatus GeneratePreviewForURL(void *thisInterface, QLPreviewRequestRef preview, CFURLRef url, CFStringRef contentTypeUTI, CFDictionaryRef options) {
    @autoreleasepool {
        NSURL *URL = (__bridge NSURL *)url;
        NSString *dataType = (__bridge NSString *)contentTypeUTI;
        NSData *appPlist = nil;
        
        if([dataType isEqualToString:kDataType_app]) {
            // get the embedded plist for the iOS app
            appPlist = [NSData dataWithContentsOfURL:[URL URLByAppendingPathComponent:@"Info.plist"]];
        } else if([dataType isEqualToString:kDataType_ipa]) {
            // get the embedded plist from an app archive using: unzip -p <URL> <files to unzip> (piped to standart output)
            NSTask *unzipTask = [NSTask new];
            [unzipTask setLaunchPath:@"/usr/bin/unzip"];
            [unzipTask setStandardOutput:[NSPipe pipe]];
            [unzipTask setArguments:@[@"-p", [URL path], @"Payload/*.app/Info.plist"]];
            [unzipTask launch];
            [unzipTask waitUntilExit];
            
            appPlist = [[[unzipTask standardOutput] fileHandleForReading] readDataToEndOfFile];
        } else {
            return noErr;
        }
        
        if(QLPreviewRequestIsCancelled(preview)) {
            return noErr;
        }

        NSMutableDictionary *synthesizedInfo = [NSMutableDictionary dictionary];
        NSURL *htmlURL = [[NSBundle bundleWithIdentifier:kPluginBundleId] URLForResource:@"template" withExtension:@"html"];
        NSMutableString *html = [NSMutableString stringWithContentsOfURL:htmlURL encoding:NSUTF8StringEncoding error:NULL];
        
        NSDictionary *appPropertyList = [NSPropertyListSerialization propertyListWithData:appPlist options:0 format:NULL error:NULL];
        [synthesizedInfo setObject:[appPropertyList objectForKey:@"CFBundleDisplayName"] forKey:@"CFBundleDisplayName"];
        [synthesizedInfo setObject:[appPropertyList objectForKey:@"CFBundleIdentifier"] forKey:@"CFBundleIdentifier"];
        [synthesizedInfo setObject:[appPropertyList objectForKey:@"CFBundleShortVersionString"] forKey:@"CFBundleShortVersionString"];
        [synthesizedInfo setObject:[appPropertyList objectForKey:@"CFBundleVersion"] forKey:@"CFBundleVersion"];
        
        for (NSString *key in [synthesizedInfo allKeys]) {
            NSString *replacementValue = [synthesizedInfo objectForKey:key];
            NSString *replacementToken = [NSString stringWithFormat:@"__%@__", key];
            [html replaceOccurrencesOfString:replacementToken withString:replacementValue options:0 range:NSMakeRange(0, [html length])];
        }
        
        NSDictionary *properties = @{ // properties for the HTML data
                                     (__bridge NSString *)kQLPreviewPropertyTextEncodingNameKey : @"UTF-8",
                                     (__bridge NSString *)kQLPreviewPropertyMIMETypeKey : @"text/html" };
        
        QLPreviewRequestSetDataRepresentation(preview, (__bridge CFDataRef)[html dataUsingEncoding:NSUTF8StringEncoding], kUTTypeHTML, (__bridge CFDictionaryRef)properties);
    }
    
    return noErr;
}

void CancelPreviewGeneration(void *thisInterface, QLPreviewRequestRef preview) {
    // Implement only if supported
}
{% endhighlight %}

As you can see, at first we open `Info.plist` (or extract it from the archive if needed), then process some data from it in the `synthesizedInfo`. All keys of the `synthesizedInfo` are filled respectively in keys loaded from `template.html`. This string is returned to the `qlmanage` alongside with parameters describing the return type of the data as an HTML.

## Conclusion

Following this guide, you can quickly create a plugin for quick preview and icons generating for your proprietary format or to any common format, which is not handled by the system.

Regarding [ProvisionQL](https://github.com/ealeksandrov/ProvisionQL) - I am opened to any suggestions and pull requests that can improve functionality within the plugin scope.
