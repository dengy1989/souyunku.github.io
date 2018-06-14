---
layout: post
title: Dubbo服务消费者调用过程
categories: dubbo
description: Dubbo服务消费者调用过程
keywords: dubbo 
---

Dubbo服务消费者调用过程

![这里写图片描述](http://www.ymq.io/images/2018/dubbo/dubbo_rpc_refer.jpg)

上图是服务消费的主过程：

首先通过`ReferenceConfig`类的`private void init()`方法会先检查初始化所有的配置信息后，调用`private T createProxy(Map<String, String> map)`创建代理，消费者最终得到的是服务的代理， 在`createProxy`接着调用`Protocol` 接口实现的`<T> Invoker<T> refer(Class<T> type, URL url)`方法生成`Invoker`实例(如上图中的红色部分)，这是服务消费的关键。接下来把`Invoker`通过`ProxyFactory`代理工厂转换为客户端需要的接口(如：`HelloWorld`)，创建服务代理并返回。

# 消费端的初始化过程

1、把服务引用的信息封装成URL并注册到zk注册中心;  
2、监听注册中心的服务的上下线;  
3、连接服务提供端，创建NettyClient对象;  
4、将这些信息包装成DubboInvoker消费端的调用链，创建消费端Invoker实例的服务代理并返回;

# 消费端的服务引用过程

1、经过负载均衡策略，调用提供者;  
2、选择其中一个服务的URL与提供者netty建立连接，使用ProxyFactory 创建远程通信，或者本地通信的，Invoker发到netty服务端;  
3、服务器端接收到该Invoker信息后，找到对应的本地Invoker，处理Invocation请求;  
4、获取异步，或同步处理结果;  

- 异步 不需要返回值：直接调用ExchangeClient.send()方法;
- 同步 需要返回值：使用ExchangeClient.request()方法，返回一个ResponseFuture，一直阻塞到服务端返回响应结果;

# 案例介绍

先看一个简单的客户端引用服务的例子，`HelloService`，`dubbo`配置如下：

```xml
<dubbo:application name="consumer-of-helloService" />
<dubbo:registry  protocol="zookeeper"  address="127.0.0.1:2181" />
<dubbo:protocol name="dubbo" port="20880" />
<dubbo:reference id="helloService" interface="com.demo.dubbo.service.HelloService" />
```

- 使用`Zookeeper`作为注册中心
- 引用远程的`HelloService`接口服务

根据之前的介绍，在Spring启动的时候，根据`<dubbo:reference>`配置会创建一个ReferenceBean，该bean又实现了Spring的FactoryBean接口，所以我们如下方式使用时：

```java
@Autowired
private HelloService helloService;
```

使用的不是ReferenceBean对象，而是ReferenceBean的getObject()方法返回的对象，该对象通过代理实现了HelloService接口，**所以要看服务引用的整个过程就需要从ReferenceBean.getObject()方法开始入手。**

## 服务引用过程

将ReferenceConfig.init()中的内容拆成具体的步骤，如下：

## 第一步：收集配置参数

```
methods=hello,
timestamp=1443695417847,
dubbo=2.5.3
application=consumer-of-helloService
side=consumer
pid=7748
interface=com.demo.dubbo.service.HelloService
```

## 第二步：从注册中心获取服务地址，返回Invoker对象

如果是单个注册中心，代码如下：

```java
Protocol refprotocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();

invoker = refprotocol.refer(interfaceClass, url);
```

上述url的内容如下：

```
registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?
application=consumer-of-helloService&
dubbo=2.5.6&
pid=8292&
registry=zookeeper&
timestamp=1443707173909&

refer=
    application=consumer-of-helloService&
    dubbo=2.5.6&
    interface=com.demo.dubbo.service.HelloService&
    methods=hello&
    pid=8292&
    side=consumer&
    timestamp=1443707173884&
```

## 第三步：使用ProxyFactory创建出Invoker的代理对象

```
ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();

proxyFactory.getProxy(invoker);
```

下面就详细说明下上述提到的几个概念：Protocol、Invoker、ProxyFactory

## 概念介绍

## Invoker

Invoker是一个可执行对象，有三种类型的Invoker：

1.  本地执行的Invoker（服务端使用）
2.  远程通信执行的Invoker（客户端使用）
3.  多个类型2的Invoker聚合成的集群版Invoker（客户端使用）

Invoker的实现情况如下：
![这里写图片描述](http://www.ymq.io/images/2018/dubbo/abstractInvoker-invoke.jpg)

先来看服务引用的第2个步骤，返回Invoker对象

对于客户端来说，Invoker应该是2、3这两种类型。先来说第2种类型，即远程通信的Invoker，看DubboInvoker的源码，调用过程AbstractInvoker.invoke()->doInvoke()：

```
    protected Result doInvoke(final Invocation invocation) throws Throwable {
        RpcInvocation inv = (RpcInvocation) invocation;
        final String methodName = RpcUtils.getMethodName(invocation);
        inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
        inv.setAttachment(Constants.VERSION_KEY, version);

        ExchangeClient currentClient;
        if (clients.length == 1) {
            currentClient = clients[0];
        } else {
            currentClient = clients[index.getAndIncrement() % clients.length];
        }
        try {
            boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
            boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
            int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY,Constants.DEFAULT_TIMEOUT);
            if (isOneway) {
                boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
                currentClient.send(inv, isSent);
                RpcContext.getContext().setFuture(null);
                return new RpcResult();
            } else if (isAsync) {
                ResponseFuture future = currentClient.request(inv, timeout) ;
                RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
                return new RpcResult();
            } else {
                RpcContext.getContext().setFuture(null);
                return (Result) currentClient.request(inv, timeout).get();
            }
        } catch (TimeoutException e) {
            throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        } catch (RemotingException e) {
            throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```

大致内容就是：
将通过远程通信将Invocation信息传递给服务器端，服务器端接收到该Invocation信息后，找到对应的本地Invoker，然后通过反射执行相应的方法，将方法的返回值再通过远程通信将结果传递给客户端。

这里分3种情况：

- 执行方法不需要返回值：直接调用`ExchangeClient.send()`方法。
- 执行方法的结果需要异步返回：使用`ExchangeClient.request()`方法返回一个`ResponseFuture`对象，通过`RpcContext`中的`ThreadLocal`使`ResponseFuture`和当前线程绑定，未等服务端响应结果就直接返回，然后服务端通过`ProtocolFilterWrapper.buildInvokerChain()`方法会调用`Filter.invoke()`方法，即`FutureFilter.invoker()->asyncCallback()`，会获取`RpcContext`的`ResponseFuture`对象，异步返回结果。
- 执行方法的结果需要同步返回：使用`ExchangeClient.request()`方法，返回一个`ResponseFuture`，一直阻塞到服务端返回响应结果

## Protocol

服务引用的第二步就是：

```
Protocol refprotocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();

invoker = refprotocol.refer(interfaceClass, url);
```

使用协议Protocol根据上述的url和服务接口来引用服务，创建出一个Invoker对象

默认实现的DubboProtocol也会经过ProtocolFilterWrapper、ProtocolListenerWrapper、RegistryProtocol的包装

首先看下RegistryProtocol.refer()方法，它干了哪些事呢？

- 将客户端的信息注册到注册中心上
- 创建一个RegistryDirectory，从注册中心中订阅自己引用的服务，将订阅到的url在RegistryDirectory内部转换成Invoker。
- RegistryDirectory是Directory的实现，Directory代表多个Invoker，可以把它看成List类型的Invoker，但与List不同的是，它的值可能是动态变化的，比如注册中心推送变更RegistryDirectory内部含有两者重要属性：

- 注册中心服务Registry
- Protocol 它会利用注册中心服务Registry来获取最新的服务器端注册的url地址，然后再利用协议Protocol将这些url地址转换成一个具有远程通信功能的Invoker对象，如DubboInvoker
- 在Directory的基础上使用Cluster将上述多个Invoker对象聚合成一个集群版的Invoker对象

**Directory和Cluster都是服务治理的重点**，接下去会单独拿一章出来讲

## ProxyFactory

服务引用的第三步就是：

```
proxyFactory.getProxy(invoker);
```

对于Server端，ProxyFactory主要负责将服务如HelloServiceImpl统一进行包装成一个Invoker，这些Invoker通过反射来执行具体的HelloServiceImpl对象的方法

对于client端，则是将上述创建的集群版Invoker（Cluster）创建出代理对象

代码如下：

```
public class JavassistProxyFactory extends AbstractProxyFactory {

    @SuppressWarnings("unchecked")
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
    }
```

可以看到是利用jdk自带的Proxy来动态代理目标对象Invoker，所以我们调用创建出来的代理对象如HelloService的方法时，会执行InvokerInvocationHandler中的逻辑：

```
public class InvokerInvocationHandler implements InvocationHandler {

    private final Invoker<?> invoker;

    public InvokerInvocationHandler(Invoker<?> handler){
        this.invoker = handler;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String methodName = method.getName();
        Class<?>[] parameterTypes = method.getParameterTypes();
        if (method.getDeclaringClass() == Object.class) {
            return method.invoke(invoker, args);
        }
        if ("toString".equals(methodName) && parameterTypes.length == 0) {
            return invoker.toString();
        }
        if ("hashCode".equals(methodName) && parameterTypes.length == 0) {
            return invoker.hashCode();
        }
        if ("equals".equals(methodName) && parameterTypes.length == 1) {
            return invoker.equals(args[0]);
        }
        //AbstractClusterInvoker.invoke()
        return invoker.invoke(new RpcInvocation(method, args)).recreate();
    }

}
```

参考：[http://dubbo.apache.org/books/dubbo-dev-book/implementation.html](http://dubbo.apache.org/books/dubbo-dev-book/implementation.html)  
参考：[https://my.oschina.net/xiaominmin/blog/1599378](https://my.oschina.net/xiaominmin/blog/1599378)

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io/2018/06/13/dubbo_rpc_refer](http://www.ymq.io/2018/06/13/dubbo_rpc_refer)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")
