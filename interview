1. isKindOfClass 和 isMemberOfClass 区别
答： isKindOfClass来确定一个对象是否是一个类的成员，或者是派生自该类的成员 isMemberOfClass只能确定一个对象是否是当前类的成员 例如：我们已经成NSObject派生了自己的类，isMemberOfClass不能检测任何的类都是基于NSObject类这一事实，而isKindOfClass可以
2. +load和+initialize对比
  1. 调用时机：
  +load是在Runtime加载类、分类的时候调用（只会调用一次），在main函数之前。
  +initialize是在类第一次接收到消息的时候调用，只会初始化一次（父类的+initialize可能会被调用多次），在main函数之后。
  2. 调用方式：
  +load是根据函数地址直接调用。
  +initialize是通过objc_msgSend调用。
