---
layout: post
title: Dubbo服务提供者发布过程
categories: dubbo
description: Dubbo服务提供者发布过程
keywords: dubbo 
---

![服务提供者暴露服务的主过程](http://www.ymq.io/images/2018/dubbo/dubbo_rpc_export.jpg)

上图是服务提供者暴露服务的主过程

首先**ServiceConfig**类拿到对外提供服务的实际类ref(如：HelloServiceImpl),然后通过ProxyFactory类的getInvoker方法使用ref生成一个AbstractProxyInvoker实例，到这一步就完成具体服务到Invoker的转化。接下来就是Invoker转换到Exporter的过程。

Dubbo处理服务暴露的关键就在Invoker转换到Exporter的过程(如上图中的红色部分)，Dubbo协议的Invoker转为Exporter发生在DubboProtocol类的export方法，它主要是打开socket侦听服务，并接收客户端发来的各种请求，通讯细节由Dubbo自己实现.

# 服务发布过程大致分成3步

1、获取注册中心信息，构建协议信息，然后将其组合。  
2、通过ProxyFactory将HelloServiceImpl封装成一个Invoker执行 。  
3、使用Protocol将invoker导出成一个Exporter（包括去注册中心注册服务等）。  

这里面就涉及到几个大的概念，ProxyFactory、Invoker、Protocol、Exporter

# Export 服务暴露的步骤

1、首先会检查各种配置信息`<dubbo:service/>、<dubbo:registry/>、<dubbo:protocol/>   `等标签的配置，填充各种属性，总之就是保证我在开始暴露服务之前，所有的东西都准备好了，并且是正确的。   
2、加载所有的注册中心，因为我们暴露服务需要注册到注册中心中去。   
3、根据配置的所有协议和注册中心url分别进行导出。   
4、进行导出的时候，又是一波属性的获取设置检查等操作。   
5、如果配置的不是remote，则做本地导出。   
6、如果配置的不是local，则暴露为远程服务。   
7、不管是本地还是远程服务暴露，首先都会获取Invoker。   
8、获取完Invoker之后，转换成对外的Exporter，缓存起来。   
9、执行DubboProtocol类的export方法，打开socket侦听服务,并接收客户端发来的各种请求。   

## 概念介绍

先看一个简单的服务端例子，dubbo配置如下：

```xml
<dubbo:application name="helloService-app" />
<dubbo:registry protocol="zookeeper" address="127.0.0.1:2181" />
<dubbo:service interface="com.demo.dubbo.service.HelloService" ref="helloService" />
<dubbo:protocol name="dubbo" port="20880" />
<bean id="helloService" class="com.demo.dubbo.server.serviceimpl.HelloServiceImpl"/>
```

- 有一个服务接口HelloService，以及它对应的实现类HelloServiceImpl。
- 将HelloService标记为dubbo服务，使用HelloServiceImpl对象来提供具体的服务。
- 使用zooKeeper作为注册中心。

## Invoker

Invoker，一个可执行对象，能够根据方法名称、参数得到相应的执行结果。接口如下：

```java
public interface Invoker<T> {

    Class<T> getInterface();

    URL getUrl();

    Result invoke(Invocation invocation) throws RpcException;

    void destroy();

}
```

而Invocation则包含了需要执行的方法、参数等信息，接口如下：

```java
public interface Invocation {

    URL getUrl();

    String getMethodName();

    Class<?>[] getParameterTypes();

    Object[] getArguments();

}
```

目前其实现类只有一个RpcInvocation

Invoker这个可执行对象的执行过程分成三种类型：

1. 本地执行的Invoker
2. 远程通信执行的Invoker
3. 多个类型2的Invoker聚合成的集群版Invoker

以HelloService接口为例：

1. **本地执行的Invoker**：**server端**，含有对应的HelloServiceImpl实现，要执行该接口方法，仅仅只需要通过反射执行HelloServiceImpl对应的方法即可。
2. **远程通信执行的Invoker**： **client端**，要想执行该接口方法，需要需要进行远程通信，发送要执行的参数信息给server端；server端，利用上述本地执行的Invoker执行相应的方法，然后将返回的结果发送给client端。这整个过程算是该类Invoker的典型的执行过程。
3. **集群版的Invoker**：**client端**，拥有某个服务的多个Invoker，此时client端需要做的就是将多个Invoker聚合成一个集群版的Invoker，client端使用的时候，仅仅通过集群版的Invoker来进行操作。集群版的Invoker会从众多的远程通信类型的Invoker中选择一个来执行（从中加入负载均衡、服务降级等策略），类似服务治理，dubbo已经实现了、

看下Invoker的实现情况：
![Invoker的实现情况](http://www.ymq.io/images/2018/dubbo/dubbo-invoker.jpg)

## ProxyFactory

对于Server端，主要负责将服务如HelloServiceImpl统一进行包装成一个Invoker，通过反射来执行具体的HelloServiceImpl对象的方法

接口定义如下：

```java
@SPI("javassist")
public interface ProxyFactory {

     //针对client端，创建出代理对象
    @Adaptive({Constants.PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker) throws RpcException;

    //针对server端，将服务对象如HelloServiceImpl包装成一个Invoker对象
    @Adaptive({Constants.PROXY_KEY})
    <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException;

}
```

ProxyFactory的接口实现有JdkProxyFactory、JavassistProxyFactory，默认是JavassistProxyFactory， JavassistProxyFactory内容如下：

```java
public class JavassistProxyFactory extends AbstractProxyFactory {

    @SuppressWarnings("unchecked")
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
    }

    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        // TODO Wrapper类不能正确处理带$的类名
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName, 
                                      Class<?>[] parameterTypes, 
                                      Object[] arguments) throws Throwable {
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    }

}
```

可以看到创建了一个AbstractProxyInvoker（这类就是本地执行的Invoker），AbstractProxyInvoker对invoke()方法的实现如下：

```java
public Result invoke(Invocation invocation) throws RpcException {
    try {
        return new RpcResult(doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments()));
    } catch (InvocationTargetException e) {
        return new RpcResult(e.getTargetException());
    } catch (Throwable e) {
        throw new RpcException("Failed to invoke remote proxy method " + invocation.getMethodName() + " to " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```

综上所述，服务发布的第二个过程就是：使用ProxyFactory将HelloServiceImpl封装成一个本地执行的Invoker。

## Protocol

从上面得知服务发布的第一、二个过程：

1、获取注册中心信息dubbo:registry和协议信息dubbo:protocol。
2、使用ProxyFactory将HelloServiceImpl封装成一个本地执行的Invoker。

**执行这个服务->执行这个本地的Invoker->调用AbstractProxyInvoker.invoke(Invocation invocation)方法，方法的执行过程就是通过反射执行HelloServiceImp**l。

**现在的问题是：客户端如何调用服务端的方法（服务注册到注册中心->客户端向注册中心订阅服务->客户端调用服务端的方法）和上述Invocation参数的来源问题**。

对于Server端来说，上述服务发布的第3步中Protocol要解决的问题是：
- 根据指定协议向注册中心注册HelloService服务。
- 当客户端根据协议调用这个服务时，将客户端传递过来的Invocation参数交给上述的Invoker来执行。
- 所以Protocol会加入远程通信这块，根据客户端的请求来获取参数Invocation。

先来看下Protocol接口的定义：

```java
@SPI("dubbo")
public interface Protocol {

    int getDefaultPort();

    //针对server端来说，将本地执行类的Invoker通过协议暴漏给外部。这样外部就可以通过协议发送执行参数Invocation，然后交给本地Invoker来执行
    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    //这个是针对客户端的，客户端从注册中心获取服务器端发布的服务信息
    //通过服务信息得知服务器端使用的协议，然后客户端仍然使用该协议构造一个Invoker。这个Invoker是远程通信类的Invoker。
    //执行时，需要将执行信息通过指定协议发送给服务器端，服务器端接收到参数Invocation，然后交给服务器端的本地Invoker来执行
    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

    void destroy();

}
```

我们再来详细看看服务发布的第3步(ServiceConfig)：

```
Exporter<?> exporter = protocol.export(invoker);
```

protocol的来历是：

```
Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

```java
public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException{
    if (arg0 == null) {
        throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null"); 
    }
    if (arg0.getUrl() == null) {
        throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
    }
    com.alibaba.dubbo.common.URL url = arg0.getUrl();
    String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
    if(extName == null) {
        throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])"); 
    }
    com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)com.alibaba.dubbo.common.ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
    return extension.export(arg0);
}

public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0,com.alibaba.dubbo.common.URL arg1) throws com.alibaba.dubbo.rpc.RpcException{
    if (arg1 == null) { 
        throw new IllegalArgumentException("url == null"); 
    }
    com.alibaba.dubbo.common.URL url = arg1;
    String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
    if(extName == null) {
        throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])"); 
    }
    com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)com.alibaba.dubbo.common.ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
    return extension.refer(arg0, arg1);
}
```

export(Invoker invoker)的过程即根据Invoker中url的信息来最终选择Protocol的实现，默认实现是DubboProtocol，然后再对DubboProtocol进行依赖注入，进行wrap包装（getExtension()方法）。

Protocol的实现情况：
![这里写图片描述](http://www.ymq.io/images/2018/dubbo/dubbo-protocol.jpg) 
可以看到在返回DubboProtocol之前，经过了ProtocolFilterWrapper、ProtocolListenerWrapper、RegistryProtocol的包装

所谓包装就是如下内容：

```java
package com.alibaba.xxx;

import com.alibaba.dubbo.rpc.Protocol;

public class XxxProtocolWrapper implemenets Protocol {
    Protocol impl;

    public XxxProtocol(Protocol protocol) { impl = protocol; }

    // 接口方法做一个操作后，再调用extension的方法
    public Exporter<T> export(final Invoker<T> invoker) {
        //... 一些操作
        impl .export(invoker);
        // ... 一些操作
    }

    // ...
}
```

使用装饰器模式，类似AOP的功能

下面主要讲解RegistryProtocol和DubboProtocol，先暂时忽略ProtocolFilterWrapper、ProtocolListenerWrapper

**RegistryProtocol.export() 主要功能是将服务注册到注册中心**

**DubboProtocol.export() 服务导出功能：**

- 创建一个DubboExporter，封装Invoker。
- 根据Invoker的url获取ExchangeServer通信对象（负责与客户端的通信模块）。

现在我们搞清楚我们的目的，通过通信对象获取客户端传来的Invocation参数，然后找到对应的DubboExporter（即能够获取到本地Invoker）就可以执行服务了。

在DubboProtocol中，每个ExchangeServer通信对象都绑定了一个ExchangeHandler对象，内容如下：

```java
private ExchangeHandler requestHandler = new ExchangeHandlerAdapter() {

    public Object reply(ExchangeChannel channel, Object message) throws RemotingException {
        if (message instanceof Invocation) {
            Invocation inv = (Invocation) message;
            Invoker<?> invoker = getInvoker(channel, inv);
            RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
            ......
            return invoker.invoke(inv);
        }
        throw new RemotingException(channel, "Unsupported request: " + message == null ? null : (message.getClass().getName() + ": " + message) + ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress());
    }
};
```

可以看到该类才是真正与客户端通信，在获取到Invocation参数后，调用getInvoker()来获取本地的Invoker（先从exporterMap中获取Exporter），就可以调用服务了。

而对于通信这块，接下来会专门来详细的说明，从reply参数可知，重点在了解**ExchangeChannel**。

## Exporter

负责维护invoker的生命周期，包含一个Invoker对象，接口定义如下：

```java
public interface Exporter<T> {

    Invoker<T> getInvoker();

    void unexport();

}
```

# 结束语

以上就是本文简略地介绍了及服务发布过程中的几个 ProxyFactory、Invoker、Protocol、Exporter 概念

参考：[http://dubbo.apache.org/books/dubbo-dev-book/implementation.html](http://dubbo.apache.org/books/dubbo-dev-book/implementation.html)  
参考：[https://blog.csdn.net/qq418517226/article/details/51818769](https://blog.csdn.net/qq418517226/article/details/51818769)

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io/2018/06/13/dubbo_rpc_export](http://www.ymq.io/2018/06/13/dubbo_rpc_export)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")
