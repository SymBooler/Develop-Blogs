<!-- Created by 张广路， 2021-09-22 16:20:00 -->
# Swift Concurrancy 的一些知识点
## async 标记的语句作为一个整体被调度，不同的task是可能在同一个线程执行的，这也是为什么协程不适合用锁，因为锁是以线程为单位，锁会阻塞线程。await会导致后面的代码被挂起。

### **1. async/await 是 Swift 5.5 引入的新语法，可以用来简化并发代码的编写。async 表示异步，await 表示等待。**
    Task 是执行单元，作为一个整体被调度器调度， 当async函数使用await调用时，当前task 被挂起，等待await返回结果后，task继续执行。
### **2. async 函数只能被async 函数调用或者在Task内使用。**
### **3. Task有两种初始化方式**
    * ```Task.init(priority:operation:)``` 创建一个新的Task，priority表示优先级，operation是async函数。一个Task可以包含多个子Task。子Task不会阻塞父Task的执行，子Task能够继承父Task的上下文环境
        - isCancelled: 用来判断当前Task是否被取消。
        - TaskLocal: 用来在当前Task内存储和获取数据。
        - priority, 除非显式设置，否则继承父Task的优先级。
    * ```Task.detached(priority:operation:)```创建一个不继承上下文的顶层Task
### **4. Task的生命周期：task被释放其内部代码不会立即释放。如果主动调用task.cancel()，则会取消task，task内的代码逻辑需要自己处理取消逻辑。**
   - 在父任务中，不能忘记等待子任务完成。
   - 当为子任务设置更高的优先级时，父任务的优先级会自动升级。
   - 取消父任务时，其每个子任务也会自动取消。
   - 任务本地值会自动有效地传播到子任务。
### **5. DiscardingTaskGroup：只关心task执行状态，不关心task的结果，子task执行完毕后，会自动释放结果，所以无法获取结果。**
### **6. ```withTaskCancellationHandler {} operation: {}``` 用来处理task取消的情况，operation是async函数。当task被取消时，会调用handler。**
### **7. Concurrency中的数据安全保护方式：**
      * 实现Sendable协议，使对象可以安全地在线程间传递。
      * Actor MailBox模式，保证消息的顺序性。
      * 使用Atomic包装器，保证线程安全。

### **8. Executor：**
    * TaskExecutor：
        
        TaskExecutor是一个协议，定义了任务执行的基本行为。它负责将任务（ExecutorJob）放入一个执行队列中，以便异步执行。
        TaskExecutor可以是并行的，意味着它可以同时执行多个任务。这适用于需要提高并行度和吞吐量的场景。
        TaskExecutor不保证任务的执行顺序，因为它可能会在多个线程上并发执行任务。
    * SerialExecutor：

        SerialExecutor是TaskExecutor的一个子协议，它确保任务按照它们被添加的顺序一个接一个地执行。
        SerialExecutor是串行的，意味着即使在多线程环境中，它也会保证任务的执行顺序，这适用于需要保证任务执行顺序的场景，比如UI更新或者需要按特定顺序执行的任务。
        SerialExecutor通常用于确保对共享资源的访问是线程安全的，因为它可以避免并发访问导致的问题。

