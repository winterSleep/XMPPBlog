## 理解XMPPIDTracker

### XMPPIDTracker的作用

	/**
	 * A common operation in XMPP is to send some kind of request with a unique id,
	 * and wait for the response to come back.
	 * The most common example is sending an IQ of type='get' with a unique id, and then awaiting the response.
	 * 
	 * In order to properly handle the response, the id must be stored.
	 * If there are multiple queries going out and/or different kinds of queries,
	 * then information about the appropriate handling of the response must also be stored.
	 * This may be accomplished by storing the appropriate selector, or perhaps a block handler.
	 * Additionally one may need to setup timeouts and handle those properly as well.
	 * 
	 * This class provides the scaffolding to simplify the tasks associated with this common operation.
	 * Essentially, it provides the following:
	 * - a dictionary where the unique id is the key, and the needed tracking info is the object
	 * - an optional timer to fire upon a timeout
	 * 
	 * The class is designed to be flexible.
	 * You can provide a target/selector or a block handler to be invoked.
	 * Additionally, you can use the basic tracking info, or you can extend it to suit your needs.

以上是英文的原话, 简单来说呢, 就是为了更好的处理服务器返回的数据, 需要保存请求时的报文id, 当服务器返回相应 id 的请求时, 可以通过 XMPPIDTracker 从内存中获取需要处理消息的 target 和 selector, 同时也可以使用 XMPPIDTracker 来处理request请求超时的的操作.

下面一起看一下 XMPPIDTracker 如何处理服务器返回的数据和request超时.

### XMPPIDTracker 的函数

两个初始化方法

	//传入执行 invokeForElement 方法时的 queue 进行初始化
	- (id)initWithDispatchQueue:(dispatch_queue_t)queue;

	//传入一个stream (暂时没什么大用途) 和 queue
	- (id)initWithStream:(XMPPStream *)stream
		   dispatchQueue:(dispatch_queue_t)queue;

四个添加回调和超时的方法:

	//添加一个 element (主要是取它的 elementID属性)、target、selector以及timeout
	- (void)addElement:(XMPPElement *)element target:(id)target selector:(SEL)selector timeout:(NSTimeInterval)timeout;
	
	//添加一个 elementID、block以及timeout
	- (void)addID:(NSString *)elementID
	        block:(void (^)(id obj, id <XMPPTrackingInfo> info))block
	      timeout:(NSTimeInterval)timeout;
	
	- (void)addElement:(XMPPElement *)element
	             block:(void (^)(id obj, id <XMPPTrackingInfo> info))block
	           timeout:(NSTimeInterval)timeout;
	
	//手动初始化一个 满足 XMPPTrackingInfo 协议的对象, 并添加到 XMPPIDTracker中进行保存
	- (void)addID:(NSString *)elementID trackingInfo:(id <XMPPTrackingInfo>)trackingInfo;
	
	- (void)addElement:(XMPPElement *)element trackingInfo:(id <XMPPTrackingInfo>)trackingInfo;
	
手动执行回调的两个方法:

	- (BOOL)invokeForID:(NSString *)elementID withObject:(id)obj;

	- (BOOL)invokeForElement:(XMPPElement *)element withObject:(id)obj;

	
下面以 addElement:target:selector:timeout 为例来看一下它如何处理超时和收到服务器返回数据时的回调

### 如何处理超时

保存 element、target、selector、timeout等信息到内存中

	- (void)addElement:(XMPPElement *)element target:(id)target selector:(SEL)selector timeout:(NSTimeInterval)timeout
	{
		AssertProperQueue();
		
		XMPPBasicTrackingInfo *trackingInfo;
		trackingInfo = [[XMPPBasicTrackingInfo alloc] initWithTarget:target selector:selector timeout:timeout];
		
		[self addElement:element trackingInfo:trackingInfo];
	}

	- (void)addElement:(XMPPElement *)element trackingInfo:(id <XMPPTrackingInfo>)trackingInfo
	{
		AssertProperQueue();
	    
	    if([[element elementID] length] == 0) return;
		
		//将 trackingInfo保存到内存中
		[dict setObject:trackingInfo forKey:[element elementID]];
		
		[trackingInfo setElementID:[element elementID]];
	    [trackingInfo setElement:element];
	    
	    //启动timer
		[trackingInfo createTimerWithDispatchQueue:queue];
	}

trackingInfo 启动timer
	
	- (void)createTimerWithDispatchQueue:(dispatch_queue_t)queue
	{
		NSAssert(queue != NULL, @"Method invoked with NULL queue");
		NSAssert(timer == NULL, @"Method invoked multiple times");
		
		if (timeout > 0.0)
		{
			timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
			
			//设置timer到时间时的操作
			dispatch_source_set_event_handler(timer, ^{ @autoreleasepool {
				
				[self invokeWithObject:nil];
				
			}});
			
			dispatch_time_t tt = dispatch_time(DISPATCH_TIME_NOW, (timeout * NSEC_PER_SEC));
			
			dispatch_source_set_timer(timer, tt, DISPATCH_TIME_FOREVER, 0);
			dispatch_resume(timer);
		}
	}
	
trackingInfo 将消息发送给target的selector方法或者block:

	- (void)invokeWithObject:(id)obj
	{
		if (block)
	    {
			block(obj, self);
		}
	    else if(target && selector)
		{
			#pragma clang diagnostic push
			#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
			[target performSelector:selector withObject:obj withObject:self];
			#pragma clang diagnostic pop
		}
	}
	
### 收到服务器的报文时, 如何回调target和Selector

收到服务器报文时, 可以执行 invokeForElement 方法来执行之前保存的 Selector

	//element(主要是使用 element的elementID来获取内存中的XMPPBasicTrackingInfo对象)
	//object 执行 [target selector] 时, 带过去的附加参数
	[xmppIDTracker invokeForElement:element withObject:object]
	
### XMPPFrameWork中如何使用它

1. 初始化

		xmppIDTracker = [[XMPPIDTracker alloc] initWithStream:xmppStream dispatchQueue:moduleQueue];
	
2. 添加target、selector、timeout到内存中

        [xmppIDTracker addElement:iq
                           target:self
                         selector:@selector(handleFetchRosterQueryIQ:withInfo:)
                          timeout:60];
3. 收到服务器报文时的回调

		- (BOOL)xmppStream:(XMPPStream *)sender didReceiveIQ:(XMPPIQ *)element
		{	
			...	
			//第一个element (主要是使用它的elementID)
			//第二个element (回调的时候, 会将它传给回调方法)
            [xmppIDTracker invokeForElement:element withObject:element];
			...
		}
		
4. handleFetchRosterQueryIQ:withInfo 方法(超时时和主动调用invokeForElement都会执行它, 不过超时时, iq参数为nil)

	有两个参数:
	* iq (服务器返回的报文, 即invokeForElement时的withObject这个参数)
	* basicTrackingInfo
		
			- (void)handleFetchRosterQueryIQ:(XMPPIQ *)iq withInfo:(XMPPBasicTrackingInfo *)basicTrackingInfo{
	    
			}