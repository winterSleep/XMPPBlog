# 理解XMPPRoster的subscription

要理解XMPPRoster, 首先, 必须要理解 subscription 这个概念, 先看一下 [RFC3921](http://wiki.jabbercn.org/RFC3921)中关于 subscription 的定义:

一个名册条目相关的出席信息订阅的状态从<item/>元素的'subscription'属性可以得到.这个属性允许的值包括:
		
	"none" -- 这个用户没有对这个联系人出席信息的订阅, 这个联系人也没有订阅用户的出席信息
	"to" -- 这个用户订阅了这个联系人的出席信息, 但是这个联系人没有订阅用户的出席信息
	"from" -- 这个联系人订阅了用户的出席信息, 但是这个用户没有订阅这个联系人的出席信息
	"both" -- 用户和联系人互相订阅了对方的出席信息

四个状态可以互相转换, 下面看一下它们之间是如何转换的.

1. 假设登录用户(zhangsan)已经有一个好友(lisi), 获取好友列表时, 返回的好友列表的报文如下, subscription 的状态为 both

		<iq type="result" id="5nKV7-6" to="lzp1@192.168.1.115/mobile">
	        <query xmlns="jabber:iq:roster">
	            <item jid="lisi@192.168.1.115" name="李四" subscription="both">
	                <group>Friends</group>
	            </item>
	        </query>
	    </iq>
	    
2. zhangsan 申请添加用户 wangwu 为好友, 发送好友请求后（**添加好友**操作将会在下一节进行介绍），登录用户（zhangsan）的好友列表报文如下:

	其中：<br/>
	name 为昵称，在添加好友时可以设定<br/>
	group为好友所在的分组<br/>
	ask 节点为 subscribe，表示正在等待对方接受好友请求<br/>
	subscription 为 none，表示暂时双方都还不是好友

		<iq type="result" id="5nKV7-6" to="lzp1@192.168.1.115/mobile">
	        <query xmlns="jabber:iq:roster">
	            <item jid="lisi@192.168.1.115" name="李四" subscription="both">
	                <group>Friends</group>
	            </item>
	            <item jid="wangwu@192.168.1.115" name="王五" ask="subscribe" subscription="none">
	                <group>Friends</group>
	            </item>
	        </query>
	    </iq>
	    
3. wangwu 收到 zhangsan 发过来的好友请求，同意好友申请（发送一个同意的报文到服务器），同时，wangwu会再发送一个**申请添加 zhangsan为好友的请求到服务器**

	这个时候，wangwu 的好友列表报文如下，其中 subscription 为 from，表示 wangwu 是 zhangsan 的好友，但 zhangsan 并不是 wangwu 的好友（用微博的表达来说就是：zhangsan 关注了 wangwu，但是 wangwu 并不一定需要关注 zhangsan ）
	
		<iq type="result" id="5nKV7-6" to="lzp1@192.168.1.115/mobile">
	        <query xmlns="jabber:iq:roster">
	            <item jid="zhangsan@192.168.1.115" name="张三" ask="subscribe" subscription="from">
	                <group>Friends</group>
	            </item>
	        </query>
	    </iq>
	    
4. zhangsan 收到 wangwu “同意zhangsan的好友请求” 后，roster列表中，wangwu的subscription变为 to

		<iq type="result" id="5nKV7-6" to="lzp1@192.168.1.115/mobile">
	        <query xmlns="jabber:iq:roster">
	            <item jid="lisi@192.168.1.115" name="李四" subscription="both">
	                <group>Friends</group>
	            </item>
	            <item jid="wangwu@192.168.1.115" name="王五" subscription="to">
	                <group>Friends</group>
	            </item>
	        </query>
	    </iq>
	    
5. 同时，zhangsan收到了 wangwu 也想添加 zhangsan为好友的请求，zhangsan发送了一个 “同意好友申请” 到服务器，此时，wangwu 的 subscription 变为了 both，好友列表如下：
	
		<iq type="result" id="5nKV7-6" to="lzp1@192.168.1.115/mobile">
	        <query xmlns="jabber:iq:roster">
	            <item jid="lisi@192.168.1.115" name="李四" subscription="both">
	                <group>Friends</group>
	            </item>
	            <item jid="wangwu@192.168.1.115" name="王五" subscription="to">
	                <group>Friends</group>
	            </item>
	        </query>
	    </iq>
	    

因此，整个流程可以理解为：

	1. A 申请 添加 B 为好友
	2. B 同意 A 的好友申请，A-->B (A to B, B from A)
	3. B 申请添加A为好友
	4. A 同意 B的好友申请   A <--> B (A both B，B both A)
	
四个步骤都完成了，表示A 与 B互为好友了。

这里理解起来有点困难，咱们再用 twitter的 “关注” 这个概念来理解一下（假设A关注B，是需要B同意了才能关注）

	1. A 想关注B，然后发送申请给B
	2. B 收到了申请，然后同意 A关注自己， 状态就变成了：（A --> B）
	3. 同时，B 也想关注A，然后发送申请给A （当然B也可以选择不关注A）
	4. A 收到B的申请， 同意B关注自己，状态就变成了 （A <--> B），A与B互相关注了。
