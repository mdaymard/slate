---
title: API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - objective_c

search: true
---


# Overview #

A smartshow is a real-time HTML5 and JavaScript animation featuring images and audio assets (multimedia slideshow in JavaScript).  It can be saved and shared as a collection of a JSON parameter file and associated image and audio files, connected by standard URIs.  It can be played in the most common modern browsers (Chrome, Firefox, Microsoft, Safari) and inside native clients via a specially-prepared WebView.  This SDK also includes optimizations for native playback, including resizing, caching, and auto-orientation via an included local server.

# Installation #

## Minimum Requirements ##
* iOS 8.0
* XCode 8.0
* CocoaPods 1.0


## Add to your Project

```
...
source 'https://github.com/CocoaPods/Specs.git'
source 'https://bitbucket.org/avclsynchronoss/avcl-synchronoss-podspecs.git'
...
```



```
...
target '<your_project>' do
...
    pod 'SmartshowEngine', '= 2.2.0'
... 
end
...
```


In the Podfile of your project, ensure you don't have "use_frameworks!" and add the sources listed on the right.



Then, add the "SmartshowEngine" pod to your target, as described on the right.



In Terminal, go to your Podfile directory, and enter ```pod install```


If this is the first time you have installed the pods for this project, it will create a workspace project. From this point forward, you must open your project via the workspace to have the dependency for the smartshow SDK. 

### Permissions ###
The Smartshow SDK requires access to the photo storage and cleartext HTTP resource loading.

On iOS 10 and higher, accessing the photos storage requires you to add ```NSPhotoLibraryUsageDescription``` in your project's plist file.

In order to accept http loads, you also need to add the ```NSAppTransportSecurity``` dictionary in your plist file, and then create the ```NSAllowsArbitraryLoads``` item (set to YES) within this dictionary.

# Quick start #


``` objective_c
#import "ViewController.h"

#import "SmartshowEngine.h"

@interface ViewController () <SmartshowStateDelegate>

@property (nonatomic, strong) SmartshowService *smartshowService;
@property (nonatomic, strong) Smartshow *smartshow;
@property (nonatomic, strong) NSMutableArray *visualItems;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];

  // Once the user accepts the photo access permission, we load and play the smartshow
  [self promptPhotoAccessPermissions:^() {
  dispatch_async(dispatch_get_main_queue(), ^{
      [self loadAndPlay];
    });
  }];
}

- (void)loadAndPlay {
    
    // The smartshow service provides methods to interact with a smartshow (play, pause, etc..)
    self.smartshowService = [[SmartshowService alloc] initWithWebView:self.webview];
    [self.smartshowService addStateDelegate:self];
    
    self.smartshowService.regionalProvider = @"My regional provider";
    self.smartshowService.subscriptionId = @"My subscription ID";
    self.smartshowService.contextHandle = @"My context handle";
    
    // The loader notice is displayed when the smartshow is loading
    [self.smartshowService setLoaderVisible:YES withColor:@"123456"];
    [self.smartshowService setLoaderNotice:@"Loader notice"];
    
    // The smartshow object contains the visual items (photos or videos), and other parameters (theme, credits, etc)
    self.smartshow = [[Smartshow alloc] init];
    [self.smartshow setStartCredits:@"Start credits"];
    [self.smartshow setEndCredits:@"End credits"];
    
    self.visualItems = [NSMutableArray array];
    
    // This will load a remote image as an example (needs an internet connection)
    /*SmartshowVisualItem *item = [SmartshowVisualItem imageFromRemoteURL:[NSURL URLWithString:@"http://lorempixel.com/640/480/nature/2/"]];
     // The first image will have a caption
     [item setCaption:@"Remote image"];
     [self.visualItems addObject:item];*/
    
    // Loads a local image from the storage (requires permission)
    NSArray *localImages = [self getMediasWithNbMax:3];
    [self.visualItems addObjectsFromArray:localImages];
    
    // We set all the visual items to the smartshow
    [self.smartshow setVisualItems:self.visualItems];
    
    // Loads a remote audio and sets some track infos
    SmartshowAudio *audio = [[SmartshowAudio alloc] init];
    audio.url = [NSURL URLWithString:@"http://www.soundhelix.com/examples/mp3/SoundHelix-Song-1"];
    audio.artist = @"Artist";
    audio.title = @"Title";
    audio.duration = @20.0;
    
    // We set the audio to the smartshow
    [self.smartshow setAudio:audio];
    
    // Will set the default theme ("classic")
    SmartshowTheme *theme = [self.smartshowService getDefaultTheme];
    [self.smartshow setTheme:theme];
    
    /*
     * We start building the smartshow
     * This will start loading the webview, and once it's done we will launch the smartshow.
     */
    [self.smartshowService build:self.smartshow completion:^ {
        [self.smartshowService play];
    }];
}

// Engine state callback
- (void)onState:(SmartshowState)state {
    NSLog(@"Engine state : %u", state);
}

// Will retrieve photos from the device storage
- (NSMutableArray *)getMediasWithNbMax:(int)nbMax{
    PHFetchOptions *options = [[PHFetchOptions alloc] init];
    NSSortDescriptor *descriptor = [NSSortDescriptor sortDescriptorWithKey:@"creationDate" ascending:NO];
    options.sortDescriptors = @[descriptor];
    PHFetchResult *result = [PHAsset fetchAssetsWithMediaType:PHAssetMediaTypeImage options:options];
    
    int nbItems = (int)[result count];
    if(nbMax > nbItems)
        nbMax = nbItems;
    
    NSMutableArray *items = [NSMutableArray array];
    
    for (int i=0; i<nbMax; i++)
    {
        PHAsset *asset = [result objectAtIndex:i];
        SmartshowVisualItem *item = [SmartshowVisualItem imageFromLocalAsset:asset];
        [items addObject:item];
    }
    
    return items;
}

// Will prompt the permission pop up for accessing photos
- (void)promptPhotoAccessPermissions:(void (^)(void))completionBlock {
    [PHPhotoLibrary requestAuthorization:^(PHAuthorizationStatus status) {
        switch (status) {
            case PHAuthorizationStatusAuthorized: {
                static dispatch_once_t once;
                dispatch_once(&once, ^{
                    completionBlock();
                });
            }
                break;
            case PHAuthorizationStatusRestricted:
                NSLog(@"user restricted access");
                break;
            case PHAuthorizationStatusDenied:
                NSLog(@"user denied access");
                break;
            default:
                break;
        }
    }];
}
```

This example works with a simple view controller containing a ```UIWebView```. 

ViewController.m



# Create the UI #

``` objective_c
webView.allowsInlineMediaPlayback = YES;
webView.mediaPlaybackRequiresUserAction = NO;
webView.scrollView.scrollEnabled = NO;
```
``` objective_c
[[NSUserDefaults standardUserDefaults] registerDefaults:@{@"UserAgent": @"sharalike-iphone"}];
```

You first have to create the WebView programmatically, and the smartshow will play in this webview.
The webview needs special configuration, so the attributes of this webview will be modified by the SDK when you pass it as a parameter to the smartshow service.

The webview parameters listed on the right will be set by the SDK.

The service will also register a new user for http connections, as described on the right.



# Create the Smartshow Service #


``` objective_c
self.smartshowService = [[SmartshowService alloc] initWithWebView:self.webview];
```


The smartshow service is responsible for the life-cycle of a smartshow. It manages interactions (play, pause, etc..) and some UI controls (loading animations, resizing, etc.).

You must pass the webview as a parameter at instantiation:



## Loading Animation ##


``` objective_c
[self.smartshowService setLoaderVisible:YES withColor:@"#123456"];
```

The smartshow SDK includes an optional loading widget which displays the first image of the smartshow and animated loading icon.  If enabled, it is displayed during loading and buffering states. To enable it and customize the color of the loading animation, set these parameters before the smartshow is built:



``` objective_c
[self.smartshowService setLoaderNotice:@"Loader notice"];
```


You can also set the text displayed on the loader (notice text):


# Create a Smartshow #

Smartshow creation and playback is controlled through two main classes:  a convenience data object ```Smartshow``` for formatting the true JSON input and a service class ```SmartshowService``` which handles interactions.

## Assemble Data ##
First, assemble the smartshow data (Smartshow class) and method setX for parameter X (eg. setAudio for parameter Audio):

### Example 1 ###


``` objective_c
-(Smartshow *)createNewSmartshow {
    Smartshow *smartshow = [[Smartshow alloc] init];

    // Theme
    SmartshowTheme *theme = [self.smartshowService getDefaultTheme];
    [self.smartshow setTheme:theme];

    // Audio
    SmartshowAudio *audio = [[SmartshowAudio alloc] init];
    audio.url = [NSURL URLWithString:@"http://www.soundhelix.com/examples/mp3/SoundHelix-Song-1"];
    audio.artist = @"Artist";
    audio.title = @"Title";
    audio.duration = @20.0;
    [self.smartshow setAudio:audio];

    // Images
    NSMutableArray *images = [NSMutableArray array];
    SmartshowVisualItem *localImage = [SmartshowVisualItem imageFromRemoteURL:[NSURL URLWithString:@"http://lorempixel.com/640/480/nature/2/"]];
    [images addObject:localImage];
    [self.smartshow setVisualItems:images];

    return smartshow;
}
```


(Assumes smartshowService is initialized)



### Smartshow Required Parameters ###

<table>
<tr><td>Required</td><td>Parameters</td><td>Alternatives</td></tr>
<tr>
<td>Yes</td><td>Theme: SmartshowTheme*</td><td></td>
<tr><td></td><td colspan=2>(Must be a valid ID for a theme in the current JavaScript component)</td></tr>
<tr><td>Yes</td><td>SmartshowAudio: SmartshowAudio</td><td></td></tr>
<tr><td>Yes</td><td>Medias: NSArray* of SmartshowVisualItem* </td><td></td></tr>
</table>

### SmartshowAudio Required Parameters ###

<table>
<tr><td>Required</td><td>Parameters</td><td>Alternatives</td></tr>
<tr>
<td>Yes</td><td> Artist: NSString*<br /> Title: NSString*</td><td>Name: NSString*</td>
<tr>
<td></td>
<td colspan=2>
Accuracy is not required; human-readable strings are advised.
</td>
</tr>
<tr><td>Yes</td><td>Url: NSURL*</td><td></td></tr>
<tr>
<td></td>
<td colspan=2>
Local file path or distant URL.
</td>
</tr>
</table>

### SmartshowVisualItem Required Parameters ###

<table>
<tr><td>Required</td><td>Parameters</td><td>Alternatives</td></tr>
<tr>
<td>Yes</td><td>AssetURL: NSURL* </td><td>RemoteURL: NSURL*</td></tr>
<tr><td></td><td colspan=2> Use assetURL if the item is local, or remoteURL if the item is distant </td>
</tr>
<tr>
<td>Yes</td><td>AssetID: NSString* </td><td>RemoteID: NSString* </td></tr>
<tr><td></td><td colspan=2> Use assetID if the item is local, or remoteID if the item is distant</td>
</tr>
</table>

Local and remote images can be mixed within the same smartshow.

### Content Type and Protocol Restrictions ###

* Protocols: file, http, https
* Images: jpg or png
* Audio: mp3, m4a, or ogg
* Videos: webm or mp4, mov on iOS

## Interacting with the Smartshow ##

### Initiating Playback ###

Initiating playback is done in two steps, any time you create a new smartshow: 


``` objective_c
[self.smartshowService build:self.smartshow completion:^ {
        // smartshow is built
        // you can start it with : [self.smartshowService play];
    }
];
```


1. Load the raw data into the WebView and initiate asset loading



``` objective_c
[self.smartshowService play];
```


2.  Begin playback (**must** be done after building has completed 


Interaction with the smartshow is asynchronous.  In most cases, this is irrelevant due to the single-threaded nature of javascript.  For the initial load of data into the webview, however, it is important that all other actions wait until after this asynchronous step has completed.

#### Example 2 ####


``` objective_c
- (void)viewDidLoad {
    ...
    // This will provide us callbacks for the smartshow state
    [self.smartshowService addStateDelegate:self];
    ...
    [self.smartshowService build:self.smartshow completion:^ {
        [self.smartshowService play];
    }];
}

``` 

cf on the right




### Changing properties during playback ###


``` objective_c
[self.smartshow <changeParameter>]; visual items, audio, theme
[self.smartshowService refresh];
```

The sequence of animations and their timing is dependent on the specifics of the smartshow attributes. If you change the smartshow parameters while it is being played, you must refresh the service in order to take the modifications into account:


***Note that refreshing the smartshow might cause it to be rebuilt if the changes are significant, requiring load time. For example, changing images will rebuild the smartshow, but changing themes will not. If the smartshow was playing before refresh, it will play again after it passes the built state.***





#### Changing the Order ####

``` objective_c
[self.smartshow setVisualItems:visualItems];
[self.smartshowService refresh];
```

The order of items (```SmartshowVisualItem```) in the smartshow is exactly the order of the item array given to the ```Smartshow``` data object. Because the specific sequence of animations and their timing can be dependent on this order, you must change the order of the array and refresh the service if the smartshow is in playback:


#### Changing the Theme ####


``` objective_c
NSArray *themes = [self.smartshowService getThemes];
```

You can change a theme by setting a ```SmartshowTheme``` to a ```Smartshow``` object. 
As with the order of images, the sequence of animations and their timing is dependent on the specifics of the theme.  To change it while the smartshow is being played, you must change the property of the smartshow and refresh the smartshow service.




##### Retrieving Available Themes #####


``` objective_c
SmartshowTheme *theme = [self.smartshowService getDefaultTheme];
``` 

You can retrieve the list of available ```SmartshowTheme``` by calling this method:


And you can also get the default theme like this :


#### Changing the Audio ####
As with the order of images, the sequence of animations and their timing is dependent on the specifics of the ```SmartshowAudio``` object. To change audio while the smartshow is being played, you must change the property of the smartshow and refresh the smartshow service.

##### Audio Synchronization #####
Using the minimum parameters for audio will not produce a smartshow that is well timed to the music.  Setting several optional parameters will improve the overall experience of the smartshow by improving the timing of animations to match the music.  All parameters are optional and independent.

##### Synchronization Parameters #####

<table>
<tr><td>Parameter</td><td>Value Type</td></tr>
<tr><td>Duration</td><td>NSNumber *</td></tr>
<tr><td colspan=2>Duration of the track in seconds</td></tr>
<tr><td>Tempo</td><td>NSNumber *</td></tr>
<tr><td colspan=2>Beats per minute</td></tr>
<tr><td>Bars</td><td>NSArray *</td></tr>
<tr><td colspan=2>Array of timestamps in seconds for the bars</td></tr>
<tr><td>LoopDuration</td><td>NSNumber *</td></tr>
<tr><td colspan=2>Timestamp in seconds for looping the audio prior to the end of the file for a more seamless audio experience</td></tr>
<tr><td>Energy</td><td>NSNumber *</td></tr>
<tr><td colspan=2>Numerical value representing the perceived speed of the music (may not always be reflected by tempo)</td></tr>
</table>

### Adding Captions ###


``` objective_c
[item setCaption:@"Caption text"];
```

Each ```SmartshowVisualItem``` can take an additionalParameter ```caption``` , which is a short, descriptive ```NSString``` to be displayed with the VisualItem in the smartshow:


Because captions affect animation timing, you must refresh the smartshow service after changing them if the smartshow is already being played.

### Adding Credits ###
An optional text slide ```NSString *``` can be displayed at the beginning and/or end of the smartshow.  Set them to a ```Smartshow``` via ```setStartCredits:(NSString *)``` and ```setEndCredits:(NSString *)``` respectively.  Credits affect animation sequencing and timing, so you must refresh the smartshow service after changing them if the smartshow is already being played.

# Playback Controls #

* Pause ``` [smartshowService pause] ```
* Resume (from pause) ``` [smartshowService resume] ```
* Replay (for smartshows in any state of playback) ``` [smartshowService replay] ```
* Kill ``` [smartshowService kill:completionBlock] ```

(Completely destroys the internal state of the javascript component, releasing the maximum possible memory subject to the javascript garbage collector; the smartshow is irrecoverable after calling this.)

# Resizing #
By default, the smartshow will adjust to take the size of its parent container with no further action.  However, in the process of resizing, the view may be distorted in not-so-pretty ways.  If you wish to mask this, call ``` [smartshowService hideStage] ``` prior to resizing and ``` [smartshowService showStage] ``` after.

# Lifecycle Events #
The smartshow can communicate its current state through an event passing mechanism, if desired.  It is built on the delegate pattern, with the interface ```SmartshowStateDelegate``` and the method ``` (void)onState:(SmartshowState) ``` .
Here are the different possible states :  


* NOTINIT

(After creating the service, before the smartshow is built)

* BUILT

(When the smartshow is built)

* PLAYING

(Playback has started)

* PAUSED

(Smartshow has been paused)

* RESUMED

(Playback has resumed after buffering or pause)

* BUFFERING

(Smartshow has paused to wait for asset loading or internal state changes like resizing)

* ENDED

(Playback has reached the end of the smartshow)

* KILLED

(Playback has been killed, unusable)

# Saving Smartshows #


``` objective_c
Smartshow *savedSmartshow =  [smartshowService save];
NSData *jsonData = [savedSmartshow getSavedSmartshowData];
``` 

Smartshows can be saved to a JSON format for later replaying with the exact same sequence of animations and timing (much like a video).  Once the smartshow is built, you can retrieve a saved smartshow and its JSON data from the service like this:



``` objective_c
Smartshow *smartshowFromSavedData = [[Smartshow alloc] initFromSavedSmartshowData:jsonData];
``` 


You can then either store the smartshow object and use it later with ```SmartshowService```, or store the JSON data and use it later to recreate the saved smartshow with this method:


# Appendix A - Media Retrieval #

## Asking user photo access permission ##
``` objective_c
- (void)promptPhotoAccessPermissions:(void (^)(void))completionBlock {
    [PHPhotoLibrary requestAuthorization:^(PHAuthorizationStatus status) {
        switch (status) {
            case PHAuthorizationStatusAuthorized: {
                static dispatch_once_t once;
                dispatch_once(&once, ^{
                    completionBlock();
                });
            }
            break;
            case PHAuthorizationStatusRestricted:
                NSLog(@"user restricted access");
            break;
            case PHAuthorizationStatusDenied:
                NSLog(@"user denied access");
            break;
            default:
            break;
        }
    }];
}
``` 

## Retrieving Local Images ##
(most recent first)

``` objective_c
- (NSMutableArray *)getMediasWithNbMax:(int)nbMax{
    PHFetchOptions *options = [[PHFetchOptions alloc] init];
    NSSortDescriptor *descriptor = [NSSortDescriptor sortDescriptorWithKey:@"creationDate" ascending:NO];
    options.sortDescriptors = @[descriptor];
    PHFetchResult *result = [PHAsset fetchAssetsWithMediaType:PHAssetMediaTypeImage options:options];

    int nbItems = (int)[result count];
    if(nbMax > nbItems)
        nbMax = nbItems;

    NSMutableArray *items = [NSMutableArray array];

    for (int i=0; i<nbMax; i++)
    {
        PHAsset *asset = [result objectAtIndex:i];
        SmartshowVisualItem *item = [SmartshowVisualItem imageFromLocalAsset:asset];
        [items addObject:item];
    }

    return items;
}
```

## Bundled Dependencies ##

* [ReactiveCocoa:2.3](https://github.com/ReactiveCocoa/ReactiveCocoa)
* [RoutingHTTPServer:1.0.2](https://github.com/mattstevens/RoutingHTTPServer)
* [DFCache:1.3.3](https://github.com/kean/DFCache)

# Appendix B - Usage monitoring #

## Setting the Subscription ID ##

The SubscriptionId should be initialized with an anonymized unique value per subscriber (ex: a GUID). Setting this parameter will de-duplicate usage hits from the same user for the purposes of billing. If this parameter is unset (or set to the same value across all installations), each individual device installation will be considered separate for the purposes of billing

Call the following method after the smartshow has been attached and before creating any smartshow to identify your subscription unit in all future usage calls. Please note this id will not be retroactively applied.

``` objective_c
self.smartshowService.subscriptionId = @"My subscription ID";
```

## Optional Usage Parameters ##

Optional parameters that will be sent along with usage monitoring information; can be used to facilitate debugging or provide additional usage metrics.

``` objective_c
self.smartshowService.debugInfo = @"My Debug info";
```

Set arbitrary key-value string pairs for debugging; sent with all future usage logging information until it is edited or deleted. Please note that all non-string values will be ignored.

``` objective_c
self.smartshowService.contextHandle = @"My context handle";
```

Set an arbitrary string for easier identification of app context; sent with all future logging information until edited or deleted.


``` objective_c
self.smartshowService.regionalProvider = @"My regional provider";
```

Set an arbitrary string to identify an organizational subunit (for instance, a subsidiary, sub-customer, or customer-group); sent with all future logging information until edited or deleted.



# Appendix C - Manual Speed Control #

The Smartshow SDK includes the capability for a user to manually influence the preferred speed, to make a Smartshow either faster or slower. Due to factors related to audio synchronization and the specifics of animation, manual control may not always be taken into account and is best thought of as guidance only.

Rational numbers are preferred to maintain at least partial audio synchronization.

These guidance parameters are completely optional. Set them like shown on the right. 

``` objective_c
[self.smartshow setSpeedMultTop:2];
[self.smartshow setSpeedMultBottom:3];
```

Note that these change the expected duration (not the animation velocity) so numerator/denominator pairs resulting in values (0-1] will result in faster Smartshows and values > 1 will result in slower Smartshows.


<table>
<tr><td>Parameter</td><td>Value Type</td></tr>
<tr><td>speedMultTop</td><td>int</td></tr>
<tr><td colspan=2>The "numerator" of the speed modifier</td></tr>
<tr><td>speedMultBottom</td><td>int</td></tr>
<tr><td colspan=2>The "denominator" of the speed modifier</td></tr>
</table>
