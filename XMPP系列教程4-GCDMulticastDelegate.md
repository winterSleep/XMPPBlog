# GCDMulticastDelegate

在XMPPFrameWork中, 发送广播时, 用的最多的方法, 就是使用 GCDMulticastDelegate 来发送广播, 我们在其它项目, 也可使用该类作为咱们自己的 NSNotificationCenter 替代方案.

## GCDMulticastDelegate类中的方法

	//添加一个监听者(delegate)到广播中, 并且指定回调时的queue(delegateQueue)
	- (void)addDelegate:(id)delegate delegateQueue:(dispatch_queue_t)delegateQueue;
	
	//将监听者(delegate)从广播队列(delegateQueue线程)中移除
	- (void)removeDelegate:(id)delegate delegateQueue:(dispatch_queue_t)delegateQueue;
	
	//将监听者(delegate)从广播队列中移除(所有线程)
	- (void)removeDelegate:(id)delegate;
	
	//移除所有监听者
	- (void)removeAllDelegates;
	
	//广播队列中的监听者数量
	- (NSUInteger)count;
	
	//某个 Class 类型的监听者数量
	- (NSUInteger)countOfClass:(Class)aClass;
	
	//某个方法的监听者数量
	- (NSUInteger)countForSelector:(SEL)aSelector;
	
	//监听者列表中, 是否有能响应方法(aSelector)的监听者
	- (BOOL)hasDelegateThatRespondsToSelector:(SEL)aSelector;
	
	//返回一个可以遍历的对象(可自己手动遍历监听者列表)
	- (GCDMulticastDelegateEnumerator *)delegateEnumerator;
	
### 初始化一个 GCDMulticastDelegate

.h头文件中定义一个内部变量
	
	@interface XMPPStream ()
	{
		...
		
		//XMPPStreamDelegate 为 multicastDelegate 遵从的协议, 方便更好的发送广播
		GCDMulticastDelegate <XMPPStreamDelegate> *multicastDelegate;
		...
	}

.m 文件中进行初始化

	multicastDelegate = (GCDMulticastDelegate <XMPPStreamDelegate> *)[[GCDMulticastDelegate alloc] init];
	
### 添加一个监听者到监听列表中

这里 addDelegate时的 delegate 对象与 multicastDelegate 是弱引用关系, 不用担心循环引用的问题.

	[multicastDelegate addDelegate:delegate delegateQueue:delegateQueue];
	
### 发送一个广播给所有监听者

循环 multicastDelegate 中的监听者列表, 查找出能响应xmppStreamConnectDidTimeout 方法的对象, 并调用监听者的 xmppStreamConnectDidTimeout 方法, 达到 NSNotificationCenter的类似功能.

	[multicastDelegate xmppStreamConnectDidTimeout:self]
	
### 从监听者列表中移除一个监听者

	- (void)removeDelegate:(id)delegate delegateQueue:(dispatch_queue_t)delegateQueue;
	
## 实现原理

首先, GCDMulticastDelegate 保存了一个 delegateNodes 对象, 用于保存监听者列表

	@interface GCDMulticastDelegate ()
	{
		NSMutableArray *delegateNodes;
	}
	
添加监听者时, 初始化一个 GCDMulticastDelegateNode 对象, 将 delegate 保存到 node 对象中, 然后将 node 对象添加到 delegateNodes 数组中, 如下:

	- (void)addDelegate:(id)delegate delegateQueue:(dispatch_queue_t)delegateQueue
	{
		if (delegate == nil) return;
		if (delegateQueue == NULL) return;
		
		//初始化一个 GCDMulticastDelegateNode 对象
		GCDMulticastDelegateNode *node =
		    [[GCDMulticastDelegateNode alloc] initWithDelegate:delegate delegateQueue:delegateQueue];
		
		//然后将 node 对象添加到 delegateNodes 数组
		[delegateNodes addObject:node];
	}

给监听者发送广播时比较复杂一点, 使用了 ["NSObject中forwardInvocation消息重定向"](http://blog.csdn.net/devday/article/details/7418022)

重写 methodSignatureForSelector 方法

	- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector

重写 forwardInvocation 方法

	- (void)forwardInvocation:(NSInvocation *)origInvocation{
		...
		
		//遍历 delegateNodes
		for (GCDMulticastDelegateNode *node in delegateNodes)
		{
			id nodeDelegate = node.delegate;
			#if __has_feature(objc_arc_weak) && !TARGET_OS_IPHONE
			if (nodeDelegate == [NSNull null])
				nodeDelegate = node.unsafeDelegate;
			#endif
			
			//判断监听者是否实现了该方法
			if ([nodeDelegate respondsToSelector:selector])
			{
				// All delegates MUST be invoked ASYNCHRONOUSLY.
				
				NSInvocation *dupInvocation = [self duplicateInvocation:origInvocation];
				
				dispatch_async(node.delegateQueue, ^{ @autoreleasepool {
					
					[dupInvocation invokeWithTarget:nodeDelegate];
					
				}});
			}
			else if (nodeDelegate == nil)
			{
				foundNilDelegate = YES;
			}
		}
		...
	}

将 delegate 从监听者列表中移除

	- (void)removeDelegate:(id)delegate delegateQueue:(dispatch_queue_t)delegateQueue
	{
		if (delegate == nil) return;
		
		NSUInteger i;
		
		//遍历 delegateNodes
		for (i = [delegateNodes count]; i > 0; i--)
		{
			GCDMulticastDelegateNode *node = [delegateNodes objectAtIndex:(i-1)];
			id nodeDelegate = node.delegate;
			#if __has_feature(objc_arc_weak) && !TARGET_OS_IPHONE
			if (nodeDelegate == [NSNull null])
				nodeDelegate = node.unsafeDelegate;
			#endif
			
			if (delegate == nodeDelegate)
			{
				//移除满足条件的监听者(如果delegateQueue == NULL, 则将所有queue中的delegate都删除)
				if ((delegateQueue == NULL) || (delegateQueue == node.delegateQueue))
				{
					node.delegate = nil;
					#if __has_feature(objc_arc_weak) && !TARGET_OS_IPHONE
					node.unsafeDelegate = nil;
					#endif
					
					[delegateNodes removeObjectAtIndex:(i-1)];
				}
			}
		}
	}