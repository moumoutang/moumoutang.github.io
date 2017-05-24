---
layout: post
title:  "从wireshark来看SSL"
categories: rourou update
---

从普通的代理或者浏览器的network我们其实很难看到很多网络通信的细节， 
握手的细节等等都看不到，这样我们就要借助于wireshark这样的工具来查看了 （对此表示还是小白） 


如果要看https，我们要在ws里的filter里面输入 ssl  
然后访问：

![Alt text](https://moumoutang.github.io/public/images/wire/clienthello.jpg)

我们可以看到过滤出的protocol中是TLS，TLS其实是SSL协议的升级版本了，可以看到它的版本号已经到1.2    
TLS 1.0通常被标示为SSL 3.1，TLS 1.1为SSL 3.2，TLS 1.2为SSL 3.3。

## SSL的通信过程
- client hello  
从上图中我们可以发现，客户端向服务器端发出了client hello的握手，其中特别重要的是:
	- 客户端协议的版本
	- Random Bytes（一段随机数)
	- 一组signature hash algorithms（客户端可以接受的一系列加密算法）
	支持的压缩方法(compression methods)
- server hello 
	![Alt text](https://moumoutang.github.io/public/images/wire/2017-05-13_15-46-16.jpg)
	![Alt text](https://moumoutang.github.io/public/images/wire/2017-05-17_10-25-53.jpg)

	如上图所示，包含了图中所示的一些信息。但是却没有发现CA证书的传递，却有了change cipher spec(表示接下来的会话要加密进行)    
	还有一句the session reuses previously negotiated keys,说明这个过程是已经复用了之前已经生成的key   
	那么初次建立连接的握手信息应该是什么呢？
   
	![Alt text](https://moumoutang.github.io/public/images/wire/2017-05-17_13-46-12.jpg)
-  client key exchange ,change cipher spec, hello request 
![Alt text](https://moumoutang.github.io/public/images/wire/2017-05-17_14-12-39.jpg)

	  - client key exchange，里面包含了一串由公钥加密的随机数
	  - change cipher spec要求在后续通信中使用加密模式
	  - 会话结束了
     
从这几次握手情况看，总共产生了三串随机数，客户端和服务器端用约定好的加密方法来加密，生成一个会话密钥，那么以后正常的对话跟这个会话密钥有什么关系    
后续正常的application data，内容会用三个随机数生成的绘画密钥来加密进行传递   

### 握手的整个流程
在SSL握手之前，这个过程还是一个正常的tcp/ip通信，所以要经过正常的三次握手之后，才进行SSL握手   
![Alt text](https://moumoutang.github.io/public/images/wire/2017-05-22_16-24-11.jpg)







  

 







