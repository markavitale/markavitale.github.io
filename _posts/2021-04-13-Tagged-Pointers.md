---
layout: post
title:  "Objective-C's Tagged Pointers and Localized Over-releases"
---

*Disclaimer: This particular class of problem can be more easily solved by using [ARC](https://clang.llvm.org/docs/AutomaticReferenceCounting.html). The code I was working in did not use ARC at the time for a variety of reasons, so manual memory management was necessary.*

### The Crash

A few years back I was investigating a crash in some Objective-C code that was pretty easy to figure through code inspection. This code didn't use ARC and required manual memory management. At some point a localized string was assigned to an instance variable and failed to `copy` or `retain` it. I did, however, remember to release the instance variable in `dealloc`, resulting in a pretty obvious crash.

```obj-c
- (void)setString:(NSString *)string {
    // ...
    _string = string; // Needs a copy or retain
    // ...
}

- (void)dealloc {
    // ...
    [_string release];
    // ...
}
```

### The Inconsistency

Despite having found the root cause of the crash, I couldn’t reconcile in my head why this crash would only occur in certain languages. In particular we were not seeing it in English, French, or Spanish, (in)conveniently the languages I chose to verify that the feature was working properly in different localizations. 

### The Explanation

What was happening? Apple's ~~too~~ clever Tagged Pointers were in play here. If the data being pointed to by a pointer is actually smaller than the size of the pointer itself, the value of the object can be packed into the pointer itself. This happens transparently for a few specific object types[^1].
 
Apple has some tricks to determine whether a pointer is a "real" pointer or a tagged pointer. The details of that are too technical for this particular post, but Mike Ash has an awesome blog post you can look at if you're more interested on the technical implementation details [^2].
 
Because the value of the object is stored in the pointer value themselves, retaining, copying, and releasing them are no-ops. If you have the pointer, you have the value. So over-releasing, leaking, and related issues don’t really exist with a tagged pointer (and the [Zombies](https://help.apple.com/instruments/mac/current/#/dev612e6956) tool won’t find memory issues with tagged pointers either). Another important detail is that tagged pointers are completely transparent to the developer. When creating an object of one of those types listed above, if it fits, Apple will create a tagged pointer, and if it doesn’t, it won’t. So you always have to manage memory correctly and can’t bank on it being a tagged pointer and ignoring memory concerns.
 
So in our crash case, purely by chance, the English, French, and Spanish strings were all small enough to fit into an Tagged Pointer and thus weren't crashing on an over-release because release is a no-op on tagged pointers. In other languages where by chance the string required a true pointer to some memory on the heap, the missing `copy`/`retain` did matter and our crashes were consistent.

Thanks for reading, I hope you found this interesting!
 
For more reference on tagged pointer strings, check out Mike Ash's super awesome post [here](https://www.mikeash.com/pyblog/friday-qa-2015-07-31-tagged-pointer-strings.html).

---

[^1]: It's not just `NSString`. Here is a list of object types that can be implemented as a tagged pointer:
    ```obj-c
    OBJC_TAG_NSAtom            = 0,
    OBJC_TAG_1                 = 1,
    OBJC_TAG_NSString          = 2,
    OBJC_TAG_NSNumber          = 3,
    OBJC_TAG_NSIndexPath       = 4,
    OBJC_TAG_NSManagedObjectID = 5,
    OBJC_TAG_NSDate            = 6,
    OBJC_TAG_7                 = 7
    ```
    From [objc-internal.h](https://opensource.apple.com/source/objc4/objc4-709/runtime/objc-internal.h.auto.html), discovered via a [Stack Overflow post](http://stackoverflow.com/questions/20362406/tagged-pointers-in-objective-c).

[^2]: [Friday Q&A 2012-07-27: Let's Build Tagged Pointers - by Mike Ash](https://www.mikeash.com/pyblog/friday-qa-2012-07-27-lets-build-tagged-pointers.html)