1. isKindOfClass 和 isMemberOfClass 区别
答： isKindOfClass来确定一个对象是否是一个类的成员，或者是派生自该类的成员 isMemberOfClass只能确定一个对象是否是当前类的成员 例如：我们已经成NSObject派生了自己的类，isMemberOfClass不能检测任何的类都是基于NSObject类这一事实，而isKindOfClass可以
2. +load和+initialize对比
   1. 调用时机：
    +load是在Runtime加载类、分类的时候调用（只会调用一次），在main函数之前。
    +initialize是在类第一次接收到消息的时候调用，只会初始化一次（父类的+initialize可能会被调用多次），在main函数之后。
   2. 调用方式：
    +load是根据函数地址直接调用。
    +initialize是通过objc_msgSend调用。
3. msg_send 过程
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
