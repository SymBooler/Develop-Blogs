我们知道iOS程序的入口函数在main.其实mian只是苹果给我们的”直观能够感受”的入口,在执行main之前,编译器已经帮我们做了相当多的事情.具体可以参考objc-os.h文件.Objective-C的Runtime库也是在main之前创建好的.我们关注sel_init()

SEL:
```objc
/*********************************************************************** 
* _objc_init 
* Bootstrap initialization. Registers our image notifier with dyld. 
* Called by libSystem BEFORE library initialization time 
**********************************************************************/

void _objc_init(void) { 
    static bool initialized = false; 
    if (initialized) return; 
    initialized = true;

    // fixme defer initialization until an objc-using image is found? 
    environ_init(); 
    tls_init(); 
    static_init(); 
    lock_init(); 
    exception_init();

    _dyld_objc_notify_register(&map_images, load_images, unmap_image); 
}
```

接口我们可以到sel_init()调用栈:
```
_|   _objc_init()
  _|   _dyld_objc_notify_register
    _|    map_images_nolock()
       _|   sel_init()
```

在map_images_nolock()方法中我们看到在sel_init()下面有arr_init():

```
void arr_init(void)
{
    AutoreleasePoolPage::init();
    SideTableInit();
}
```
这个函数就是我们熟悉的AutoreleasePoolPage的初始化和全局SideTable的初始化,这个以后再分析.这里我们看一下sel_init()所做的工作:
```
/***********************************************************************
* sel_init
* Initialize selector tables and register selectors used internally. **********************************************************************/
void sel_init(size_t selrefCount)
{
    // save this value for later
    SelrefCount = selrefCount;

#if SUPPORT_PREOPT
    builtins = preoptimizedSelectors();

    if (PrintPreopt && builtins) {
        uint32_t occupied = builtins->occupied; 
        uint32_t capacity = builtins->capacity;

        _objc_inform("PREOPTIMIZATION: using selopt at %p", builtins); 
        _objc_inform("PREOPTIMIZATION: %u selectors", occupied);
        _objc_inform("PREOPTIMIZATION: %u/%u (%u%%) hash table occupancy", occupied, capacity, (unsigned)(occupied/(double)capacity*100));
    }
#endif

    // Register selectors used by libobjc 
    #define s(x) SEL_##x = sel_registerNameNoLock(#x, NO)
    #define t(x,y) SEL_##y = sel_registerNameNoLock(#x, NO)

    mutex_locker_t lock(selLock); 
    s(load); 
    s(initialize); 
    t(resolveInstanceMethod:, resolveInstanceMethod); 
    t(resolveClassMethod:, resolveClassMethod); 
    t(.cxx_construct, cxx_construct); 
    t(.cxx_destruct, cxx_destruct); 
    s(retain); 
    s(release); 
    s(autorelease); 
    s(retainCount); 
    s(alloc); 
    t(allocWithZone:, allocWithZone); 
    s(dealloc); 
    s(copy); 
    s(new); 
    t(forwardInvocation:, forwardInvocation); 
    t(_tryRetain, tryRetain); 
    t(_isDeallocating, isDeallocating); 
    s(retainWeakReference); 
    s(allowsWeakReference); 
    #undef s 
    #undef t
}
```
我们可以看到一些常见系统内置方法分别调用__sel_registerName()这个方法:
```objc
static SEL __sel_registerName(const char *name, bool shouldLock, bool copy)
{
    SEL result = 0;

    if (shouldLock) selLock.assertUnlocked();
    else selLock.assertLocked();

    if (!name) return (SEL)0;|

    result = search_builtins(name);
    if (result) return result;

    conditional_mutex_locker_t lock(selLock, shouldLock);
    if (namedSelectors) {
        result = (SEL)NXMapGet(namedSelectors, name);
    }
    if (result) return result;

    // No match. Insert.

    if (!namedSelectors) { 
        namedSelectors = NXCreateMapTable(NXStrValueMapPrototype, (unsigned)SelrefCount); 
    }
    if (!result) {
        // 初始化sel_alloc
        result = sel_alloc(name, copy);

        // 将selector 插入到NXMapTable 表中
        // fixme choose a better container (hash not map for starters)
        NXMapInsert(namedSelectors, sel_getName(result), result);
    }

    return result;
}
```
这方法的作用是将selector注册到NXMapTable表中,如果selector不存在则调用selector初始化方法,然后将selecto name作为key,selector作为value存到NXMapTable哈希表中:
```objc
static SEL sel_alloc(const char *name, bool copy) { 
    selLock.assertLocked(); 
    return (SEL)(copy ? strdupIfMutable(name) : name);
}
```
在objc.h文件中我们可以看到SEL的声明:
```objc
/// An opaque type that represents a method selector.
typedef struct objc_selector *SEL;
```
至此我们可以理解了,SEL就是一个表示方法的selector指针,映射方法的名字.
# IMP:
```objc
/// A pointer to the function of a method implementation.
#if !OBJC_OLD_DISPATCH_PROTOTYPES
typedef void (*IMP)(void /* id, SEL, ... */ );
#else
typedef id _Nullable (*IMP)(id _Nonnull, SEL _Nonnull, ...);
#endif
```
从定义来看,IMP是一个指向实现函数的指针.IMP也是实现函数的入口,其和SEL的关系等以后将消息发送在细说.
# Method:
method的声明结构:
```objc
typedef struct method_t *Method;
```
继续查看method_t的定义:
```objc
struct method_t {
    SEL name;
    const char *types;
    MethodListIMP imp;

    struct SortBySELAddress : public std::binary_function<const method_t&, const method_t&, bool> {
        bool operator() (const method_t& lhs, const method_t& rhs) {
            return lhs.name < rhs.name; 
        }
    }
```
method_t中有两个我们熟悉的成员变量:SEL和MethodListIMP,看一下MethodListIMP:
```objc
#if __has_feature(ptrauth_calls)
// Method lists use process-independent signature for compatibility. 
// Method caches use process-dependent signature for extra protection.
//   (fixme not yet __ptrauth(...) because of `stp` inline asm in objc-cache.mm)
using MethodListIMP = IMP __ptrauth_objc_method_list_imp; 
using MethodCacheIMP = StorageSignedFunctionPointer<IMP, ptrauth_key_process_dependent_code>;

#else

using MethodListIMP = IMP;
using MethodCacheIMP = IMP;

#endif
```
MethodListIMP其实就是IMP,method可以理解为SEL(方法名称)和IMP(方法实现)相互对应的集合体.正常的情况一个SEL对应一个IMP,而SEL和IMP的绑定到运行时才确定的.

# 相关API
下面是NSObject.h和runtime.h文件为我们提供的有关SEL,IMP和Method相关的接口及说明:
### runtime.h文件:
### 根据SEL获取实例Method指针

```objc
// 获取Method声明 
OBJC_EXPORT Method _Nullable 
class_getInstanceMethod(Class _Nullable cls, SEL _Nonnull name) OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0, 2.0);

// 获取Method实现
/***********************************************************************
* class_getInstanceMethod.  Return the instance method for the
* specified class and selector.
**********************************************************************/ 
Method class_getInstanceMethod(Class cls, SEL sel)
{
    if (!cls  ||  !sel) return nil;

    // This deliberately avoids +initialize because it historically did so.

    // This implementation is a bit weird because it's the only place that
    // wants a Method instead of an IMP.

    #warning fixme build and search caches
    // Search method lists, try method resolver, etc.
    lookUpImpOrNil(cls, sel, nil, NO/*initialize*/, NO/*cache*/, YES/*resolver*/);

    #warning fixme build and search caches

    return _class_getMethod(cls, sel);
}
```
这里面调用了_class_getMethod()私有函数:
```objc
/***********************************************************************
* _class_getMethod
* fixme
* Locking: read-locks runtimeLock
**********************************************************************/ 
static Method _class_getMethod(Class cls, SEL sel)
{
    mutex_locker_t lock(runtimeLock);
    return getMethod_nolock(cls, sel);
}

/***********************************************************************
* getMethod_nolock
* fixme
* Locking: runtimeLock must be read- or write-locked by the caller
**********************************************************************/
static method_t *
getMethod_nolock(Class cls, SEL sel)
{
    method_t *m = nil;

    runtimeLock.assertLocked();

    // fixme nil cls?
    // fixme nil sel?

    assert(cls->isRealized());

    while (cls  &&  ((m = getMethodNoSuper_nolock(cls, sel))) == nil) {
        cls = cls->superclass;
    }

    return m;
}
```
这个函数内部调用的是getMethodNoSuper_nolock():

```objc
static method_t *
getMethodNoSuper_nolock(Class cls, SEL sel)
{
     runtimeLock.assertLocked();

    assert(cls->isRealized());
    // fixme nil cls?
    // fixme nil sel?

    for (auto mlists = cls->data()->methods.beginLists(), end = cls->data()->methods.endLists(); mlists != end; ++mlists) {
        method_t *m = search_method_list(*mlists, sel);
        if (m) return m;
    }

    return nil;
}
```
查找方法的过程是先从本class的方法列表中查看是否存在,不存再在看父类递归这个过程( cls = cls->superclass).这个我们在以后讲消息发送,转发时在细看.

# 根据SEL获取类Method指针
```objc
OBJC_EXPORT Method _Nullable class_getClassMethod(Class _Nullable cls, SEL _Nonnull name) OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0, 2.0);
```
类对象的方法列表存放在元类中,所以获取类方法要去元类中查找,其在获取的时候参数已经指明元类:

```objc
/*********************************************************************** 
* class_getClassMethod. Return the class method for the specified
* class and selector.
**********************************************************************/
Method class_getClassMethod(Class cls, SEL sel)
{
    if (!cls  ||  !sel) return nil;
    //这里cls获取的是元类
    return class_getInstanceMethod(cls->getMeta(), sel);
}
```
返回一个函数的实现指针:
```objc
OBJC_EXPORT IMP _Nullable class_getMethodImplementation(Class _Nullable cls, SEL _Nonnull name) OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);
```
其实现过程:
```objc
IMP class_getMethodImplementation(Class cls, SEL sel)
{
    IMP imp;
    if (!cls || !sel) return nil; 
    imp = lookUpImpOrNil(cls, sel, nil, YES/*initialize*/, YES/*cache*/, YES/*resolver*/);

// Translate forwarding function to C-callable external version
    if (!imp) {
        return _objc_msgForward;
    }
    return imp;
}
```
其内部我们可以看到两个主要的调用函数:lookUpImpOrNil()和_objc_msgForward.前面函数是获取SEL对应的IMP,其实现过程如下:
```objc
/*********************************************************************** 
* lookUpImpOrNil. 
* Like lookUpImpOrForward, but returns nil instead of _objc_msgForward_impcache **********************************************************************/
IMP lookUpImpOrNil(Class cls, SEL sel, id inst,
                   bool initialize, bool cache, bool resolver)
{
    IMP imp = lookUpImpOrForward(cls, sel, inst, initialize, cache, resolver); 
    if (imp == _objc_msgForward_impcache) return nil; 
    else return imp; 
}

// MARK: - 获取imp,先从缓存中查找imp,如果存在直接返回imp.
/***********************************************************************
* lookUpImpOrForward.
* The standard IMP lookup.
* initialize==NO tries to avoid +initialize (but sometimes fails)
* cache==NO skips optimistic unlocked lookup (but uses cache elsewhere)
* Most callers should use initialize==YES and cache==YES.
* inst is an instance of cls or a subclass thereof, or nil if none is known. 
* If cls is an un-initialized metaclass then a non-nil inst is faster. 
* May return _objc_msgForward_impcache. IMPs destined for external use 
* must be converted to _objc_msgForward or _objc_msgForward_stret. 
* If you don't want forwarding at all, use lookUpImpOrNil() instead.
**********************************************************************/
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, bool initialize, bool cache, bool resolver)
{
    IMP imp = nil;
    bool triedResolver = NO;

    runtimeLock.assertUnlocked();

    // Optimistic cache lookup
    if (cache) {
        imp = cache_getImp(cls, sel);
        if (imp) return imp;
    }

// runtimeLock is held during isRealized and isInitialized checking 
// to prevent races against concurrent realization.

// runtimeLock is held during method search to make 
// method-lookup + cache-fill atomic with respect to method addition. 
// Otherwise, a category could be added but ignored indefinitely because 
// the cache was re-filled with the old value after the cache flush on 
// behalf of the category.

runtimeLock.lock();
checkIsKnownClass(cls);

if (!cls->isRealized()) {
    realizeClass(cls);
}

if (initialize && !cls->isInitialized()) { 
    runtimeLock.unlock(); 
    _class_initialize (_class_getNonMetaClass(cls, inst));
    runtimeLock.lock(); 
    // If sel == initialize, _class_initialize will send +initialize and 
    // then the messenger will send +initialize again after this 
    // procedure finishes. Of course, if this is not being called 
    // from the messenger then it won't happen. 2778172 
}

retry:
    runtimeLock.assertLocked();

// 从缓存中尝试查找IMP
// Try this class's cache.

imp = cache_getImp(cls, sel);
if (imp) goto done;

// 从本类的方法列表中尝试查找IMP
// Try this class's method lists.

{
    Method meth = getMethodNoSuper_nolock(cls, sel);
    if (meth) { 
        log_and_fill_cache(cls, meth->imp, sel, inst, cls); 
        imp = meth->imp; goto done;
    }
}

// 从父类的方法列表中尝试查找IMP
// Try superclass caches and method lists.
{
    unsigned attempts = unreasonableClassCount();
    for (Class curClass = cls->superclass; curClass != nil; curClass = curClass->superclass) {
        // Halt if there is a cycle in the superclass chain. 
        if (--attempts == 0) { 
            _objc_fatal("Memory corruption in class list."); 
        }

        // Superclass cache.
        imp = cache_getImp(curClass, sel);
        if (imp) {
            if (imp != (IMP)_objc_msgForward_impcache) {
                // Found the method in a superclass. Cache it in this class. 
                log_and_fill_cache(cls, imp, sel, inst, curClass); 
                goto done; 
            }
            else { 
                // Found a forward:: entry in a superclass. 
                // Stop searching, but don't cache yet; call method 
                // resolver for this class first. 
                break; 
            }
        }

        // Superclass method list.
        Method meth = getMethodNoSuper_nolock(curClass, sel);
        if (meth) { 
            log_and_fill_cache(cls, meth->imp, sel, inst, curClass); 
            imp = meth->imp; 
            goto done; 
        }
    }
}

// 如果以上过程都没有找到,尝试一次动态方法解析
// No implementation found. Try method resolver once.

if (resolver  &&  !triedResolver) {
    runtimeLock.unlock();
    _class_resolveMethod(cls, sel, inst);
    runtimeLock.lock();
    // Don't cache the result; we don't hold the lock so it may have 
    // changed already. Re-do the search from scratch instead.
    triedResolver = YES;
    goto retry;
}

// 如果方法解析也没有IMP,启动消息转发
// No implementation found, and method resolver didn't help.
// Use forwarding.

imp = (IMP)_objc_msgForward_impcache;
cache_fill(cls, sel, imp, inst);

done:
    runtimeLock.unlock();

    return imp;
}
```
根据SEL在缓存中如果没有找到IMP,则在本类的Method获取找IMP,如果没有找到去父类中查找.以上过程都没有找到IMP的话,启动一次方法解析,方法解析也没有IMP的话就启动消息转发.

### 一个对象是否响应某个方法

```objc
// MARK: - 一个实例对象是否响应某个方法 
/** 
* Returns a Boolean value that indicates whether instances of a class respond to a particular selector. 
* 
* @param cls The class you want to inspect. 
* @param sel A selector. 
* 
* @return \c YES if instances of the class respond to the selector, otherwise \c NO. 
* 
* @note You should usually use \c NSObject's \c respondsToSelector: or \c instancesRespondToSelector: 
* methods instead of this function. 
*/

OBJC_EXPORT BOOL class_respondsToSelector(Class _Nullable cls, SEL _Nonnull sel) OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);

// MARK: - 是否响应某个方法 
BOOL class_respondsToSelector(Class cls, SEL sel) { 
    return class_respondsToSelector_inst(cls, sel, nil); 
}

// MARK: - 是否响应某个方法,其内部是通过获取IMP是否存在来判断 
// inst is an instance of cls or a subclass thereof, or nil if none is known. 
// Non-nil inst is faster in some cases. See lookUpImpOrForward() for details.

bool class_respondsToSelector_inst(Class cls, SEL sel, id inst)
{
    IMP imp;

    if (!sel  ||  !cls) return NO;

    // Avoids +initialize because it historically did so.
    // We're not returning a callable IMP anyway.

    imp = lookUpImpOrNil(cls, sel, inst, NO/*initialize*/, YES/*cache*/, YES/*resolver*/); 
    return bool(imp); 
}
```
其最终还是痛SEL是否有对应IMP来判断对象能否响应某个方法.

### 获取一个类的方法列表
```objc
// MARK: - 获取一个类的方法列表 
/** 
* Describes the instance methods implemented by a class. 
* 
* @param cls The class you want to inspect. 
* @param outCount On return, contains the length of the returned array. 
* If outCount is NULL, the length is not returned. 
* 
* @return An array of pointers of type Method describing the instance methods 
* implemented by the class—any instance methods implemented by superclasses are not included. 
* The array contains *outCount pointers followed by a NULL terminator. You must free the array with free(). 
* 
* If cls implements no instance methods, or cls is Nil, returns NULL and *outCount is 0. 
* 
* @note To get the class methods of a class, use \c class_copyMethodList(object_getClass(cls), &count). 
* @note To get the implementations of methods that may be implemented by superclasses, 
* use \c class_getInstanceMethod or \c class_getClassMethod. 
*/

OBJC_EXPORT Method _Nonnull * _Nullable class_copyMethodList(Class _Nullable cls, unsigned int * _Nullable outCount) OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);

// MARK: - 获取一个类所有实现的方法列表 
/*********************************************************************** 
* class_copyMethodList 
* fixme 
* Locking: read-locks runtimeLock 
**********************************************************************/
Method *
class_copyMethodList(Class cls, unsigned int *outCount)
{
    unsigned int count = 0;
    Method *result = nil;

    if (!cls) {
        if (outCount) *outCount = 0;
        return nil;
    }

    mutex_locker_t lock(runtimeLock);

    assert(cls->isRealized());

    count = cls->data()->methods.count();

    if (count > 0) {
        result = (Method *)malloc((count + 1) * sizeof(Method));

        count = 0;
        for (auto& meth : cls->data()->methods) {
            result[count++] = &meth;
        }
        result[count] = nil;
    }

    if (outCount) *outCount = count;
    return result;
}
```
### 获取一个方法名称
```
// MARK: - 获取一个方法名称 
/** 
* Returns the name of a method. 
* 
* @param m The method to inspect. 
* 
* @return A pointer of type SEL. 
* 
* @note To get the method name as a C string, call \c sel_getName(method_getName(method)). 
*/

OBJC_EXPORT SEL _Nonnull method_getName(Method _Nonnull m) OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);

// MARK: - 获取一个方法名称 
/*********************************************************************** 
* method_getName 
* Returns this method's selector. 
* The method must not be nil. 
* The method must already have been fixed-up. 
* Locking: none 
**********************************************************************/

SEL method_getName(Method m) {
    if (!m) return nil;

    assert(m->name == sel_registerName(sel_getName(m->name)));
    return m->name;
}
```
获取一个方法名字最终还是通过sel_registerName()来获取,也是我们最开始的部分sel_init()提到.

# 获取一个方法的IMP
```objc
// MARK: - 获取一个方法的实现 
/** 
* Returns the implementation of a method. 
* 
* @param m The method to inspect. 
* 
* @return A function pointer of type IMP. 
*/

OBJC_EXPORT IMP _Nonnull method_getImplementation(Method _Nonnull m) OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);

// MARK: - 获取一个方法的IMP
IMP method_getImplementation(Method m) {
    return m ? m->imp : nil;
}
```
通过直接Method中的imp成员变量
# 获取一个方法的参数和返回值类型
```objc
// MARK: - 获取一个方法的参数和返回值类型 
/** 
* Returns a string describing a method's parameter and return types. 
* 
* @param m The method to inspect. 
* 
* @return A C string. The string may be \c NULL. 
*/
OBJC_EXPORT const char * _Nullable method_getTypeEncoding(Method _Nonnull m) OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);

// MARK: - 获取一个方法的参数和返回值类型 
/*********************************************************************** 
* method_getTypeEncoding 
* Returns this method's old-style type encoding string. 
* The method must not be nil. 
* Locking: none 
**********************************************************************/
const char * method_getTypeEncoding(Method m) {
    if (!m) return nil;
    return m->types;
}
```
获取一个方法的参数数量
```objc
// MARK: - 获取一个方法参数数量 
/** 
* Returns the number of arguments accepted by a method. 
* 
* @param m A pointer to a \c Method data structure. Pass the method in question. 
* 
* @return An integer containing the number of arguments accepted by the given method. 
*/

OBJC_EXPORT unsigned int method_getNumberOfArguments(Method _Nonnull m) OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0, 2.0);

// MARK: - 获取一个方法的参数数量 
/*********************************************************************** 
* method_getNumberOfArguments. 
**********************************************************************/
unsigned int method_getNumberOfArguments(Method m) { 
    if (!m) return 0; 
    return encoding_getNumberOfArguments(method_getTypeEncoding(m)); 
}
```
获取一个方法的返回值类型
```objc
// MARK: - 获取一个方法的返回值类型 
/** 
* Returns a string describing a method's return type. 
* 
* @param m The method to inspect. 
* 
* @return A C string describing the return type. You must free the string with \c free(). 
*/
OBJC_EXPORT char * _Nonnull method_copyReturnType(Method _Nonnull m) OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);

// MARK: - 获取方法的返回值类型 
char * method_copyReturnType(Method m) { 
    return encoding_copyReturnType(method_getTypeEncoding(m)); 
}
```
获取方法某个参数的类型
```objc
// MARK: - 获取方法某个参数的类型 
/** 
* Returns a string describing a single parameter type of a method. 
* 
* @param m The method to inspect. 
* @param index The index of the parameter to inspect. 
* 
* @return A C string describing the type of the parameter at index \e index, or \c NULL 
* if method has no parameter index \e index. You must free the string with \c free(). 
*/
OBJC_EXPORT char * _Nullable method_copyArgumentType(Method _Nonnull m, unsigned int index) OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);

// MARK: - 获取方法某个参数类型 
char * method_copyArgumentType(Method m, unsigned int index) { 
    return encoding_copyArgumentType(method_getTypeEncoding(m), index); 
}
```

### 获取方法返回值类型
```objc
// MARK: - 获取方法返回值类型 
/** 
* Returns by reference a string describing a method's return type. 
* 
* @param m The method you want to inquire about. 
* @param dst The reference string to store the description. 
* @param dst_len The maximum number of characters that can be stored in \e dst. 
* 
* @note The method's return type string is copied to \e dst. 
* \e dst is filled as if \c strncpy(dst, parameter_type, dst_len) were called. 
*/

OBJC_EXPORT void method_getReturnType(Method _Nonnull m, char * _Nonnull dst, size_t dst_len) OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);

// MARK: - 获取方法返回值类型 
void method_getReturnType(Method m, char *dst, size_t dst_len) { 
    encoding_getReturnType(method_getTypeEncoding(m), dst, dst_len); 
}
```

### 更新设置某个方法的IMP
```objc
// MARK: - 更新设置某个方法的IMP 
/** 
* Sets the implementation of a method. 
* 
* @param m The method for which to set an implementation. 
* @param imp The implemention to set to this method. 
* 
* @return The previous implementation of the method. 
*/
OBJC_EXPORT IMP _Nonnull method_setImplementation(Method _Nonnull m, IMP _Nonnull imp) OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);

// MARK: - 更新设置某个方法IMP
IMP method_setImplementation(Method m, IMP imp) {
    // Don't know the class - will be slow if RR/AWZ are affected 
    // fixme build list of classes whose Methods are known externally? 
    mutex_locker_t lock(runtimeLock); 
    return _method_setImplementation(Nil, m, imp);
}

// MARK: - 更新设置某个方法的IMP 
/*********************************************************************** 
* method_setImplementation 
* Sets this method's implementation to imp. 
* The previous implementation is returned. 
**********************************************************************/
static IMP _method_setImplementation(Class cls, method_t *m, IMP imp) {
    runtimeLock.assertLocked();

    if (!m) return nil;
    if (!imp) return nil;

    IMP old = m->imp;
    m->imp = imp;

    // Cache updates are slow if cls is nil (i.e. unknown) 
    // RR/AWZ updates are slow if cls is nil (i.e. unknown) 
    // fixme build list of classes whose Methods are known externally?

    flushCaches(cls);

    updateCustomRR_AWZ(cls, m);

    return old;
}
```
先找到old imp,然后把old imp 替换成 new imp.

### 交换两个方法的实现即交换两个方法的IMP
```objc
// MARK: - 交换两个方法的实现即交换两个方法的IMP 
/** 
* Exchanges the implementations of two methods. 
* 
* @param m1 Method to exchange with second method. 
* @param m2 Method to exchange with first method. 
* 
* @note This is an atomic version of the following: * \code 
* IMP imp1 = method_getImplementation(m1); 
* IMP imp2 = method_getImplementation(m2); 
* method_setImplementation(m1, imp2); 
* method_setImplementation(m2, imp1); 
* \endcode 
*/

OBJC_EXPORT void method_exchangeImplementations(Method _Nonnull m1, Method _Nonnull m2) OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);

// MARK: - 交换两个方法的实现即交换两个方法的IMP
void method_exchangeImplementations(Method m1, Method m2) {
    if (!m1  ||  !m2) return;

    mutex_locker_t lock(runtimeLock);

    IMP m1_imp = m1->imp;
    m1->imp = m2->imp;
    m2->imp = m1_imp;

    // RR/AWZ updates are slow because class is unknown 
    // Cache updates are slow because class is unknown 
    // fixme build list of classes whose Methods are known externally?

    flushCaches(nil);

    updateCustomRR_AWZ(nil, m1);
    updateCustomRR_AWZ(nil, m2);
}
```

# NSObject.h文件
### 执行某个方法
执行某个方法,一般用于方法解析中动态添加方法,下面这几个方法功能类似:
```objc
// MARK: - 执行某个方法 一般用于方法解析中动态添加方法 
- (id)performSelector:(SEL)aSelector; 
- (id)performSelector:(SEL)aSelector withObject:(id)object; 
- (id)performSelector:(SEL)aSelector withObject:(id)object1 withObject:(id)object2;

// MARK: - 执行某个方法 一般用于方法解析中动态添加方法 
- (id)performSelector:(SEL)sel { 
    if (!sel) [self doesNotRecognizeSelector:sel]; 
    return ((id(*)(id, SEL))objc_msgSend)(self, sel); 
}

- (id)performSelector:(SEL)sel withObject:(id)obj { 
    if (!sel) [self doesNotRecognizeSelector:sel]; 
    return ((id(*)(id, SEL, id))objc_msgSend)(self, sel, obj);
}
```
先判断这个方法是否存在doesNotRecognizeSelector(),不存在报错”unrecognized selector sent to instance”.然后调用了message.h中的消息发送函数.
# 内省方法:是否响应某个方法
```objc
// MARK: - 内省方法:是否响应某个方法 - (BOOL)respondsToSelector:(SEL)aSelector; 
// MARK: - 是否响应某个方法 
- (BOOL)respondsToSelector:(SEL)sel { 
    if (!sel) return NO;
    return class_respondsToSelector_inst([self class], sel, self); 
}

// MARK: - 是否响应某个方法,其内部是通过获取IMP是否存在来判断 
// inst is an instance of cls or a subclass thereof, or nil if none is known. 
// Non-nil inst is faster in some cases. See lookUpImpOrForward() for details.

bool class_respondsToSelector_inst(Class cls, SEL sel, id inst)
{
    IMP imp; if (!sel || !cls) return NO;

    // Avoids +initialize because it historically did so. 
    // We're not returning a callable IMP anyway. 

    imp = lookUpImpOrNil(cls, sel, inst, NO/*initialize*/, YES/*cache*/, YES/*resolver*/); 
    return bool(imp); 
}
```
# 一个实例对象是否响应某个方法
```
// MARK: - 一个对象是否响应某个方法 
+ (BOOL)instancesRespondToSelector:(SEL)aSelector; 
// MARK: - 一个对象是否响应某个方法 
+ (BOOL)instancesRespondToSelector:(SEL)sel { 
    if (!sel) return NO; 
    return class_respondsToSelector(self, sel); 
}
```
# 获取一个SEL对应的IMP
```objc
// MARK: - 获取一个SEL对应的IMP 
- (IMP)methodForSelector:(SEL)aSelector; 
// MARK: - 获取一个SEL对应的IMP 
- (IMP)methodForSelector:(SEL)sel { 
    if (!sel) [self doesNotRecognizeSelector:sel]; 
    return object_getMethodImplementation(self, sel);
}
```
# 获取一个实例方法SEL对应的IMP
```objc
// MARK: - 获取一个实例方法SEL对应的IMP 
+ (IMP)instanceMethodForSelector:(SEL)aSelector; 
// MARK: - 获取一个实例方法SEL对应的IMP 
+ (IMP)instanceMethodForSelector:(SEL)sel { 
    if (!sel) [self doesNotRecognizeSelector:sel]; 
    return class_getMethodImplementation(self, sel); 
}
```
# 不能响应某个SEL
```objc
// MARK: - 不能响应某个SEL,报错 
- (void)doesNotRecognizeSelector:(SEL)aSelector; 
// MARK: - 不能响应某个方法 
// Replaced by CF (throws an NSException) 
- (void)doesNotRecognizeSelector:(SEL)sel { 
    _objc_fatal("-[%s %s]: unrecognized selector sent to instance %p", object_getClassName(self), sel_getName(sel), self); 
}
```
# 动态方法解析
动态方法解析,一般用来给某个没有实现的方法添加一个实现;返回false时,执行消息转发流程
```objc
// MARK: - 动态方法解析 
+ (BOOL)resolveClassMethod:(SEL)sel OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0); 
+ (BOOL)resolveInstanceMethod:(SEL)sel OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);

// MARK: - 动态方法解析
+ (BOOL)resolveClassMethod:(SEL)sel {
    return NO;
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    return NO;
}
```

# 消息转发相关
```objc

// MARK: - 消息转发相关
- (id)forwardingTargetForSelector:(SEL)aSelector OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);

- (void)forwardInvocation:(NSInvocation *)anInvocation OBJC_SWIFT_UNAVAILABLE(""); 
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector OBJC_SWIFT_UNAVAILABLE("");

+ (NSMethodSignature *)instanceMethodSignatureForSelector:(SEL)aSelector OBJC_SWIFT_UNAVAILABLE("");
```
本文主要介绍了SEL,IMP,Method的定义和实现,以及系统为我们提供的常用API.这里面涉及到到向一个对象发送消息的流程及转发过程,这个会在以后的章节中讲到.