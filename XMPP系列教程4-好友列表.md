# XMPP获取好友列表

## 报文格式

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

上面的代码, 其实只是构造了如下的一个XML报文发送出去到服务器(注意看这里的xmlns是 jabber:iq:roster):

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
	
从服务器返回的报文来看, XMPPRoster应该会走如下方法:

	[xmppIDTracker invokeForElement:iq withObject:iq];
	
### 