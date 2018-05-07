---
layout: post
title: "组件化之组件生命周期管理"
date: 2017-10-25 18:18:59 +0800
comments: true
categories: iOS
author: 青木
---
在没有实行组件化的项目中，经常会在 AppDelegate 看到各类初始化代码，这一部分代码一般用以配置某些 key 以及 secret ，或者开启某些服务，常见的有第三方推送、统计分析、IM服务等。当然，也有可能是开启一些自身的服务，比如 log 日志、 数据库初始化等。当一个 App 达到一定体量后， 未经整理的 AppDelegate 可能会变得臃肿。那么在实行组件化之后，该如何处理这部分代码呢？

<!--more-->

## 不管理组件生命周期

不对组件生命周期进行管理，那么只能继续将这些初始化代码放在主工程的 AppDelegate 中，而针对上文所说的 AppDelegate 臃肿的问题，也可以通过简单的封装来优化。

但是，这种做法会引发组件独立性问题。比如存在能独立运行的组件 A、B，B 依赖 A， A 生效需要在 App Launch 时调用配置代码 Code-A。如果采用上述做法，那么组件 A 所在示例工程的 AppDelegate 中，需要调用 Code-A 进行配置，而组件 B 因为依赖了 组件 A ，要使组件 B 能成功运行，也需要在 B 的示例工程添加 Code-A 进行配置。同样主工程的 AppDelegate 中也存在一份 Code-A 配置代码。可以看到，这种重复手动配置的做法是比较繁琐和难看的，这也是为什么要对组件生命周期进行管理的原因。

## 现有实现管理方案

从组件和主工程的关系切入，既然组件需要在 App 生命周期的某些阶段处理特定的事务，那么就提供特定的回调方法供组件使用。 App 生命周期各个阶段产生的事件，可以通过让 AppDelegate 遵守 UIApplicationDelegate 协议并实现不同的代理方法进行捕获。

要想把当前阶段 App 产生的事件分发给各个组件，最简单的方案就是如 [limboy](http://limboy.me/tech/2016/03/10/mgj-components.html) 所说，在 AppDelegate 的各个代理方法里，手动调一遍组件的对应方法，如果组件实现了对应的代理方法，就执行：

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    [MGJApp startApp];

    [[ModuleManager sharedInstance] loadModuleFromPlist:[[NSBundle mainBundle] pathForResource:@"modules" ofType:@"plist"]];
    NSArray *modules = [[ModuleManager sharedInstance] allModules];
    for (id<ModuleProtocol> module in modules) {
        if ([module respondsToSelector:_cmd]) {
            [module application:application didFinishLaunchingWithOptions:launchOptions];
        }
    }

    [self trackLaunchTime];
    return YES;
}
```

不过这种方式缺点也很明显，组件需要依赖主工程的 AppDelegate 是否实现了 UIApplicationDelegate 的代理方法，如果没有的话，即使组件方实现了对应的代理方法，依然无法捕获到事件。

再来看下 [caojun](http://www.jianshu.com/u/d8a653fc1cb1) 的处理方案 [YTXModule](https://github.com/mdsb100/YTXModule)。
这个方案主要思路是通过 runtime method swizzling，替换 AppDelegate 中实现的 UIApplicationDelegate 代理方法，然后在 swizzled method 中，执行事件分发。 YTXModule 提供了一些宏定义，精简了方法替换流程：

```objc
@implementation UIApplication (YTXModule)
- (void)module_setDelegate:(id<UIApplicationDelegate>) delegate
{

    static dispatch_once_t delegateOnceToken;
    dispatch_once(&delegateOnceToken, ^{
        SWIZZLE_DELEGATE_METHOD(applicationDidFinishLaunching:);
        ...
    });
    [self module_setDelegate:delegate];
}
@end

@implementation YTXModule
...
+ (BOOL)ytxmodule_application:(UIApplication *)application didFinishLaunchingWithOptions:(nullable NSDictionary *)launchOptions
{   
    DEF_APPDELEGATE_METHOD_CONTAIN_RESULT(application, launchOptions);
}
...
@end
```
这里需要注意的是，由于 method swizzling 是在不同类型载体（AppDelegate对象 <-> YTXModule类）间交换的方法，所以会造成在 `+ytxmodule_applicationDidFinishLaunching:` 中调用 `self` 时，获取的并不是 YTXModule类，而是 AppDelegate对象，因为方法替换实际上替换了 IMP，并没有改变实参，参照 `objc_msgSend(id self, SEL op, ... )` 的参数排列，可以明确第一个参数是消息接收者，也就是 AppDelegate对象。通过上述分析可以知道，如果直接进行方法替换，不做特殊处理，使用以下代码将会抛出 `unrecognized selector ` 异常 ：

```objc
+ (BOOL)ytxmodule_application:(UIApplication *)application didFinishLaunchingWithOptions:(nullable NSDictionary *)launchOptions
{
	// self is AppDelegate instance
    [self ytxmodule_application:application didFinishLaunchingWithOptions:launchOptions];
}
```

而以下代码，是可以正常运行的：

```objc
+ (BOOL)ytxmodule_application:(UIApplication *)application didFinishLaunchingWithOptions:(nullable NSDictionary *)launchOptions
{
 	[YTXModule ytxmodule_application:application didFinishLaunchingWithOptions:launchOptions];
}
```

所以 caojun 在方法替换时，给 AppDelegate 添加了相同命名的实例方法，规避了这个异常 ：

```objc
void Swizzle(Class class, SEL originalSelector, Method swizzledMethod)
{
    Method originalMethod = class_getInstanceMethod(class, originalSelector);
    SEL swizzledSelector = method_getName(swizzledMethod);

    BOOL didAddMethod =
    class_addMethod(class,
                    originalSelector,
                    method_getImplementation(swizzledMethod),
                    method_getTypeEncoding(swizzledMethod));

    if (didAddMethod && originalMethod) {
        class_replaceMethod(class,
                            swizzledSelector,
                            method_getImplementation(originalMethod),
                            method_getTypeEncoding(originalMethod));
    } else {
        method_exchangeImplementations(originalMethod, swizzledMethod);
    }
    // 这一步给 AppDelegate 添加相同命名的实例方法，并且其 IMP 是 AppDelegate 自身方法的原实现
    class_addMethod(class,
                    swizzledSelector,
                    method_getImplementation(swizzledMethod),
                    method_getTypeEncoding(swizzledMethod));
}
```

虽说这种方案也能实现事件的分发，但是在不同类型载体间使用 method swizzling  还是应该避免的，对其他开发者不是很友好。并且这种方案也存在依赖 YTXModule 是否替换了 UIApplicationDelegate 的代理方法问题，如果没有，组件方是无法捕获事件的。


## 一种更加优雅的方案

> 分发、代理

看到这两个关键词，可以直接联想到 runtime 的另一重要组成部分，消息转发。以下是我结合消息转发实现的组件生命周期管理方案。

先看下 UML 类图：

![](/assets/images/2017-10-25-zu-jian-sheng-ming-zhou-qi/1509178037666.jpg)

首先是 TDFModule ，模块基类，所有想要捕获 App 生命周期事件的模块都需要创建一个继承 TDFModule 的类，并且遵守 TDFModuleProtocol 协议：

```

/**
 模块子类必须遵守此协议
 */
@protocol TDFModuleProtocol <UIApplicationDelegate>
@end

/**
 模块优先级

 - TDFModulePriorityVeryLow: 极底
 - TDFModulePriorityLow: 低
 - TDFModulePriorityMedium: 中
 - TDFModulePriorityHigh: 高
 - TDFModulePriorityVeryHigh: 极高
 */
typedef NS_ENUM(NSInteger, TDFModulePriority) {
    TDFModulePriorityVeryLow = 0,
    TDFModulePriorityLow = 1,
    TDFModulePriorityMedium = 2,
    TDFModulePriorityHigh = 3,
    TDFModulePriorityVeryHigh = 4,
};

@interface TDFModule : NSObject
+ (instancetype)module;

/**
 在 load 中调用，以注册模块
 */
+ (void)registerModule;

/**
 模块优先级

 主工程模块的调用最先进行，剩余附属模块，
 内部会根据优先级，依次调用 UIApplicationDelegate 代理
 默认是 TDFModulePriorityMedium

 @return 优先级
 */
+ (TDFModulePriority)priority;
@end



@implementation TDFModule
- (instancetype)init {
    if (self = [super init]) {
        if (![self conformsToProtocol:@protocol(TDFModuleProtocol)]) {
            @throw [NSException exceptionWithName:@"TDFModuleRegisterProgress" reason:@"subclass should confirm to <TDFModuleProtocol>." userInfo:nil];
        }
    }

    return self;
}

+ (instancetype)module {
    return [[self alloc] init];
}

+ (void)registerModule {
    [TDFModuleManager addModuleClass:self];
}

+ (TDFModulePriority)priority {
    return TDFModulePriorityMedium;
}
```
子类需要在 `+load` 方法中调用 `registerModule` 才能让模块具备捕获 App 事件的能力。因为 TDFModuleProtocol 直接遵守的 UIApplicationDelegate 协议，子类可以和 AppDelegate 一样，直接实现自己感兴趣的代理方法即可：

```
@interface TDFAModule : TDFModule <TDFModuleProtocol>
@end

@implementation TDFAModule
+ (void)load {
    [self registerModule];
}

+ (TDFModulePriority)priority {
    return TDFModulePriorityHigh;
}

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    NSLog(@"%@, %@", NSStringFromClass([self class]), NSStringFromSelector(_cmd));
    return YES;
}

- (void)applicationWillEnterForeground:(UIApplication *)application {
    NSLog(@"%@, %@", NSStringFromClass([self class]), NSStringFromSelector(_cmd));
}

- (void)applicationDidBecomeActive:(UIApplication *)application {
    NSLog(@"%@, %@", NSStringFromClass([self class]), NSStringFromSelector(_cmd));
}

- (void)applicationWillTerminate:(UIApplication *)application {
    NSLog(@"%@, %@", NSStringFromClass([self class]), NSStringFromSelector(_cmd));
}

@end
```
以上就是一个简单的使用示例。

接下来是 TDFModuleManager ，模块管理类。这个单例类主要负责模块的储存，以及在 UIApplication 的 `-setDelegate:` 中，把原来 delegate 替换成自己的 delegate proxy 。

```
@interface TDFModuleManager : NSObject {
    @package
    TDFApplicationDelegateProxy *_proxy;
}
@property (strong, nonatomic, readonly) TDFApplicationDelegateProxy *proxy;
@property (strong, nonatomic, readonly) NSArray <TDFModule *> *modules;

+ (instancetype)shared;
+ (void)addModuleClass:(Class)cls;
+ (void)removeModuleClass:(Class)cls;
@end


static NSMutableArray const * TDFModuleClassArray = nil;

@implementation TDFModuleManager
+ (instancetype)shared {
    static dispatch_once_t onceToken;
    static TDFModuleManager *singleton = nil;
    dispatch_once(&onceToken, ^{
        singleton = [[self alloc] init];
    });
    return singleton;
}

+ (void)addModuleClass:(Class)cls {
    NSParameterAssert(cls && [cls isSubclassOfClass:[TDFModule class]]);

    if (!TDFModuleClassArray) {
        TDFModuleClassArray = [NSMutableArray array];
    }

    if (![TDFModuleClassArray containsObject:cls]) {
        [TDFModuleClassArray addObject:cls];
    }
}

+ (void)removeModuleClass:(Class)cls {
    [TDFModuleClassArray removeObject:cls];
}

- (void)generateRegistedModules {
    [self.mModules removeAllObjects];

    [TDFModuleClassArray sortUsingDescriptors:@[[[NSSortDescriptor alloc] initWithKey:@"priority" ascending:NO]]];

    for (Class cls in TDFModuleClassArray) {
        TDFModule *module = [cls module];
        NSAssert(module, @"module can't be nil of class %@", NSStringFromClass(cls));

        if (![self.mModules containsObject:module]) {
            [self.mModules addObject:module];
        }
    }
}

- (TDFApplicationDelegateProxy *)proxy {
    if (!_proxy) {
        _proxy = [[TDFApplicationDelegateProxy alloc] init];
    }

    return _proxy;
}

- (NSArray<TDFModule *> *)modules {
    return (NSArray<TDFModule *> *)self.mModules;
}

- (NSMutableArray<TDFModule *> *)mModules {
    if (!_mModules) {
        _mModules = [NSMutableArray array];
    }

    return _mModules;
}
@end

static void MCDSwizzleInstanceMethod(Class cls, SEL originalSelector, Class targetCls, SEL swizzledSelector) {
    Method originalMethod = class_getInstanceMethod(cls, originalSelector);
    Method swizzledMethod = class_getInstanceMethod(targetCls, swizzledSelector);
    BOOL didAddMethod = class_addMethod(cls, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod));
    if (didAddMethod) {
        class_replaceMethod(cls, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
    } else {
        method_exchangeImplementations(originalMethod, swizzledMethod);
    }
}

@implementation UIApplication (TDFModule)
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        MCDSwizzleInstanceMethod(self, @selector(setDelegate:), self, @selector(mcd_setDelegate:));
    });
}

- (void)mcd_setDelegate:(id <UIApplicationDelegate>)delegate {
    TDFModuleManager.shared.proxy.realDelegate = delegate;
    [TDFModuleManager.shared generateRegistedModules];

    [self mcd_setDelegate:(id <UIApplicationDelegate>)TDFModuleManager.shared.proxy];
}
@end
```

最后是这个方案的重点，也就是 TDFApplicationDelegateProxy 类：

```objc
@interface TDFApplicationDelegateProxy : NSObject
@property (strong, nonatomic) id <UIApplicationDelegate> realDelegate;
@end

@implementation TDFApplicationDelegateProxy
- (Protocol *)targetProtocol {
    return @protocol(UIApplicationDelegate);
}

- (BOOL)isTargetProtocolMethod:(SEL)selector {
    unsigned int outCount = 0;
    struct objc_method_description *methodDescriptions = protocol_copyMethodDescriptionList([self targetProtocol], NO, YES, &outCount);

    for (int idx = 0; idx < outCount; idx++) {
        if (selector == methodDescriptions[idx].name) {
            return YES;
        }
    }
    free(methodDescriptions);

    return NO;
}

- (BOOL)respondsToSelector:(SEL)aSelector {
    if ([self.realDelegate respondsToSelector:aSelector]) {
        return YES;
    }

    for (TDFModule *module in [TDFModuleManager shared].modules) {
        if ([self isTargetProtocolMethod:aSelector] && [module respondsToSelector:aSelector]) {
            return YES;
        }
    }

    return [super respondsToSelector:aSelector];
}

- (id)forwardingTargetForSelector:(SEL)aSelector {
    if (![self isTargetProtocolMethod:aSelector] && [self.realDelegate respondsToSelector:aSelector]) {
            return self.realDelegate;
    }
    return self;
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    struct objc_method_description methodDescription = protocol_getMethodDescription([self targetProtocol], aSelector, NO, YES);

    if (methodDescription.name == NULL && methodDescription.types == NULL) {
        return [[self class] instanceMethodSignatureForSelector:@selector(doNothing)];
    }

    return [NSMethodSignature signatureWithObjCTypes:methodDescription.types];;
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    NSMutableArray *allModules = [NSMutableArray arrayWithObjects:self.realDelegate, nil];
    [allModules addObjectsFromArray:[TDFModuleManager shared].modules];
    
    // BOOL 型返回值做特殊 | 处理
    if (anInvocation.methodSignature.methodReturnType[0] == 'B') {
        BOOL realReturnValue = NO;
        
        for (TDFModule *module in allModules) {
            if ([module respondsToSelector:anInvocation.selector]) {
                [anInvocation invokeWithTarget:module];
                
                BOOL returnValue = NO;
                [anInvocation getReturnValue:&returnValue];
                
                realReturnValue = returnValue || realReturnValue;
            }
        }
        
        [anInvocation setReturnValue:&realReturnValue];
    } else {
        for (TDFModule *module in allModules) {
            if ([module respondsToSelector:anInvocation.selector]) {
                [anInvocation invokeWithTarget:module];
            }
        }
    }
}

- (void)doNothing {}
@end
```

先说 `-respondsToSelector:` ，由于系统内部会调用这个方法，判断是否实现了对应的 UIApplicationDelegate 代理方法，所以这里结合 AppDelegate 以及所有注册的 Module 判断是否有相应实现。

当 `-respondsToSelector:` 返回 YES 后，程序来到消息转发第二步 Fast forwarding path ，对应方法 `-forwardingTargetForSelector:`，在这一步，我们判断转发的方法是否为 UIApplicationDelegate 的代理方法，如果不是，并且 realDelegate（也就是 AppDelegate） 能响应，就直接把消息转发给 realDelegate。

如果在上一步中没有把消息转发给 realDelegate，那么就到了消息转发的最后一步 Normal forwarding path ，对应方法 `-methodSignatureForSelector:` 和 `-forwardInvocation:`，在这一步我们首先根据协议直接返回代理方法的签名，然后在 `-forwardInvocation:` 方法中，按照优先级，依次把消息转发给注册的模块。

在不做额外操作的前提下， `-forwardInvocation:` 中只有最后一次调用的返回值会成为实际返回值，当实现类似 `- (BOOL)application:(UIApplication *)app openURL:(NSURL *)url options:(NSDictionary<UIApplicationOpenURLOptionsKey,id> *)options` 等返回 BOOL 值的代理方法时，就会出现问题。所以这里通过判断返回值是否为 BOOL 类型，去执行不同的操作。如果为 BOOL 类型，则对所有返回值执行逻辑或操作，并将结果设置成实际返回值。

总结起来，流程如下：

![](/assets/images/2017-10-25-zu-jian-sheng-ming-zhou-qi/1509083789825.jpg)

经过上面几步，就可以把 App 的事件分发给各个组件了，而且组件对事件的捕获是不依赖于外界（AppDelegate）实现的，只要进行注册就可以了，个人认为还是比较优雅的。

## Demo地址

[TDFModuleKit](https://github.com/tripleCC/TDFModuleKit)

## 参考

[iOS App组件化开发实践](http://www.jianshu.com/p/48fbcbb36c75)<br>
[蘑菇街 App 的组件化之路](http://www.jianshu.com/p/48fbcbb36c75)<br>
