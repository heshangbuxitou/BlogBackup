---
title: Java NIO最终实践之Web服务器
date: 2018-01-19 16:11:31
tags:
- NIO
- select
- Servlet
- Tomcat 5.0
categori0es: 
- Java
---

# 前言

## C10K问题

在互联网开始的早期，使用互联网的人较少，一台服务器同时在线的连接也不是很多，所以最初的服务器设计的时候使用进程或者是线程的方式分配一个TCP连接，这个时候不存在C10K的难题。

当到了Web2.0的时代，互联网不再是单纯的浏览网页了，它开始需要进行交互，随着互联网的进一步发展，用户界面和界面交互都变得非常复杂起来，应用程序的逻辑也随之变的更加复杂，即时通信和在线的实时互动已经变的非常普遍了，假设每个用户都必须要与服务器保持一个或者多个TCP连接，而且每一个TCP连接需要占用一个进程(线程)的资源，这样的话，一个服务器的并发连接数是非常高的，一个普通的大一点网页服务的连接可能就过亿了。进程是操作系统最宝贵的资源，一台机器创建不了这么多进程，如果是C10k就要创建1万个进程，这个是操作系统无法承受的。就算是分布式系统，维持1亿用户在线也需要10万台服务器，成本是巨大的，只有FLAG、BAT这样的公司才有财力购买如此多的服务器。

## 怎样解决

解决这个问题的方法其实就在我们身边，要知道在我们打电话的时候，其中无线频率是稀有的资源，不能每个人都有各自的频率，因此无线频率提供商使用多路复用技术通过一个频率发送多个呼叫。在一个例子中，把呼叫分成一些段，然后给这些段很短的持续时间，并在接收端重新装配。这就叫做*时分多路复用*（time-division multiplexing），即 TDM。而我们解决这个难题的方式也是一样，只不过名字有点不同，它就是*IO多路复用*。

关于*IO多路复用*的知识你需要到网上寻找更加详细的知识，或者我以前的文章，这里就不细讲了，我们这篇主要讲的是Java，Java在JDK1.4版本的时候发布了NIO，因此我们在Java中也可以使用*IO多路复用*来处理连接的IO了。

# 使用NIO的Web服务器

## 不阻塞到阻塞

对于使用NIO的服务器，我们是必须要是非阻塞的读写，否则NIO将不能正常工作，但是在我们的Web服务器中会引发许多问题，因为一个请求过来，我们的客户端或服务器应用程序可能读取完整信息、部分消息或者根本读取不到消息。另外，非阻塞读可能读取到太多的消息，从而强制为下一个呼叫准备一个额外的缓冲区。最后，不像流那样，读取了零字节并不表明已经完全接收了消息。

因此这就表示使用`readline`等方法都有一些困难，因此我们需要使用`java.io.PipedInput`和`PipedOutputStream`类来把生产者/消费者模型应用到消费者非阻塞`IO`。当读取非阻塞通道时，把它写到正由第二个线程消费的管道。读写数据被我们用一个管道分开了，这样让一个线程单独负责处理非阻塞通道（生产者），让另一个线程单独负责把数据作为流消费（消费者）。管道也为应用程序服务器解决了非阻塞 `IO` 问题，因为`servlet`在消费`IO`时将采用阻塞语义。

## Server类

在这里Server类主要是对多路复用循环的处理，它监听我们感兴趣的IO事件，并且通过`selector.select();`返回发生的事件数，我们通过对事件返回的`Iterator`进行遍历处理，注意，所有的数据都是在这个循环中读取的。通常会把从特定`socket`中读取字节的任务分配给一个新线程。使用的是`NIO`选择器事件驱动方法，实际上可以用单个线程处理成千上万的客户机，不过，我们还会在后面看到线程仍有一个角色要扮演。这里就是监控发生的事件，然后把它发给`ServerEventHandler`类进行处理。

``` java
	public void listen() {
		SelectionKey key = null;
		try {
			for (;;) {
				selector.select();
				Iterator it = selector.selectedKeys().iterator();
				while (it.hasNext()) {
					key = (SelectionKey) it.next();
					handleKey(key);
					it.remove();
				}
			}
		} catch (IOException e) {
			key.cancel();
		} catch (NullPointerException e) {
			// NullPointer at sun.nio.ch.WindowsSelectorImpl, Bug: 4729342
			e.printStackTrace();			
		}
    }
    
    private void handleKey(SelectionKey key)
    throws IOException {
    if (key.isAcceptable())
        myHandler.acceptNewClient(selector, key);
    else if (key.isReadable())
        myHandler.readDataFromSocket(key);
	}
```

## ServerEventHandle类

ServerEventHandler类是处理IO事件的类。当新的连接到来时，我们就实例化一个新的`Client`对象，该对象代表了那个客户端的状态。数据是以非阻塞方式从通道中读取的，并被写到`Client`对象中。 

我们使用生产者/消费者模型来处理`Client`，当有新的`Client`对象被处理时，把它放到队列中，消费者可以在队列中获取到`Client`并进行处理，当然，在队列为空的时候消费者线程是阻塞的。

``` java
public void acceptNewClient(Selector selector, SelectionKey key)
    throws IOException, ClosedChannelException {
    ServerSocketChannel server = (ServerSocketChannel) key.channel();
    SocketChannel channel = server.accept();
    channel.configureBlocking(false);
    SelectionKey readKey = channel.register(selector, SelectionKey.OP_READ);
    readKey.attach(new Client(readKey, q));
}

public void readDataFromSocket(SelectionKey key) throws IOException {
    int count = ((SocketChannel)key.channel()).read(byteBuffer);
    if ( count > 0) {
        byteBuffer.flip();
        byte[] data = new byte[count];
        byteBuffer.get(data, 0, count);
        ((Client)key.attachment()).write(data);
    } else if ( count < 0) {
        key.channel().close();
    }
    byteBuffer.clear();
}
```

这里使用的队列`Queue`，它提供了队列放数据（Client），这个也可以看做一个简单的线程池:

``` java
public class Queue extends LinkedList
{
	private int waitingThreads = 0;

	public synchronized void insert(Object obj)
	{
		addLast(obj);
		notify();
	}

	public synchronized Object remove()
	{
		if ( isEmpty() ) {
			try	{ waitingThreads++; wait();} 
			catch (InterruptedException e)	{Thread.interrupted();}
			waitingThreads--;
		}
		return removeFirst();
	}

	public boolean isEmpty() {
		return 	(size() - waitingThreads <= 0);
	}
}
```

这里我们创建的消费者线程数与建立的连接数是没有关系的，应该根据处理器的数量和请求的长度或持续时间进行调整，如果感觉处理请求的速度较慢，可以添加多一点线程，这样处理速度就会加快了。

`Client`类有两个用途。首先，通过把传入的非阻塞`IO`转换成可由`Servlet API`消费的阻塞 InputStream ，它解决了阻塞/非阻塞问题。其次，它管理特定客户端的请求状态。因为当全部读取消息时，非阻塞通道没有给出任何提示，所以强制我们在协议层处理这一情况。`Client`类在任意指定的时刻都指出了它是否正在参与进行中的请求。如果它准备处理新请求，`write()`方法就会为请求处理而将该客户端排到队列中。如果它已经参与了请求，它就只是使用 PipedInputStream 和 PipedOutputStream 类把传入的字节转换成一个 InputStream 。如图是处理的过程和转换：

![](/images/NIOWebServer/image1.gif)

在`Client`自己排队后，我们的消费者线程就可以消费它了。我们使用`RequestHandlerThread`类承来处理`Client`。至此，我们已经看到主线程是如何连续地循环的，它要么接受新客户机，要么读取新的 `I/O`。工作线程循环等待新请求。当客户机在请求队列上变为可用时，它就马上被`remove()`方法中阻塞的第一个等待线程所消费。`RequestHandlerThread`类代码如下：

``` java
	public void run() {
		while (true) {
			Client client = (Client) myQueue.remove();
			try {
				for (; ; ) {
					HttpRequest req = new HttpRequest(client.clientInputStream,
							myServletContext);
					HttpResponse res = new HttpResponse(client.key);
					defaultServlet.service(req, res);
					if (client.notifyRequestDone())
						break;
				}
			} catch (Exception e) {
				client.key.cancel();
				client.key.selector().wakeup();
			}
		}
	}
```

然后该线程创建新的`HttpRequest`和`HttpResponse`实例，并调用`defaultServlet`的 service 方法。注意，`HttpRequest`是用`Client`对象的`clientInputStream`属性构造的。 `PipedInputStream`就是负责把非阻塞`IO`转换成阻塞流。

从现在开始，请求处理就与我们在`J2EE Servlet API`中期望的相似。当对`servlet`的调用返回时，工作线程在返回到队列中之前，会检查是否有来自相同客户端的另一个请求可用。事实上，线程会对队列尝试另一个`remove()`调用，并变成阻塞，直到下一个请求可用。

# 性能

以下是在大量的连接下使用NIO的Web服务器与`Tomcat5.0`比较的结果，因为`Tomcat`使用的是标准IO创建的Web服务器。下面是一些说明：

1. Tomcat 是用最大的线程数量`2000`来配置的，而我们的服务器只允许用`4`个工作线程运行。

2. 每个服务器是针对相同的一组简单 HTTP get 测试的，这些 HTTP get 基本上由文本内容组成。
把加载工具（Microsoft Web Application Stress Tool）设置为使用“Keep-Alive”会话，导致了大约要为每个用户分配一个 socket。然后它导致了在 Tomcat 上为每个用户分配一个线程，而 NIO 服务器用固定数量的线程来处理相同的负载。

下图展示了在不断增加负载下的“请求/秒”率。在 200 个用户时，性能是相似的。但当用户数量超过 600 时，Tomcat 的性能开始急剧下降。这最有可能是由于在这么多的线程间切换上下文的开销而导致的。相反，基于`NIO`的服务器的性能则以线性方式下降。记住，Tomcat 必须为每个用户分配一个线程，而 NIO 服务器只配置有`4`个工作线程。

请求/秒
![](/images/NIOWebServer/image2.gif)

还有一张图也进一步显示了`NIO`的性能。它展示了操作的`Socket`连接错误数/分钟。同样，在大约 `600`个用户时，`Tomcat`的性能急剧下降，而基于`NIO`的服务器的错误率保持相对较低。

Socket 连接错误数/分钟
![](/images/NIOWebServer/image3.gif)

以上代码皆可以在我的github中找到。[https://github.com/heshangbuxitou/](https://github.com/heshangbuxitou/)

# 参考资料

1. [NIO Web服务器示例](http://yangyi.iteye.com/blog/699550)
2. [关于 NIO 你不得不知道的一些“地雷”](http://www.importnew.com/26563.html)
3. [谈谈 Tomcat 请求处理流程](http://www.importnew.com/27729.html#comment-638147)
4. [Apache Tomcat](https://zh.wikipedia.org/zh-hans/Apache_Tomcat)