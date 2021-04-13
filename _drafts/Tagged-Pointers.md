---
layout: post
title:  "Tagged Pointers and Localized Overreleases"
---


# {{ post.title }}
Disclaimer: This particular class of problem can be more easily solved by using [ARC](https://clang.llvm.org/docs/AutomaticReferenceCounting.html). The code I was working in did not use ARC at the time for a variety of reasons, so manual memory management was necessary.

I’ve been working on a crash where we forgot to retain/copy an NSString in the overridden setter of a property, but remembered to release it in dealloc. This obviously gives us a crash in the instance that we receive an autoreleased string and never copy/retain it. This seems like a very straightforward crash to track down, and doesn’t warrant any new learning. Here’s the weird part: the crash only repro’d in certain languages: Portuguese (Brazil), Japanese, Chinese, Russian, Thai and other languages. The crash did not repro in English, Spanish, or French.
 
This struck me as very weird. Despite having found the root cause of the crash, I couldn’t reconcile in my head why this crash would only occur in certain languages. I then remembered a concept I had only vaguely heard of before, tagged pointers. Apple is clever and doesn’t want to waste memory on the heap when it doesn’t have to. For very specific object types, Apple may decide that instead of giving you a real pointer that points to some memory on the heap somewhere, it will instead just put the value of your ‘object’ directly into the pointer itself. Through some searching online, I found the following list of object types that can be implemented as a tagged pointer:
 
*OBJC_TAG_NSAtom            = 0,
*OBJC_TAG_1                 = 1,
*OBJC_TAG_NSString          = 2,
*OBJC_TAG_NSNumber          = 3,
*OBJC_TAG_NSIndexPath       = 4,
*OBJC_TAG_NSManagedObjectID = 5,
*OBJC_TAG_NSDate            = 6,
*OBJC_TAG_7                 = 7
 
From http://stackoverflow.com/questions/20362406/tagged-pointers-in-objective-c
 
Apple has some little tricks to determine whether a pointer is a real pointer or a tagged pointer, and you can look at https://www.mikeash.com/pyblog/friday-qa-2012-07-27-lets-build-tagged-pointers.html to try and understand the details.
 
One important detail is that since the value of the object is stored in the pointer value themselves, retaining, copying, and releasing them are no-ops. If you have the pointer, you have the value. So over releasing, leaking, and related issues don’t really exist with a tagged pointer (and the Zombies tool won’t find memory issues with tagged pointers either). Another very important detail is that you can’t ask for a tagged pointer specifically. When creating an object of one of those types listed above, if it fits, Apple will create a tagged pointer, and if it doesn’t, it won’t. So you always have to manage memory correctly and can’t bank on it being a tagged pointer and ignoring memory concerns.
 
So the weirdness with this bug was around Apple doing some tagged pointer magic with our strings in certain languages which prevented us from crashing, while using standard NSString pointers to memory on the heap in other languages, causing the behavioral difference that caused us to crash. I wasn’t super familiar with how tagged pointers actually worked, and learning about them today helped me understand an otherwise mind-boggling bug, and thought I would share.
 
More reference: https://www.mikeash.com/pyblog/friday-qa-2015-07-31-tagged-pointer-strings.html
