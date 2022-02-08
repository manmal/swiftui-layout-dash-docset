---
layout: post
title: Yet another article about method swizzling
---

Many Objective-C developers disregard method swizzling, considering it a bad practice. I don't like method swizzling, I love it. Of course it is risky and can hurt you like a bullet. Carefully done, though, it makes it possible to fill annoying gaps in system frameworks which would be impossible to fill otherwise. From simply providing a convenient way to [track the parent popover controller of a view controller](https://github.com/defagos/CoconutKit/blob/f6d8d3486a2a201d0cda4657e681c3d56cf7a261/CoconutKit/Sources/ViewControllers/UIPopoverController+HLSExtensions.m#L16-L74) to implementing [Cocoa-like bindings on iOS](https://github.com/defagos/CoconutKit/blob/f6d8d3486a2a201d0cda4657e681c3d56cf7a261/CoconutKit/Sources/Bindings/UIView+HLSViewBinding.m#L237-L255), method swizzling has always been an invaluable tool to me.

* TOC
{:toc}

Implementing swizzling correctly is not easy, though, probably because it looks straightforward at first (all is needed is a few Objective-C runtime function calls, after all). Though the Web is [crawling with articles](https://www.google.ch/webhp?sourceid=chrome-instant&ion=1&espv=2&ie=UTF-8#q=objective-c%20swizzling%20right) about the right way to swizzle a method, I sadly found issues with all of them.

The implementation discussed in this article may have issues of its own, but I do hope sharing it will help improve existing implementations, as well as mine. For this reason, do not regard these few words as a _Done right_ article, only as a small step towards hopefully better swizzling implementations. There is a correct way to swizzle methods, but it probably still remains to be found.

## Function prototype

Since the Objective-C runtime is a series of functions, I decided to implement swizzling as a function as well. Most existing implementations, for example the well-respected [JRSwizzle](https://github.com/rentzsch/jrswizzle), exchange `IMP`s associated with two selectors. Ultimately, though, method swizzling is about changing, not exchanging, which is why I prefer a function expecting an original `SEL` and an `IMP` arguments:

```objc
IMP class_swizzleSelector(Class clazz, SEL selector, IMP newImplementation);
``` 

instead of two `SEL` arguments:

```objc
IMP class_swizzleSelectorWithSelector(Class clazz, SEL selector, SEL swizzlingSelector); 
```

Moreover, using an `IMP` (in other words a C-function) instead of a selector implementation avoids potential clashes if your swizzling selector name convention happens to be the same as the one used elsewhere, especially when dealing with 3rd party code. Don't be too optimistic, accidental method overriding due to [bad conventions](https://github.com/search?l=objective-c&q=%22%28void%29commonInit%22&ref=searchresults&type=Code&utf8=%E2%9C%93) can happen all the time.

The function above returns the original implementation, which must be properly cast and called from within the swizzling method implementation, so that the original behavior is preserved. If the method to swizzle is not implemented, I decided the function must do nothing and return `NULL`.

## Issues in class hierarchies

When swizzling methods in class hierarchies, we must take extra care when the method we swizzle is not implemented by the class on which it is swizzled, but is implemented by one of its parents. For example, the `-awakeFromNib` method, declared and implemented at the `NSObject` level, is neither implemented by the `UIView` nor by the `UILabel` subclasses. When calling this method on an instance of any of theses classes, it is therefore the `NSObject` implementation which gets called:

![Standard hierarchy](/images/standard_hierarchy.jpg)

If we naively swizzle the `-awakeFromNib` method both at the `UIView` and `UILabel` levels, we get the following result:

![Standard hierarchy, swizzled](/images/standard_hierarchy_swizzled.jpg)

As we see, when `-[UILabel awakeFromNib]` is now called, the swizzled `UIView `implementation does not get called, which is not what is expected from proper swizzling.

The situation would be completely different if the `-awakeFromNib` method was implemented on `UIView` and `UILabel`. If this was the case, and if each implementation properly called the `super` method counterpart first, we would namely obtain:

![Tweaked hierarchy](/images/tweaked_hierarchy.jpg)

and, after swizzling:

![Tweaked hierarchy, swizzled](/images/tweaked_hierarchy_swizzled.jpg)

No swizzling implementation I encountered correctly deals with this issue, [not even JRSwizzle](https://github.com/rentzsch/jrswizzle/issues/4). As should be clear from the last picture above, the solution to this problem is to ensure a method is always implemented by a class before swizzling it. If this is not the case, an implementation must be injected first, simply calling the super method counterpart. This way, all implementations will correctly be called after swizzling.

## First implementation attempt

Based on the above, I first implemented instance method swizzling as follows:

```objc
#import <objc/runtime.h>
#import <objc/message.h>

IMP class_swizzleSelector(Class clazz, SEL selector, IMP newImplementation)
{
    // If the method does not exist for this class, do nothing
    Method method = class_getInstanceMethod(clazz, selector);
    if (! method) {
        return NULL;
    }
    
    // Make sure the class implements the method. If this is not the case, inject an implementation, calling 'super'
    const char *types = method_getTypeEncoding(method);
    class_addMethod(clazz, selector, imp_implementationWithBlock(^(__unsafe_unretained id self, va_list argp) {
        struct objc_super super = {
            .receiver = self,
            .super_class = class_getSuperclass(clazz)
        };
        
        id (*objc_msgSendSuper_typed)(struct objc_super *, SEL, va_list) = (void *)&objc_msgSendSuper;
        return objc_msgSendSuper_typed(&super, selector, argp);
    }), types);
    
    // Can now safely swizzle
    return class_replaceMethod(clazz, selector, newImplementation, types);
}
```
For class method swizzling, it suffices to call the above function on a metaclass:

```objc
IMP class_swizzleClassSelector(Class clazz, SEL selector, IMP newImplementation)
{
    return class_swizzleSelector(object_getClass(clazz), selector, newImplementation);
}
```

The `imp_implementationWithBlock` function is used as a trampoline to accomodate any kind of method prototype through a variable argument list `va_list`. The `super` method call is made by properly casting `objc_msgSendSuper`, available from `<objc/message.h>`. In order to prevent ARC from inserting incorrect memory management calls, the `self` parameter of the implementation block has been marked with `__unsafe_unretained`.

## Large struct return values

As pointed out by [Peter Steinberger](https://twitter.com/steipete/status/540818091014627329) and [@__block](https://twitter.com/__block/status/540794469046812674) on Twitter, struct returns require special care on some architectures.

Most method calls are funnelled through the `objc_msgSend` function, returning the result in a register. For large structs which cannot fit in a register, though, the compiler might generate a call to the special `objc_msgSend_stret` function, which returns the parameter on the stack instead. According to the [ABI](https://developer.apple.com/library/mac/documentation/DeveloperTools/Conceptual/LowLevelABI/000-Introduction/introduction.html), this happens on 32-bit architectures for types whose size is neither 1, 2, 4 nor 8. The implementation above does not account for this special case and must therefore be changed to account for such cases. 

For method returning large structs, we need the `imp_implementationWithBlock` function to generate the correct implementation by having the block return a large struct. The kind of struct and its layout are irrelevant, we only need it to be sufficiently large so that the compiler can make the right decision. As for `objc_msgSend_stret`, there is an `objc_msgSendSuper_stret` for super calls to methods returning large structs, which we need to use instead. 

For large struct returns, instead of calling the `class_swizzleSelector` function above, we therefore must call the following `class_swizzleSelector_stret` function:

```objc
#import <objc/runtime.h>
#import <objc/message.h>

IMP class_swizzleSelector_stret(Class clazz, SEL selector, IMP newImplementation)
{
    // If the method does not exist for this class, do nothing
    Method method = class_getInstanceMethod(clazz, selector);
    if (! method) {
        return NULL;
    }
    
    // Make sure the class implements the method. If this is not the case, inject an implementation, only calling 'super'
    const char *types = method_getTypeEncoding(method);
    class_addMethod(clazz, selector, imp_implementationWithBlock(^(__unsafe_unretained id self, va_list argp) {
        struct objc_super super = {
            .receiver = self,
            .super_class = class_getSuperclass(clazz)
        };
        
        // Sufficiently large struct
        typedef struct LargeStruct_ {
            char dummy[16];
        } LargeStruct;
        
        // Cast the call to objc_msgSendSuper_stret appropriately
        LargeStruct (*objc_msgSendSuper_stret_typed)(struct objc_super *, SEL, va_list) = (void *)&objc_msgSendSuper_stret;
        return objc_msgSendSuper_stret_typed(&super, selector, argp);
    }), types);
    
    // Can now safely swizzle
    return class_replaceMethod(clazz, selector, newImplementation, types);
}
```

## Wrapping up: IMP-swizzling

Having a separate `class_swizzleSelector_stret` function which must appropriately be called when large structs are returned is rather inconvenient. Fortunately, its implementation can be merged into `class_swizzleSelector` by checking return type size information for 32-bit architectures first. We obtain a single function for method swizzling with an `IMP`:

```objc
#import <objc/runtime.h>
#import <objc/message.h>

IMP class_swizzleSelector(Class clazz, SEL selector, IMP newImplementation)
{
    // If the method does not exist for this class, do nothing
    Method method = class_getInstanceMethod(clazz, selector);
    if (! method) {
        // Cannot swizzle methods which are not implemented by the class or one of its parents
        return NULL;
    }
    
    // Make sure the class implements the method. If this is not the case, inject an implementation, only calling 'super'
    const char *types = method_getTypeEncoding(method);
    
#if !defined(__arm64__)
    NSUInteger returnSize = 0;
    NSGetSizeAndAlignment(types, &returnSize, NULL);
    
    // Large structs on 32-bit architectures
    if (sizeof(void *) == 4 && types[0] == _C_STRUCT_B && returnSize != 1 && returnSize != 2 && returnSize != 4 && returnSize != 8) {
        class_addMethod(clazz, selector, imp_implementationWithBlock(^(__unsafe_unretained id self, va_list argp) {
            struct objc_super super = {
                .receiver = self,
                .super_class = class_getSuperclass(clazz)
            };
            
            // Sufficiently large struct
            typedef struct LargeStruct_ {
                char dummy[16];
            } LargeStruct;
            
            // Cast the call to objc_msgSendSuper_stret appropriately
            LargeStruct (*objc_msgSendSuper_stret_typed)(struct objc_super *, SEL, va_list) = (void *)&objc_msgSendSuper_stret;
            return objc_msgSendSuper_stret_typed(&super, selector, argp);
        }), types);
    }
    // All other cases
    else {
#endif
        class_addMethod(clazz, selector, imp_implementationWithBlock(^(__unsafe_unretained id self, va_list argp) {
            struct objc_super super = {
                .receiver = self,
                .super_class = class_getSuperclass(clazz)
            };
            
            // Cast the call to objc_msgSendSuper appropriately
            id (*objc_msgSendSuper_typed)(struct objc_super *, SEL, va_list) = (void *)&objc_msgSendSuper;
            return objc_msgSendSuper_typed(&super, selector, argp);
        }), types);
#if !defined(__arm64__)
    }
#endif
    
    // Swizzling
    return class_replaceMethod(clazz, selector, newImplementation, types);
}
``` 

The `_stret` variants are not available on ARM 64, thus the extra preprocessor adornments.

### Example of use
{:.no_toc}

To swizzle a method, define a static C-function for the new implementation and call `class_swizzleSelector` or `class_swizzleClassSelector` to set it as new one. Save the original implementation into a function pointer matching the function signature, and make sure the new implementation calls it somehow:

```objc
static id (*initWithFrame)(id, SEL, CGRect) = NULL;
static void (*awakeFromNib)(id, SEL) = NULL;
static void (*dealloc)(__unsafe_unretained id, SEL) = NULL;

static id swizzle_initWithFrame(UILabel *self, SEL _cmd, CGRect frame)
{
    if ((self = initWithFrame(self, _cmd, frame))) {
        // ...
    }
    return self;
}

static void swizzle_awakeFromNib(UILabel *self, SEL _cmd)
{
    awakeFromNib(self, _cmd);
    
    // ...
}

static void swizzle_dealloc(__unsafe_unretained UILabel *self, SEL _cmd)
{
    // ...
    
    dealloc(self, _cmd);
}

@implementation UILabel (SwizzlingExamples)

+ (void)load
{
    initWithFrame = (__typeof(initWithFrame))class_swizzleSelector(self, @selector(initWithFrame:), (IMP)swizzle_initWithFrame);
    awakeFromNib = (__typeof(awakeFromNib))class_swizzleSelector(self, @selector(awakeFromNib), (IMP)swizzle_awakeFromNib);
    dealloc = (__typeof(dealloc))class_swizzleSelector(self, sel_getUid("dealloc"), (IMP)swizzle_dealloc);
}

@end
``` 

Note that I added an extra `__unsafe_unretained` specifier to the `swizzle_dealloc` prototype to ensure ARC does not insert additional memory management calls. I also cheated by getting the `dealloc` selector with `sel_getUid`, since `@selector(dealloc)` cannot be used with ARC.

## Block-swizzling

Thanks to `imp_implementationWithBlock`, we can provide a block instead of an `IMP` for the new implementation:

```objc
IMP class_swizzleSelectorWithBlock(Class clazz, SEL selector, id newImplementationBlock)
{
    IMP newImplementation = imp_implementationWithBlock(newImplementationBlock);
    return class_swizzleSelector(clazz, selector, newImplementation);
}

IMP class_swizzleClassSelectorWithBlock(Class clazz, SEL selector, id newImplementationBlock)
{
    IMP newImplementation = imp_implementationWithBlock(newImplementationBlock);
    return class_swizzleClassSelector(clazz, selector, newImplementation);
}
``` 

The block signature itself does not include the selector parameter, as specified in the `imp_implementationWithBlock` [documentation](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/ObjCRuntimeRef/index.html).

### Example of use
{:.no_toc}

The above example can be rewritten using blocks, eliminating the need for static methods and function pointers:

```objc
@implementation UILabel (SwizzlingExamples)

+ (void)load
{
    __block IMP originalInitWithFrame = class_swizzleSelectorWithBlock(self, @selector(initWithFrame:), ^(UILabel *self, CGRect frame) {
        if ((self = ((id (*)(id, SEL, CGRect))originalInitWithFrame)(self, @selector(initWithFrame:), frame))) {
            // ...
        }
        return self;
    });
    
    __block IMP originalAwakeFromNib = class_swizzleSelectorWithBlock(self, @selector(awakeFromNib), ^(UILabel *self) {
        ((void (*)(id, SEL))originalAwakeFromNib)(self, @selector(awakeFromNib));
        
        // ...
    });
    
    __block IMP originalDealloc = class_swizzleSelectorWithBlock(self, sel_getUid("dealloc"), ^(__unsafe_unretained UILabel *self) {
        // ...

        ((void (*)(id, SEL))originalDealloc)(self, sel_getUid("dealloc"));
    });
}

@end
``` 

Returned original implementations must be saved into `__block` variables to be accessible from within the corresponding implementation blocks.

## Macros

Some redundancy is found in both examples of use above, but can be eliminated by defining a few convenience macros:

```objc
#define SwizzleSelector(clazz, selector, newImplementation, pPreviousImplementation) \
    (*pPreviousImplementation) = (__typeof((*pPreviousImplementation)))class_swizzleSelector((clazz), (selector), (IMP)(newImplementation))

#define SwizzleClassSelector(clazz, selector, newImplementation, pPreviousImplementation) \
    (*pPreviousImplementation) = (__typeof((*pPreviousImplementation)))class_swizzleClassSelector((clazz), (selector), (IMP)(newImplementation))

#define SwizzleSelectorWithBlock_Begin(clazz, selector) { \
    SEL _cmd = selector; \
    __block IMP _imp = class_swizzleSelectorWithBlock((clazz), (selector),
#define SwizzleSelectorWithBlock_End );}

#define SwizzleClassSelectorWithBlock_Begin(clazz, selector) { \
    SEL _cmd = selector; \
    __block IMP _imp = class_swizzleClassSelectorWithBlock((clazz), (selector),
#define SwizzleClassSelectorWithBlock_End );}
``` 

To emphasize that the `IMP`-swizzling macros set the new implementation variable, the corresponding macro parameter needs to be adorned with an ampersand. 

Block-swizzling, on the other hand, has been turned into a pair of macros. This avoids block parameters, which do not work well with macros:

* Macro expansion of a block turns the block implementation into a single line, confusing the debugger
* A block can contain commas, preventing correct macro argument detection (this can be avoided by enclosing the blocks within parentheses, though)

Moreover, the block-swizzling macros declare a hidden scope where the selector `_cmd` and original implementation `_imp` are immediately available.

### Example of use
{:.no_toc}

The two previous examples can now be rewritten as follows:

```objc
@implementation UILabel (SwizzlingExamples)

// Function pointers and static functions, as defined above

+ (void)load
{
    SwizzleSelector(self, @selector(initWithFrame:), swizzle_initWithFrame, &initWithFrame);
    SwizzleSelector(self, @selector(awakeFromNib), &swizzle_awakeFromNib, &awakeFromNib);
    SwizzleSelector(self, sel_getUid("dealloc"), &swizzle_dealloc, &dealloc);
}

@end
``` 

respectively:

```objc
@implementation UILabel (SwizzlingExamples)

+ (void)load
{
    SwizzleSelectorWithBlock_Begin(self, @selector(initWithFrame:))
    ^(UILabel *self, CGRect frame) {
        if ((self = ((id (*)(id, SEL, CGRect))_imp)(self, _cmd, frame))) {
            // ...
        }
        return self;
    }
    SwizzleSelectorWithBlock_End;
    
    SwizzleSelectorWithBlock_Begin(self, @selector(awakeFromNib))
    ^(UILabel *self) {
        ((void (*)(id, SEL))_imp)(self, _cmd);
        
        // ...
    }
    SwizzleSelectorWithBlock_End;

    SwizzleSelectorWithBlock_Begin(self, sel_getUid("dealloc"))
    ^(__unsafe_unretained UILabel *self) {
        // ...
        
        ((void (*)(id, SEL))_imp)(self, _cmd);
    }
    SwizzleSelectorWithBlock_End;
}

@end
``` 

I especially like the compactness of block-swizzling using macros. There is no need to define separate functions and method pointers, and the swizzled method implementation can be easily recognized, enclosed between the `Begin` and `End` macros.

## Conclusion

The implementation discussed in this article might not be optimal, but should cover most practical cases. It is available from my [CoconutKit framework](https://github.com/defagos/CoconutKit), with a comprehensive [test suite](https://github.com/defagos/CoconutKit/blob/f6d8d3486a2a201d0cda4657e681c3d56cf7a261/CoconutKit-tests/Sources/Core/HLSRuntimeTestCase.m#L1277-L1370), or from the following [gist](https://gist.github.com/defagos/1312fec96b48540efa5c). Have fun with method swizzling!
