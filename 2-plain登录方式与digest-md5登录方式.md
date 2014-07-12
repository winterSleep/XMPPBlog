#  PLAIN登录方式与Digest-MD5登录方式

## 两种登录方式说明

### PLAIN登录验证方式

	PLAIN登录方式, 没有经过任何的加密, 完全是将用户名密码做Base64后, 直接传到服务器
	
### Digest-MD5登录验证方式

	Digest-MD5登录验证方式, 是使用 CC_MD5 做一了一个不可逆加密, 再传输到服务器
	
## 从Objective-c层面来查看两种登录方式的不同

### 简单介绍一下服务器返回的feature信息, 看客户端如何处理

#### 服务器发送feature到客户端的报文如下

	<stream:features>
        <starttls xmlns="urn:ietf:params:xml:ns:xmpp-tls"/>
        <mechanisms xmlns="urn:ietf:params:xml:ns:xmpp-sasl">
            <mechanism>DIGEST-MD5</mechanism>
            <mechanism>PLAIN</mechanism>
            <mechanism>ANONYMOUS</mechanism>
            <mechanism>CRAM-MD5</mechanism>
        </mechanisms>
        <compression xmlns="http://jabber.org/features/compress">
            <method>zlib</method>
        </compression>
        <auth xmlns="http://jabber.org/features/iq-auth"/>
        <register xmlns="http://jabber.org/features/iq-register"/>
    </stream:features>
    
**从上面服务器返回的报文可以看出, 服务器支持4种登录方式 **

#### XMPPStream类如何来处理服务器返回的features报文?

在XMPPStream中, 有如下方法可以监听服务器发送过来的XML报文

	- (void)xmppParser:(XMPPParser *)sender didReadElement:(NSXMLElement *)element{
		...
		//保存features节点数据
		[rootElement setChildren:[NSArray arrayWithObject:element]];
		//处理features
		[self handleStreamFeatures];
		...
	}
	
客户端接收到features后, 会将 XML 报文保存到 rootElement 中.

**XMPPFrameWork根据 <starttls xmlns="urn:ietf:params:xml:ns:xmpp-tls"/> 来判断是否需要开始tls握手连接**

### XMPPStream如何判断Socket连接成功?

#### 服务器返回TLS连接成功报文

	<?xml version='1.0' encoding='UTF-8'?>
	<stream:stream xmlns:stream="http://etherx.jabber.org/streams" xmlns="jabber:client"
    from="192.168.1.115" id="3aeb273e" xml:lang="en" version="1.0">
    <stream:features>
        <mechanisms xmlns="urn:ietf:params:xml:ns:xmpp-sasl">
            <mechanism>DIGEST-MD5</mechanism>
            <mechanism>PLAIN</mechanism>
            <mechanism>ANONYMOUS</mechanism>
            <mechanism>CRAM-MD5</mechanism>
        </mechanisms>
        <compression xmlns="http://jabber.org/features/compress">
            <method>zlib</method>
        </compression>
        <auth xmlns="http://jabber.org/features/iq-auth"/>
        <register xmlns="http://jabber.org/features/iq-register"/>
    </stream:features>	

**请注意, 这里是没有 starttls 节点的**

在下面方法中, XMPP会根据相应上下文, 返回一个 didConnect 广播:

	- (void)handleStreamFeatures{
		...
		[multicastDelegate xmppStreamDidConnect:self]
		...
	}
	
### XMPPStream如何处理不同的验证登录方式?

#### 在xmppStreamDidConnect广播中, 发起一个登录请求

	- (BOOL)authenticateWithPassword:(NSString *)inPassword error:(NSError **)errPtr{
		...
		//判断是否支持DigestMD5
		if ([self supportsDigestMD5Authentication])
		{
			someAuth = [[XMPPDigestMD5Authentication alloc] initWithStream:self password:password];
			result = [self authenticate:someAuth error:&err];
		}
		//是否支持PLAIN
		else if ([self supportsPlainAuthentication])
		{
			someAuth = [[XMPPPlainAuthentication alloc] initWithStream:self password:password];
			result = [self authenticate:someAuth error:&err];
		}	
		...
	}

再看一下 [self authenticate:someAuth error:&err] 方法

	- (BOOL)authenticate:(id <XMPPSASLAuthentication>)inAuth error:(NSError **)errPtr{
		...
		// inAuth 发起一个 start 操作
		if ([inAuth start:&err])
		{
			//保存 inAuth 指针的引用
			auth = inAuth;
			result = YES;
		}
		...
	}

接下来再看一下 [inAuth start:&err] 方法中它做了些什么.

#### 看DigestMd5 的类 XMPPDigestMD5Authentication 如何处理Start

首先, 发出一个auth报文到服务器

	- (BOOL)start:(NSError **)errPtr
	{
		XMPPLogTrace();
		
		// <auth xmlns="urn:ietf:params:xml:ns:xmpp-sasl" mechanism="DIGEST-MD5" />
		NSXMLElement *auth = [NSXMLElement elementWithName:@"auth" xmlns:@"urn:ietf:params:xml:ns:xmpp-sasl"];
		[auth addAttributeWithName:@"mechanism" stringValue:@"DIGEST-MD5"];
		
		//发送一个auth节点, 开始SASL握手操作
		[xmppStream sendAuthElement:auth];
		awaitingChallenge = YES;
		
		return YES;
	}

接下来等待服务器返回一个 challenge 
	
	<challenge xmlns="urn:ietf:params:xml:ns:xmpp-sasl">cmVhbG09IjE5Mi4xNjguMS4xMTUiLG5vbmNlPSJjVmYrclQzbVg4TklqUTVyemU5UUQ4K3BsMVhqUitaUjRRSlBqNkJpIixxb3A9ImF1dGgiLGNoYXJzZXQ9dXRmLTgsYWxnb3JpdGhtPW1kNS1zZXNz</challenge>

接着, 客户端发送一个response报文到服务器

	<response xmlns="urn:ietf:params:xml:ns:xmpp-sasl">Y2hhcnNldD11dGYtOCx1c2VybmFtZT0ibHpwMSIscmVhbG09IjE5Mi4xNjguMS4xMTUiLG5vbmNlPSJjVmYrclQzbVg4TklqUTVyemU5UUQ4K3BsMVhqUitaUjRRSlBqNkJpIixuYz0wMDAwMDAwMSxjbm9uY2U9IitSNzRpUjhzUkdxdE51VWdHY0JYYWZUbzI4N2VxWFVocVpGbno1RTAiLGRpZ2VzdC11cmk9InhtcHAvMTkyLjE2OC4xLjExNSIsbWF4YnVmPTY1NTM2LHJlc3BvbnNlPWYyNWY2NTVlMDQzOTkyMGU0YjJiNjQ3NDQ1Mzk5YzgxLHFvcD1hdXRoLGF1dGh6aWQ9Imx6cDEi</response>
	
服务器通知客户端验证成功

	<success xmlns="urn:ietf:params:xml:ns:xmpp-sasl">cnNwYXV0aD1iMzg0ODI2YTdjY2ZjNmJlNWFjNThiY2RlM2JlNGZlNQ==</success>
	
具体的实现流程可以到 XMPPDigestMD5Authentication.m 中去查看, 代码行数并不是很多

至此, 一个加密的登录就完成了.

#### 看Plain 的类 XMPPPlainAuthentication 如何处理Start

直接发送一个由明文的username 和 password , 然后base64之后的字符串到服务器

	- (BOOL)start:(NSError **)errPtr
	{
		XMPPLogTrace();
		
		// From RFC 4616 - PLAIN SASL Mechanism:
		// [authzid] UTF8NUL authcid UTF8NUL passwd
		// 
		// authzid: authorization identity
		// authcid: authentication identity (username)
		// passwd : password for authcid
		
		NSString *username = [xmppStream.myJID user];
		
		//从这里可以看到, 完全是明文的
		NSString *payload = [NSString stringWithFormat:@"\0%@\0%@", username, password];
		NSString *base64 = [[payload dataUsingEncoding:NSUTF8StringEncoding] xmpp_base64Encoded];
		
		// <auth xmlns="urn:ietf:params:xml:ns:xmpp-sasl" mechanism="PLAIN">Base-64-Info</auth>
		
		NSXMLElement *auth = [NSXMLElement elementWithName:@"auth" xmlns:@"urn:ietf:params:xml:ns:xmpp-sasl"];
		[auth addAttributeWithName:@"mechanism" stringValue:@"PLAIN"];
		[auth setStringValue:base64];
		
		[xmppStream sendAuthElement:auth];
		
		return YES;
	}

然后, 客户端等待服务器返回数据:

	- (XMPPHandleAuthResponse)handleAuth:(NSXMLElement *)authResponse
	{
		XMPPLogTrace();
		
		// We're expecting a success response.
		// If we get anything else we can safely assume it's the equivalent of a failure response.
		
		if ([[authResponse name] isEqualToString:@"success"])
		{
			return XMPP_AUTH_SUCCESS;
		}
		else
		{
			return XMPP_AUTH_FAIL;
		}
	}

从代码可以看到, Plain的登录方式比Digest-MD5的登录方式简单很多, 同时也很不安全.