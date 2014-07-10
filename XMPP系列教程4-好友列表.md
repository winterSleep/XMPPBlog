# XMPP获取好友列表

本篇文章主要是介绍如何获取好友列表, 包括客户端的请求和服务器返回的数据.
关于XMPP的好友协议, 可以参考[RFC3921](http://wiki.jabbercn.org/RFC3921)

## 获取好友列表报文格式

### 发送的报文

	<iq id="5nKV7-6" type="get"><query xmlns="jabber:iq:roster"></query></iq>

### 收到的报文
	
	<iq type="result" id="5nKV7-6" to="lzp1@192.168.1.115/mobile">
        <query xmlns="jabber:iq:roster">
            <item jid="lzp2@192.168.1.115" name="qqq" ask="subscribe" subscription="none">
                <group>Friends</group>
            </item>
        </query>
    </iq>
    
## 理解 XMPPModule

1. XMPPModule 是大部分XMPP模块的基类, 包括 XMPPRoster、XMPPAutoPing、XMPPMUC等模块， 都是继承自XMPPModule。

2. XMPPModule的主要功能是：监听XMPPStream类的广播，并转换成相应的广播转发出去。

3. XMPPMudle 的方法主要有以下几个：

	
		//设置module的stream, 并将自己添加到xmppStream的广播中
		- (BOOL)activate:(XMPPStream *)aXmppStream; 
		
		//将自己从XMPPStream的广播中移除, 同时将 module 的stream置为nil 
		- (void)deactivate;
		
		//将 delegate 对象添加到 module 的广播队列中(delegate对象为弱引用)
		- (void)addDelegate:(id)delegate delegateQueue:(dispatch_queue_t)delegateQueue;
		
		//将 delegate 对象从 module 的广播队列中移除
		- (void)removeDelegate:(id)delegate delegateQueue:(dispatch_queue_t)delegateQueue;
		
		//将 delegate 对象从 module 的广播队列中移除
		- (void)removeDelegate:(id)delegate;
		
## XMPPRoster类

XMPPRoster类继承自XMPPModule，主要用于处理Roster相关的网络请求(比如：获取好友列表、添加好友、接受好友请求操作等)，并将相应的网络请求广播出去

### XMPPRoster类的一些方法

	//初始化方法, XMPPRosterStorage 有两种模式, 一种是 Memory方式, 一种是Core Data方式
	//XMPPRosterMemoryStorage 类的实例对象为 Memory方式
	//XMPPRosterCoreDataStorage 类的实例对象为使用 Core Data 对roster进行存储
	- (id)initWithRosterStorage:(id <XMPPRosterStorage>)storage;

	//登录成功后, 是否自动去获取好友列表
	@property (assign) BOOL autoFetchRoster;
	
	//是否自动接受已知用户的好友请求 (这个比较复杂, 后面会有 subscription 相应的章节来讲这个开关的用途)
	@property (assign) BOOL autoAcceptKnownPresenceSubscriptionRequests;
	
	//手动获取roster列表(当 autoFetchRoster 为NO的时候, 需要手动调用才能获取到好友列表)
	- (void)fetchRoster;
	
### 初始化XMPPRsoter

	//初始化一个memory storage对象
	_xmppRosterStorage = [[XMPPRosterMemoryStorage alloc] init];
	
	//初始化一个xmppRoster对象, 
	_xmppRoster = [[XMPPRoster alloc] initWithRosterStorage:_xmppRosterStorage dispatchQueue:_xmppQueue];
	
	//设置自动获取roster
	_xmppRoster.autoFetchRoster = YES;
	
	//自动同意已知好友的"好友请求"
	_xmppRoster.autoAcceptKnownPresenceSubscriptionRequests = YES;
	
	//设置xmppStream, 并将 _xmppRoster添加到 _xmppStream 的广播中
    [_xmppRoster activate:_xmppStream];
    
    //将 self 添加到广播队列中
    [_xmppRoster addDelegate:self delegateQueue:_xmppQueue]
    
至此, XMPPRoster 对象初始化完成

### XMPPRoster如何自动获取好友列表?

XMPPStream 登录成功后, 会发送 xmppStreamDidAuthenticate 广播, XMPPRoster接收到这个广播后, 会自动去获取roster列表 (xmppStreamDidAuthenticate 函数何时调用, 可参考第二节)

	- (void)xmppStreamDidAuthenticate:(XMPPStream *)sender
	{
		// This method is invoked on the moduleQueue.
		
		XMPPLogTrace();
		
		if ([self autoFetchRoster])
		{
			//获取 roster 列表
			[self fetchRoster];
		}
	}
	
接下来看下 fetchRoster 它做了些什么呢?

	- (void)fetchRoster
	{
		...	
		
		NSXMLElement *query = [NSXMLElement elementWithName:@"query" xmlns:@"jabber:iq:roster"];
		XMPPIQ *iq = [XMPPIQ iqWithType:@"get" elementID:[xmppStream generateUUID]];
		[iq addChild:query];
	    [xmppIDTracker addElement:iq target:self selector:@selector(handleFetchRosterQueryIQ:withInfo:) timeout:60];
		[xmppStream sendElement:iq];
		[self _setRequestedRoster:YES];

		...
	}

上面的代码, 添加了一个element到 xmppIDTracker对象中, 用于处理请求超时以及服务器正常返回数据时的操作, 如下代码:

	[xmppIDTracker addElement:iq
					   target:self
                    selector:@selector(handleFetchRosterQueryIQ:withInfo:)
                     timeout:60];
                          
同时构造了如下的一个XML报文发送出去到服务器(注意看这里的xmlns是 jabber:iq:roster):

	<iq id="5nKV7-6" type="get"><query xmlns="jabber:iq:roster"></query></iq>
	
服务器返回报文:

	<iq type="result" id="5nKV7-6" to="lzp1@192.168.1.115/mobile">
        <query xmlns="jabber:iq:roster">
            <item jid="lzp2@192.168.1.115" name="qqq" ask="subscribe" subscription="none">
                <group>Friends</group>
            </item>
        </query>
    </iq>
	
XMPPRoster解析报文(从服务器返回的报文能看出来，服务器返回的是一个 iq 节点的报文，所以XMPPRoster在 didReceiveIQ 这个广播中解析 roster报文)
		
	- (BOOL)xmppStream:(XMPPStream *)sender didReceiveIQ:(XMPPIQ *)iq
	{
		NSXMLElement *query = [iq elementForName:@"query" xmlns:@"jabber:iq:roster"];
		//判断 服务器返回的是否是roster报文	
	    if (query)
		{
	        if([iq isSetIQ])
	        {
	            [multicastDelegate xmppRoster:self didReceiveRosterPush:iq];
	            
	            NSArray *items = [query elementsForName:@"item"];
	            [self _addRosterItems:items];
	        }
	        else if([iq isResultIQ])
	        {
	            [xmppIDTracker invokeForElement:iq withObject:iq];
	        }
			
			return YES;
		}
		
		return NO;
	}
	
服务器返回roster列表时, 执行了 [xmppIDTracker invokeForElement:iq withObject:iq]; 方法, 根据上一节XMPPIDTracker的介绍, 该方法会调用 handleFetchRosterQueryIQ:withInfo: 方法, 如下:

	- (void)handleFetchRosterQueryIQ:(XMPPIQ *)iq withInfo:(XMPPBasicTrackingInfo *)basicTrackingInfo{
    
	    dispatch_block_t block = ^{ @autoreleasepool {
	        
	        NSXMLElement *query = [iq elementForName:@"query" xmlns:@"jabber:iq:roster"];
	        
			BOOL hasRoster = [self hasRoster];
			
			//如果之前未获取过好友列表, 则发送的广播
			if (!hasRoster)
			{
	            [xmppRosterStorage clearAllUsersAndResourcesForXMPPStream:xmppStream];
	            [self _setPopulatingRoster:YES];
	            
	            //发送一个 xmppRosterDidBeginPopulating 广播
	            [multicastDelegate xmppRosterDidBeginPopulating:self];
				[xmppRosterStorage beginRosterPopulationForXMPPStream:xmppStream];
			}
			
			NSArray *items = [query elementsForName:@"item"];
	        [self _addRosterItems:items];
			
			if (!hasRoster)
			{
				// We should have our roster now
				
				[self _setHasRoster:YES];
	            [self _setPopulatingRoster:NO];
	            [multicastDelegate xmppRosterDidEndPopulating:self];
				[xmppRosterStorage endRosterPopulationForXMPPStream:xmppStream];
				
				// Process any premature presence elements we received.
				
				for (XMPPPresence *presence in earlyPresenceElements)
				{
					[self xmppStream:xmppStream didReceivePresence:presence];
				}
	            
				[earlyPresenceElements removeAllObjects];
			}
	        
	    }};
		
		if (dispatch_get_specific(moduleQueueTag))
			block();
		else
			dispatch_async(moduleQueue, block);
	    
	}

### 监听Roster获取完成时的回调

/**
 * Sent when the initial roster is received.<br/>
 * 当roster开始往 storage(coreData 或者 memory) 添加数据时的回调
**/

	- (void)xmppRosterDidBeginPopulating:(XMPPRoster *)sender;

/**
 * Sent when the initial roster has been populated into storage.<br/>
 * 当roster往 storage(coreData 或者 memory) 添加数据完成后的回调
**/

	- (void)xmppRosterDidEndPopulating:(XMPPRoster *)sender;

/**
 * Sent when the roster receives a roster item.<br/>
 * 在 发送 xmppRosterDidBeginPopulating 广播后, 会陆续收到didReceiveRosterItem 的回调<br/>
 * 在 发送 xmppRosterDidEndPopulating 广播后, 表示roster列表遍历完成
**/

	- (void)xmppRoster:(XMPPRoster *)sender didReceiveRosterItem:(NSXMLElement *)item;
	
监听以上三个方法, 可以完成基本的好友获取操作, 使用 XMPPRosterMemoryStorage 类的回调则可以监听到更多的回调, 可以自己参照代码进行查看.
好友列表完成时的回调方法如下:

	- (void)xmppRosterDidPopulate:(XMPPRosterMemoryStorage *)sender{
		//通过 sortedUsersByName 方法可以从 memory对象中获取roster列表
		NSArray *roster = [sender sortedUsersByName];
	}
	
**其它回调方法, 比如: 删除好友、添加好友、接收到好友请求等操作，将会慢慢道来**