# Openfire连接管理器(Connection Manager)

by Rusher @2013-05-30

## ConncetionManager介绍

![connection-managers](http://www.igniterealtime.org/images/connection-managers.gif)

连接管理器放在Openfire的前端，用于保持客户端的连接，连接管理器与Openfire之间保持少量的连接，这样，便能减轻Openfire的连接压力。CM与OF之间通过扩展协议传输客户端与服务端的包。


## 协议分析

在XEP文档中没有找到CM和OF之间的扩展协议，所以，下面简单的分析一下协议内容。

_[SEND]表示CM发送给OF，[RECV]表示OF发给CM_

**第一步**：ConnectionWorkerThread创建与OF的TCP连接和XMPP连接。

	<stream:stream 	xmlns:stream="http://etherx.jabber.org/streams" 
					xmlns="jabber:connectionmanager" 
					to="q6334/Connection Worker - 1" 
					version="1.0">

CM收到服务端的回应后，将服务端配置的密码和streamID算出SHA-1值，发送`<handshake/>`进行验证，

	<handshake>5f3129447e25856fa19d83a472237d720492968a</handshake>

服务端验证成功后返回一个空的`<handshake/>`。

服务端发送设置项:

	[RECV]
	<iq type="set" id="868-16">
	  <configuration xmlns="http://jabber.org/protocol/connectionmanager">
	    <starttls xmlns="urn:ietf:params:xml:ns:xmpp-tls"></starttls>
	    <mechanisms xmlns="urn:ietf:params:xml:ns:xmpp-sasl">
	      <mechanism>DIGEST-MD5</mechanism>
	      <mechanism>PLAIN</mechanism>
	      <mechanism>ANONYMOUS</mechanism>
	      <mechanism>CRAM-MD5</mechanism>
	    </mechanisms>
	    <compression xmlns="http://jabber.org/features/compress">
	      <method>zlib</method>
	    </compression>
	    <auth xmlns="http://jabber.org/features/iq-auth"></auth>
	    <register xmlns="http://jabber.org/features/iq-register"></register>
	  </configuration>
	</iq>

	[SEND]
	<iq type="result" id="868-16" to="openfire.irusher.com" from="q6334/Connection Worker - 1">
	  <configuration xmlns="http://jabber.org/protocol/connectionmanager">
	    <starttls xmlns="urn:ietf:params:xml:ns:xmpp-tls"></starttls>
	    <mechanisms xmlns="urn:ietf:params:xml:ns:xmpp-sasl">
	      <mechanism>DIGEST-MD5</mechanism>
	      <mechanism>PLAIN</mechanism>
	      <mechanism>ANONYMOUS</mechanism>
	      <mechanism>CRAM-MD5</mechanism>
	    </mechanisms>
	    <compression xmlns="http://jabber.org/features/compress">
	      <method>zlib</method>
	    </compression>
	    <auth xmlns="http://jabber.org/features/iq-auth"></auth>
	    <register xmlns="http://jabber.org/features/iq-register"></register>
	  </configuration>
	</iq>

**第二步**：客户端连接CM后，建立CM与OF之间的会话

	[SEND]
	<iq type="set" to="openfire.irusher.com" from="q6334/Connection Worker - 1" id="234-0">
	  <session xmlns="http://jabber.org/protocol/connectionmanager" id="q63347a14e957">
	    <create>
	      <host name="localhost" address="127.0.0.1"/>
	    </create>
	  </session>
	</iq>


	[RECV]	
	<iq type="result" id="234-0" from="openfire.irusher.com" to="q6334/Connection Worker - 1">
	  <session xmlns="http://jabber.org/protocol/connectionmanager" id="q63347a14e957">
	    <create>
	      <host name="localhost" address="127.0.0.1"/>
	    </create>
	  </session>
	</iq>

CM与OF的会话建立之后，客户端与OF之间的XMPP包内容，都被包含在`<route/>`标签内，同时，标签还带有一个'streamid'属性，用于标识客户端的会话。

	[SEND]
	<route to="openfire.irusher.com" from="q6334/Connection Worker - 1" streamid="q63347a14e957">
	  <auth xmlns="urn:ietf:params:xml:ns:xmpp-sasl" mechanism="DIGEST-MD5"></auth>
	</route>

	[RECV]
	<route from="openfire.irusher.com" to="q6334" streamid="q63347a14e957">
	  <challenge xmlns="urn:ietf:params:xml:ns:xmpp-sasl">cmVhbG09Im9wZW5maXJlLmlydXNoZXIuY29tIixub25jZT0iLytPaWQ1QWNLdXduTFJqbGZZNmZNU1pzeWJuSlVUdVlUMUFqRExWayIscW9wPSJhdXRoIixjaGFyc2V0PXV0Zi04LGFsZ29yaXRobT1tZDUtc2Vzcw==</challenge>
	</route>
	
	[SEND]	
	 <route to="openfire.irusher.com" from="q6334/Connection Worker - 1" streamid="q63347a14e957">
	   <response xmlns="urn:ietf:params:xml:ns:xmpp-sasl">dXNlcm5hbWU9InJvYm90YSIscmVhbG09Im9wZW5maXJlLmlydXNoZXIuY29tIixub25jZT0iLytPaWQ1QWNLdXduTFJqbGZZNmZNU1pzeWJuSlVUdVlUMUFqRExWayIsY25vbmNlPSI5NjA2Q0I1NS04NDQ1LTQ4MzAtOTE1My0wMkQxMTQ5RTM2RkIiLG5jPTAwMDAwMDAxLHFvcD1hdXRoLGRpZ2VzdC11cmk9InhtcHAvb3BlbmZpcmUuaXJ1c2hlci5jb20iLHJlc3BvbnNlPWNiZWRmZmUyMDg1MmQ3NmEwYzkzZTJjYTk5OGE4ZWUwLGNoYXJzZXQ9dXRmLTg=</response>
	 </route>
	   
	 [RECV]  
	 <route from="openfire.irusher.com" to="q6334" streamid="q63347a14e957">
	   <success xmlns="urn:ietf:params:xml:ns:xmpp-sasl">cnNwYXV0aD0yYWIzOTkyYTQ2YjYzZWJhZWUwZDNjYTc1MjAzZThiZg==</success>
	 </route>  
	 
	 [SEND]
	 <route to="openfire.irusher.com" from="q6334/Connection Worker - 1" streamid="q63347a14e957">
	   <iq type="set">
	     <bind xmlns="urn:ietf:params:xml:ns:xmpp-bind"></bind>
	   </iq>
	 </route>  
	 
	 [RECV]
	 <route streamid="q63347a14e957" from="openfire.irusher.com" to="q6334">
	   <iq type="result" to="openfire.irusher.com/q63347a14e957">
	     <bind xmlns="urn:ietf:params:xml:ns:xmpp-bind">
	       <jid>robota@openfire.irusher.com/q63347a14e957</jid>
	     </bind>
	   </iq>
	 </route>  
	 
	 [SEND]
	 <route to="openfire.irusher.com" from="q6334/Connection Worker - 1" streamid="q63347a14e957">
	   <iq type="set">
	     <session xmlns="urn:ietf:params:xml:ns:xmpp-session"></session>
	   </iq>
	 </route>  
	 
	 [RECV]
	 <route streamid="q63347a14e957" from="openfire.irusher.com" to="q6334">
	   <iq type="result" to="robota@openfire.irusher.com/q63347a14e957"/>
	 </route>  
	 
	 [SEND]
	 <route to="openfire.irusher.com" from="q6334/Connection Worker - 1" streamid="q63347a14e957">
	   <iq type="get">
	     <query xmlns="jabber:iq:roster"></query>
	   </iq>
	 </route>  
	 
	 [RECV]
	 <route streamid="q63347a14e957" from="openfire.irusher.com" to="q6334">
	   <iq type="result" to="robota@openfire.irusher.com/q63347a14e957">
	     <query xmlns="jabber:iq:roster">
	       <item jid="robotx@openfire.irusher.com" ask="subscribe" subscription="from"/>
	       <item jid="robotb@openfire.irusher.com" name="robotb" subscription="both">
	         <group>Friends</group>
	       </item>
	     </query>
	   </iq>
	 </route>  
	 
	 [SEND]
	 <route to="openfire.irusher.com" from="q6334/Connection Worker - 1" streamid="q63347a14e957">
	   <iq type="get" to="robota@openfire.irusher.com">
	     <vCard xmlns="vcard-temp"></vCard>
	   </iq>
	 </route>  
	 
	 [SEND]
	 <route to="openfire.irusher.com" from="q6334/Connection Worker - 1" streamid="q63347a14e957">
	   <presence>
	     <x xmlns="vcard-temp:x:update">
	       <photo/>
	     </x>
	     <c xmlns="http://jabber.org/protocol/caps" hash="sha-1" node="http://code.google.com/p/xmppframework" ver="k6gP4Ua5m4uu9YorAG0LRXM+kZY="></c>
	   </presence>
	 </route>  
	 
	 [RECV]
	 <route streamid="q63347a14e957" from="openfire.irusher.com" to="q6334">
	   <iq type="result" from="robota@openfire.irusher.com" to="robota@openfire.irusher.com/q63347a14e957">
	     <vCard xmlns="vcard-temp"></vCard>
	   </iq>
	 </route>  
	 
	 [SEND]
	 <route to="openfire.irusher.com" from="q6334/Connection Worker - 1" streamid="q63347a14e957">
	   <presence>
	     <x xmlns="vcard-temp:x:update">
	       <photo/>
	     </x>
	     <c xmlns="http://jabber.org/protocol/caps" hash="sha-1" node="http://code.google.com/p/xmppframework" ver="k6gP4Ua5m4uu9YorAG0LRXM+kZY="></c>
	     <x xmlns="vcard-temp:x:update">
	       <photo/>
	     </x>
	     <c xmlns="http://jabber.org/protocol/caps" hash="sha-1" node="http://code.google.com/p/xmppframework" ver="k6gP4Ua5m4uu9YorAG0LRXM+kZY="></c>
	   </presence>
	 </route>  
	 
	 [SEND]
	 <route to="openfire.irusher.com" from="q6334/Connection Worker - 1" streamid="q63347a14e957">
	   <message type="chat" to="robotb@openfire.irusher.com" from="robota@openfire.irusher.com/q63347a14e957" id="13053016172881">
	     <body>ABC</body>
	   </message>
	 </route>  
	 
	 [RECV]
	 <route streamid="q63347a14e957" from="openfire.irusher.com" to="q6334">
	   <message to="robota@openfire.irusher.com/q63347a14e957" from="admin@openfire.irusher.com" type="chat">
	     <received xmlns="http://yonglang.co/xmpp/server/reveived" id="13053016172881" timestamp="2013-05-30T16:17:28Z"></received>
	   </message>
	 </route>  
	 
	 [SEND]
	 <route to="openfire.irusher.com" from="q6334/Connection Worker - 1" streamid="q63347a14e957">
	   <presence type="unavailable">
	     <x xmlns="vcard-temp:x:update">
	       <photo/>
	     </x>
	   </presence>
	 </route>  
	 
**结束**：客户端断开连接，CM和OF关闭会话。
	 
	 [SEND]
	 <iq type="set" to="openfire.irusher.com" from="q6334/Connection Worker - 1" id="784-1">
	   <session xmlns="http://jabber.org/protocol/connectionmanager" id="q63347a14e957">
	     <close/>
	   </session>
	 </iq>  
	 
	 [RECV]
	 <iq type="result" id="784-1" from="openfire.irusher.com" to="q6334/Connection Worker - 1">
	   <session xmlns="http://jabber.org/protocol/connectionmanager" id="q63347a14e957">
	     <close/>
	   </session>
	 </iq>


## 代码结构

CM和Openfire都是基于Mina框架。

代码示意图：

![cm_architecture.png](./cm_architecture.png)

### 连接与会话

####1.客户端与CM的TCP连接

CM建立一个`SocketAcceptor`，用于接受客户端的Socket,设置多线程模式,使用线程池中的线程接受客户端发来的XML内容。为`SocketAcceptor`绑定一个`ClientConnectionHandler`。`ClientConnectionHandler`是`IoHandlerAdapter`的一个实现类，可以对客户端的会话事件进行处理:

	void exceptionCaught(IoSession session, Throwable cause)
	void messageReceived(IoSession session, Object message)
	void messageSent(IoSession session, Object message)
	void sessionClosed(IoSession session)
	void sessionCreated(IoSession session)
	void sessionIdle(IoSession session, IdleStatus status)
	void sessionOpened(IoSession session)
	
`ClientConnectionHandler`主要完成连接的建立(`NIOConnection`)，XML解析器(`XmlPullParser`)，XMPP协议操作类(`ClientStanzaHandler`)的建立，以及连接关闭时资源的释放。当CM收到客户端的消息后，具体的XMPP协议解析交给`ClientStanzaHandler`完成。`ClientStanzaHandler`通过解析XML的内容完成以下工作：

a. 建立CM与OF的会话(`ClientSession`)
b. 建立客户端与CM之间的XMPP连接
c. 判断是否使用TLS，流压缩
d. 将消息交与`ServerRouter`处理。

####2.CM与OF连接

CM与OF之间保持少量的TCP连接，默认是5个，连接所在的线程由线程池维护，连接池由类`ServerSurrogate`保持，在`ConnectionManager`启动的时候启动连接池。每个`ConnectionWorkerThread`线程建立一个TCP连接，在这之上建立XMPP协议层的连接([XEP-0114: Jabber Component Protocol](http://xmpp.org/extensions/xep-0114.html)),协议内容已经在**协议分析**中提到。

CM与OF连接的Socket由`SocketConnection`持有，此类负责向Socket写或者从Socket读取数据。

Socket的字节输入流经过字节流字符流转换类流入`XMPPPacketReader`类，再由`ServerPacketReader`的后台线程读取，建立线程池，在多个线程中用`ServerPacketHandler`处理XMPP协议。协议内容已经在**协议分析**中提到，在XMPP连接初始建立的时候，`ServerPacketHandler`通过解析XML内容来决定`SocketConnection`是否使用安全连接，是否压缩等特性。

####3.CM与OF建立新的会话

CM与OF之间只是传输的作用，并没有其他的业务逻辑，所有的客户端发向OF的XMPP协议内容，都被包裹在'<route/>'元素中。但是由于是多个客户端的内容通过少量的连接发向OF，所以对内容进行标识，以保证服务端的回复内容返回到正确的客户端。在**1.a**中提到`ClientStanzaHandler`通过解析XML的内容建立CM与OF的会话(`ClientSession`)，每个会话内使用相同的`streamdid`,作为'<route/>'元素的属性，来标识不同的客户端。`ClientSession`的超类`Session`持有一个静态的HashMap，以`streamid`为键，`ClientSession`(或者`HttpSession`，在**BOSH**中涉及)为值，这样就可以方便的获取到`streamid`对应的会话了。


### 消息流转

`ServerSurrogate`代理转发所有流向服务器的数据。

`SocketSenderTracker`定时扫描CM与OF之间的连接，检查连接是否可用。

### BOSH

TODO

### 安全连接

TODO

## 配置与部署

TODO

