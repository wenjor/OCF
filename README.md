[TOC]

#### 一、Runloop

##### 1.概念

一个对象，管理了其所需处理的消息与事件，并提供一个入口函数来执行Event Loop的逻辑。

`Event Loop`

```python
function loop() {
    initialize();
    do {
        var message = get_next_message();
        process_message(message);
    } while (message != quit);
}
```

关键在于如何管理消息/实践并提供了入口函数（接受->等待->处理）来执行上述逻辑。

##### 2.与线程的关系

线程对象有`pthread_t`和`NSThread`

可通过`pthread_main_thread_np()`或 `[NSTread currentThread]`来获取当前进程。

线程和RunLoop之间一一对应，并保存于一个全局的Dictionary里。线程创建之初没有RunLoop不主动获取就一直不会有。RunLoop的销毁时发生在线程结束时。只能在线程内部获得其RunLoop（主线程除外）。

##### 3.RunLoop对外接口

CFRunLoopRef
CFRunLoopModeRef
CFRunLoopSourceRef
CFRunLoopTimerRef
CFRunLoopObserverRef

<img src="RunLoop_0.png" alt="RunLoop_0" style="zoom: 50%;" />

#### 二、RunTime

##### 1.概念

RunTime动态创建类和对象、并进行消息的转发。

Runtime是OC的底层实现，对开发者来说使用其更像是OC的trick实现机制。

##### 2.消息发送

OC底层调用是通过`objc_mgsSend`实现，也就是OC底层就是用runtime实现的C语言代码

```objc
//Person.m
@implementation Person
-(void)run{
    NSLog(@"runing perfectly");
}
@end
//main.m
#import <objc/message.h>//引入objc_msgSend，还需要buildSetting关闭相关设定
………………………………
 Person *p = [Person new];
objc_msgSend(p, @selector(run));
```

甚至由于Runtime提供了通过类名获取类的函数，得到SEL的函数

```objc
objc_getClass(char * _Nonnull name);
sel_registerName(const char* _Nonnull str);
```

```objc
id p1 = objc_msgSend( objc_msgSend( objc_getClass("Person"),sel_registerName("alloc")),
                     sel_registerName("init") );
        objc_msgSend(p1, @selector(run));
```



##### 3.消息传递机制

在没有找到对应的类方法或实例方法时：

动态方法解析->快速消息转发->普通消息转发

##### 3.1动态方法解析

首先会调用`resolveInstanceMethod`（增加实例方法） 或 `resolveClassMethod`（增加类方法）让你添加方法的实现，如果添加并返回YES系统运行时就会重新启动一次消息发送。

利用

```objc
class_addMethod( Class _Nullable __unsafe_unretained cls, SEL _Nonnull name,IMP _Nonnull imp, const char * _Nullable types)
```

增加一个方法及具体实现，

第一个参数填`self`

第二个填追加函数的SEL（`@selector(methodname)` or `sel_registerName("methodname")`),

第三个参数需要一个IMP，可以使用`imp_implementationWithBlock(id _Nonnull block)`来实现，

第四个参数types，每个方法都有两个被隐藏的参数`self`（当前对象）,`_cmd`（当前方法） ，第二个第三个参数必须是`@` 和`:` ，这里选用一个没有参数和返回值的方法，所以为`V@:`

```objc
+(BOOL)resolveInstanceMethod:(SEL)sel{
    if (sel == sel_registerName("wahaha")) {
        class_addMethod(self, sel_registerName("wahaha"), imp_implementationWithBlock(^(){
            NSLog(@"wahaha");
        }), "v@:");
    }
    return YES;
}
```

##### 3.2 快速消息转发

实现了`- forwardingTargetForSelector:`方法，系统就会进入该方法继续处理消息，这个方法的作用是把之前没办法处理的消息转发给别的对象去处理：

```go
//返回一个对象继续处理消息
- (id)forwardingTargetForSelector:(SEL)aSelector{
    if (aSelector == sel_registerName("wahaha")) {
        return [Dog new];
    }
    return nil;
}
```

##### 3.3普通消息转发

如果上一步也没有对消息进行处理，会进入最后一步，这里涉及两个方法。首先调用`methodSignatureForSelector:`方法来获取函数的参数和返回值，如果返回为`nil`，程序会Crash掉，并抛出`unrecognized selector sent to instance`异常信息。如果返回一个函数签名，系统就会创建一个`NSInvocation`对象并调用`-forwardInvocation:`方法。我们同样在这里对之前的消息进行处理一次：

```objectivec
//返回方法签名
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector{
    if (aSelector == sel_registerName("wahaha")) {
        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }
    return [super methodSignatureForSelector:aSelector];
}

//转发消息
- (void)forwardInvocation:(NSInvocation *)anInvocation {
    SEL sel = anInvocation.selector;
    Dog *dog = [Dog new];
    if ([dog respondsToSelector:sel]) {
        [anInvocation invokeWithTarget:dog];
    }
}
```





#### 三、补充

##### 1. isa指针与superclass

isa指针为从实例指向本类的指针，而NSobject指向自己（本类为自身）

OC的类本质上也是对象，指向其元类

```objc
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class super_class                       OBJC2_UNAVAILABLE;  // 父类

    const char *name                        OBJC2_UNAVAILABLE;  // 类名
    long version                            OBJC2_UNAVAILABLE;  // 类的版本信息，默认为0
    long info                               OBJC2_UNAVAILABLE;  // 类信息，供运行期使用的一些位标识

    long instance_size                      OBJC2_UNAVAILABLE;  // 类的实例变量大小
    struct objc_ivar_list *ivars            OBJC2_UNAVAILABLE;  // 类的成员变量链表

    struct objc_method_list **methodLists   OBJC2_UNAVAILABLE;  // 方法定义的链表
    struct objc_cache *cache                OBJC2_UNAVAILABLE;  // 方法缓存

    struct objc_protocol_list *protocols    OBJC2_UNAVAILABLE;  // 协议链表
#endif

} OBJC2_UNAVAILABLE;
```



<img src="image-20200730160213538.png" alt="image-20200730160213538" style="zoom: 33%;" />
