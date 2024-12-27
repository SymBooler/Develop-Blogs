1. 如果存在缓存&&缓存中有imp,直接返回imp
   ```objc
      // 首先从缓存中查找,如果找到了imp,直接返回 
      // Optimistic cache lookup 
      if (cache) { imp = cache_getImp(cls, sel); if (imp) return imp; }
   ```
   
2. 缓存中没有找到imp,尝试从本类方法列表中获取imp,其先把imp缓存起来,然后返回imp:
   ```objc
    // 从本类的方法列表中尝试查找IMP
    // Try this class's method lists.
    {
        Method meth = getMethodNoSuper_nolock(cls, sel);
        if (meth) {
            // 缓存imp log_and_fill_cache(cls, meth->imp, sel, inst, cls); imp = meth->imp; goto done;
        }
    }
    ```
3. 如果本类中没有找到imp,则
    1. 从父类继承链方法缓存中查找imp,找到直接返回imp;
    2. 如果从分类继承链中没有找到imp,则从父类继承链的方法列表中查找imp,找到后先缓存imp然后返回imp;
        ```objc
            // 从父类的方法列表中尝试查找IMP
        // Try superclass caches and method lists.
            {
                unsigned attempts = unreasonableClassCount();
                // 沿着继承链查找
            for (Class curClass = cls->superclass; curClass != nil; curClass = curClass->superclass) {
            // Halt if there is a cycle in the superclass chain.
                if (--attempts == 0) { _objc_fatal("Memory corruption in class list."); }
            // 从父类缓存中查找imp
            // Superclass cache.
                imp = cache_getImp(curClass, sel);
                if (imp) { if (imp != (IMP)_objc_msgForward_impcache) {
                // Found the method in a superclass. Cache it in this class.
                    log_and_fill_cache(cls, imp, sel, inst, curClass); goto done;
                } else {
                    // Found a forward:: entry in a superclass.
                    // Stop searching, but don't cache yet; call method
                    // resolver for this class first. break;
                    }
                }
            // 从父类方法列表中查找imp
            // Superclass method list.
                Method meth = getMethodNoSuper_nolock(curClass, sel);
                    if (meth) {
                    log_and_fill_cache(cls, meth->imp, sel, inst, curClass);
                    imp = meth->imp;
                    goto done;
                    }
                }
            }
        ```
   3. 如果以上尝试都没有找到imp,则尝试一次_class_resolveMethod(动态方法解析),把以上3步再走一遍:
        ```objc
            // 如果以上过程都没有找到,尝试一次动态方法解析
            // No implementation found. Try method resolver once.
            if (resolver && !triedResolver) {
                runtimeLock.unlock();
                // 动态方法解析
                _class_resolveMethod(cls, sel, inst);
                runtimeLock.lock();
                // Don't cache the result; we don't hold the lock so it may have
                // changed already. Re-do the search from scratch instead.
                triedResolver = YES;
                goto retry;
            }
        ```
    4. 如果动态方法解析也没有返回imp,则直接启动消息转发并会缓存转发的imp:
        ```objc
            // 如果方法解析也没有IMP,启动消息转发
            // No implementation found, and method resolver didn't help.
            // Use forwarding.
            imp = (IMP)_objc_msgForward_impcache;
            // 缓存imp
            cache_fill(cls, sel, imp, inst);
        ```
    查找imp的过程相对简单,苹果为了提高性能添加了缓存功能.如果没有找到相应的函数实现指针,则会通过动态方法解析和消息转发来给objc_msgSend()提供一些”补救措施”

消息转发
    在Person类中声明了一个- age()方法,并没有实现,运行一下程序:
```objc
        Person *person = [[Person alloc]init];
        [person age];
```
- age方法我们没有添加实现,其调用栈,走到了IMP lookUpImpOrForward()查找imp的动态方法解析中,也就是我们上面提到的第5步:

可以得出初步结论,如果我们没有给提个声明的方法添加实现,动态方法解析会被调用.最终程序走到了static void _objc_terminate(void){}函数里面,抛出一个常见的错误:
```objc
2019-04-25 11:49:55.039691+0800 RuntimeDylibTest[6426:102439] *** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[Person age]: unrecognized selector sent to instance 0x100e8e5b0' *** First throw call stack: ( 0 CoreFoundation 0x00007fff44dbc43d __exceptionPreprocess + 256 1 libobjc.A.dylib 0x000000010036c64f objc_exception_throw + 47 2 CoreFoundation 0x00007fff44e39255 -[NSObject(NSObject) __retain_OA] + 0 3 CoreFoundation 0x00007fff44d5bad0 ___forwarding___ + 1486 4 CoreFoundation 0x00007fff44d5b478 _CF_forwarding_prep_0 + 120 5 RuntimeDylibTest 0x0000000100000c9d main + 301 6 libdyld.dylib 0x00007fff71d9c085 start + 1 7 ??? 0x0000000000000001 0x0 + 1 ) libc++abi.dylib: terminating with uncaught exception of type NSException
```
这个错误意思是我们向一个对象(person)发送了一个不能识别的消息.通常这种错误,苹果给我提供了三种补救措施.这就是接下来要讲的内容–消息转发.之前我们讲过一个sel对应一个imp,而这个imp就是函数实现指针,正常的消息发送就是找到了sel对应的imp,而消息转发就是是正常消息调用过程遇到问题了没有找到与之对应的imp而启动转发流程的过程,尝试将调用的sel转发到我们指定的imp的过程

## 动态方法解析resolveMethod
在运行时系统中正常消息发送如果没有找到imp,会默认走一次动态方法解析.在NSObject.h文件中,关于动态方法解析的接口有两个方法
```objc
// MARK: - 动态方法解析 
+ (BOOL)resolveClassMethod:(SEL)sel OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0); 
+ (BOOL)resolveInstanceMethod:(SEL)sel OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);
```
其在NSObject.mm的实现默认返回值是false,即在没有找到sel对应的imp时则会继续执行转发流程直到找不到imp报错,如果我们给没有实现的sel添加了一个imp并且返回true时,则最终方法调用会走我们添加的imp.
```objc
// MARK: - 动态方法解析 
+ (BOOL)resolveClassMethod:(SEL)sel { return NO; }
+ (BOOL)resolveInstanceMethod:(SEL)sel { return NO; }
```
如果我们自己重写了这个两个方法,并返回true,就可以给未实现的方法添加一个实现imp.
```objc
/**

添加一个newAge

@param obj obj

@param _cmd 当前方法的sel

*/

void newAge (id obj, SEL _cmd) {
    NSLog(@"added newAge method:%@",NSStringFromSelector(_cmd));
}
/**

动态方法解析

@param sel 不能识别的sel

@return 消息是否处理了

*/
+ (BOOL)resolveInstanceMethod:(SEL)sel { 
    if (sel == @selector(age)) { 
            class_addMethod([self class], @selector(age), (IMP)newAge, "V@:"); 
            return true; 
        } 
        return [super resolveInstanceMethod:sel];
}
```
运行时一些类型编码符合V表示void;@表示id或者obj对象;:表示SEL;_cmd表示当前方法的sel.其他的可以看苹果接口文档).

可以看到调用-age方法时走到我们添加的imp里面了.

## 快速消息转发(重定向) fast forwarding
当我们没有重写动态方法解析或者没有未实现sel添加imp时,则会启动快速消息转发流程.快速消息转发需要重写系统提供的一个方法:

```objc
- (id)forwardingTargetForSelector:(SEL)aSelector OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0); 
- (id)forwardingTargetForSelector:(SEL)sel { return nil; }
```
其实现在NSObject.mm文件中,快速消息转发可以将某个没有实现的sel转发给某个对象obj,让这个对象来处理这个消息.其过程也是消息发送和转发的全过程,如果在obj所在的类中也没有找到sel对应的imp则会报错. 这里我们将-age方法转发给NSObject实例,因为NSObject类中并没有age方法的实现,所以直接报错.
```objc
// MARK: - 快速消息转发 
- (id)forwardingTargetForSelector:(SEL)aSelector { 
    if (aSelector == @selector(age)) { 
        return [[NSObject alloc]init]; 
    } 
    return [super forwardingTargetForSelector:aSelector]; 
}
2019-04-25 15:12:53.252362+0800 RuntimeDylibTest[18037:222826] *** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[NSObject age]: unrecognized selector sent to instance 0x100e622e0' *** First throw call stack: ( 0 CoreFoundation 0x00007fff44dbc43d __exceptionPreprocess + 256 1 libobjc.A.dylib 0x000000010036c64f objc_exception_throw + 47 2 CoreFoundation 0x00007fff44e39255 -[NSObject(NSObject) __retain_OA] + 0 3 CoreFoundation 0x00007fff44d5bad0 ___forwarding___ + 1486 4 CoreFoundation 0x00007fff44d5b478 _CF_forwarding_prep_0 + 120 5 RuntimeDylibTest 0x0000000100000d8d main + 237 6 libdyld.dylib 0x00007fff71d9c085 start + 1 7 ??? 0x0000000000000001 0x0 + 1 )
```
当我们新创建一个NewPerson类,并添加一个- age方法,然后将快速消息转发的方法返回值给NewPerson实例.可以看到消息被正常处理了:
```objc
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if (aSelector == @selector(age)) {
        return [[NewPerson alloc]init];
    }
    return [super forwardingTargetForSelector:aSelector];
}
2019-04-25 15:21:34.094385+0800 RuntimeDylibTest[18083:227346] new person age
```

## 正常消息转发 Normal forward
当快速消息转发方法返回值为nil或者self时,将启动下一层级的消息转发流程也就是正常消息转发.正常消息转发可以说是给一个sel补救的最后措施,也是任何不能识别sel的转发中心,通常我们可以在NSObject的分类里面将不能识别的消息汇总在一起,统一处理.这里系统为我们提供了两个必现实现的方法:
```
- (void)forwardInvocation:(NSInvocation *)anInvocation OBJC_SWIFT_UNAVAILABLE(""); 
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector OBJC_SWIFT_UNAVAILABLE("");
```
  NSInvocation是一个消息调用类,它包含了所有OC消息的成分:target,SEL,参数,返回值等.NSInvocation可以将消息转换成一个对象,消息的每一个参数能够直接设定,而当一个NSInvocation对象调度时返回值是可以自己设定的,一个NSInvocation对象能够重复调度不同target,而且他的sel也能设置为另一个方法签名.NSInvocation遵守NSCoding协议，但是仅支持NSPortCoder编码，不支持归档型操作。

使用步骤
1. 根据方法创建签名对象(NSMethodSignature对象)
2. 根据签名对象创建调用对象(NSInvocation对象)
3. 设置调用对象(NSInvocation对象)的相关信息
4. 调用方法
5. 获取方法返回值
在正常消息转发过程中,重写- (NSMethodSignature )methodSignatureForSelector:(SEL)aSelector;
方法返回方法签名就是用于生产NSInvocation对象,然后我们就可以在- (void)forwardInvocation:(NSInvocation )anInvocation;
方法中截货具体消息的发送对象target,选择子sel等信息,然后统一处理.
```
// MARK: - 正常消息转发
- (void)forwardInvocation:(NSInvocation *)anInvocation { 
    SEL sel = anInvocation.selector; 
    if (sel == @selector(age)) {
        NSLog(@"umimplementation age"); 
    } else {
        NSLog(@"give user a hint"); 
    }
}
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector { 
    NSMethodSignature *methodSignature = [super methodSignatureForSelector:aSelector];
    if (!methodSignature) {
        methodSignature = [NSMethodSignature signatureWithObjCTypes:"v@:*"];
    }
    return methodSignature;
}
```
# 总结
消息是Runtime核心的问题之一,消息发送和转发的其实是对imp的查找处理过程,对这些过程了解后应该对实际开发遇到的问题及解决会有较大帮助