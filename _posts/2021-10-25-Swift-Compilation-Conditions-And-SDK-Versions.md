---
layout: post
title:  "Swift Active Compilation Conditions and SDK Versions"
permalink: swift-compilation-conditions
---

### Objective-C's Preprocessor Macros and Conditional Compilation
Availability checks can be handy for issues where runtime behavior needs to change, but sometimes you're using an API that simply doesn't exist or is materially different in an old SDK. Compiling new SDK APIs with an old SDK won't compile, even behind an `@available` check. In Objective-C, we can use preprocessor macros to decide which piece of code to compile based on a macro defined by Apple about SDK availability.

```obj-c
#import "AvailabilityVersions.h"

// If __MAC_12_0 is available, we're at least compiling with the macOS Monterey 12.0 SDKs
#ifdef __MAC_12_0
// Do something that requires the Monterey-SDK to compile
#else
// Handle this scenario in the pre-Monterey way
#endif
```

This approach allows the code to compile as expected with and without the new SDK, allowing production builds to use stable versions of Xcode and SDKs while developers can move forward with beta support for new OS releases all with the same code.

### Swift and Conditional Compilation
Swift has very limited preprocessing capabilities[^1] which can complicate using preprocessor macros for deciding what code compiles in what scenarios. Swift can't simply check the AvailabilityVersions.h declarations and conditionally compile on the fly. Swift does however allow conditional compilation using the `Active Compilation Conditions` build setting in Xcode or the `SWIFT_ACTIVE_COMPILATION_CONDITIONS` macro in an xcconfig file.

By combining this with some advanced xcconfig features like conditional assignment and variable substitution[^2] along with `SWIFT_ACTIVE_COMPILATION_CONDITIONS` and Xcode's provided `SDK_VERSION_MAJOR`, we can add our own compilation condition only when compiling against a specific SDK version.

```
// Define our macOS 12.0 SDK compilation conditions
MY_SWIFT_ACTIVE_COMPILATION_CONDITIONS_FOR_SDK_120000[sdk=macos*] = MAC_OS_VERSION_12_0_SDK_AVAILABLE

// Conditionally include our compilation conditions based on the SDK
SWIFT_ACTIVE_COMPILATION_CONDITIONS= $(inherited) $(MY_SWIFT_ACTIVE_COMPILATION_CONDITIONS_FOR_SDK_$(SDK_VERSION_MAJOR))
```

When compiling with these settings in an xcconfig, we can now conditionally compile in Swift based on the SDK we compiled against.

```swift
// If MAC_OS_VERSION_12_0_SDK_AVAILABLE is an active compilation condition, we compiled against the macOS 12.0 SDK
#if MAC_OS_VERSION_12_0_SDK_AVAILABLE
// Do something that requires the Monterey-SDK to compile
#else
// Handle this scenario in the pre-Monterey way
#endif
```

*Note: There is one difference here - when this code inevitably compiles against a future macOS 13.0 SDK, we will lose this compilation condition. These types of macros should be temporary staging while migrating from one SDK to another and not long-lived. If you do find yourself needing to extend this type of compilation condition across SDKs, you could manually include the macOS 12.0 SDK macros in your definition for macOS 13.0 SDK active compilation conditions.*
```
// An eventual 13.0 SDK compilation conditions could include the 12.0 SDK conditions if necessary
MY_SWIFT_ACTIVE_COMPILATION_CONDITIONS_FOR_SDK_130000[sdk=macos*] = MAC_OS_VERSION_13_0_SDK_AVAILABLE $(MY_SWIFT_ACTIVE_COMPILATION_CONDITIONS_FOR_SDK_120000)
```

### A Practical Example

In the macOS Monterey 12.0 SDK, AVFoundation's `-[AVCaptureAudioDataOutput recommendedAudioSettingsForAssetWriterWithOutputFileType:]` API was improved to provide types for the dictionary returned by the API.[^3]

#### Before
```obj-c
- (NSDictionary *)recommendedAudioSettingsForAssetWriterWithOutputFileType:(AVFileType)outputFileType
```

#### After
```obj-c
- (NSDictionary<NSString *,id> *)recommendedAudioSettingsForAssetWriterWithOutputFileType:(AVFileType)outputFileType
```

If some code was relying on these types that were documented but not specified in code, it might look something like this:
```swift
guard let recommendedAudioSettings = avCaptureAudioDataOutput.recommendedAudioSettingsForAssetWriter(...) as! Dictionary<String, any>? else {
    return
}
```

That code will fail to compile against the Monterey SDK with the message `Conditional downcast from '[String : Any]?' to 'Dictionary<String, Any>' does nothing` when warnings are upconverted to errors via `SWIFT_TREAT_WARNINGS_AS_ERRORS = YES`. To get things to compile properly against both SDKs while keeping warnings as errors turned on, we can use our compilation condition to decide which version to compile based on the active SDK.
```swift
#if MAC_OS_VERSION_12_0_SDK_AVAILABLE
guard let recommendedAudioSettings = avCaptureAudioDataOutput.recommendedAudioSettingsForAssetWriter(...) else {
    return
}
#else
guard let recommendedAudioSettings = avCaptureAudioDataOutput.recommendedAudioSettingsForAssetWriter(...) as? Dictionary<String, any> else {
    return
}
#endif
```

---
[^1]: [Using compiler directives in Swift - Swift by Sundell](https://www.swiftbysundell.com/articles/using-compiler-directives-in-swift/)
[^2]: [The Unofficial Guide to xcconfig files](https://pewpewthespells.com/blog/xcconfig_guide.html)
[^3]: [http://codeworkshop.net/objc-diff/sdkdiffs/macos/12.0/AVFoundation.html](http://codeworkshop.net/objc-diff/sdkdiffs/macos/12.0/AVFoundation.html)
