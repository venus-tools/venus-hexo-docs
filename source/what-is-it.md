---
layout:     post
title:      自研网关纳管Spring Cloud(一)
date:       2017-08-16 14:00:00 +0800
summary:    自研网关中间件整合Spring Cloud
toc: true
cutocnum: true
categories:
- Janus网关
tags:
- 网关中间件
---

**摘要**:  本文主要从网关的需求，以及Spring Cloud Zuul的线程模型和源码瓶颈分析结合，目前最近一段时间自研网关中间件纳管Spring Cloud的经验汇总整理。
 
 
## 一.自研网关纳管Spring Cloud的原因

### 1.1 为什么要自研网关

1.网关配置实时生效，配置灰度，回滚等
2.网关的性能，特别是防刷，限流，WAF等
3.动态Filter ，目前Zuul可以做到动态Filter，Filter配置下发，实时动态Filter
4.对网关的监控，告警，流量调拨，网关集群。
5.流程审计，增加Dsboard便捷的操作。

 <!-- more -->


### 1.2 回顾Web容器线程模型

Servlet只是基于Java技术的web组件，该组件由容器托管，用于生成动态内容。Servlet容器是web Server或application server 的一部分，供基于Request/Response发送模型的网络服务，解码基于MIME的请求，并格式化基于MIME的响应。Servlet容器包含并管理Servlet生命周期。典型的Servlet容器有Tomcat、Jetty。

<img src="/images/mw/gw/janus-03.png" width="650px" height="450px">

如上图所示，Tomcat基于NIO的多线程模型，如下图所示，其基于典型的Acceptor/Reactor线程模型，在Tomcat的线程模型中，Worker线程用来处理Request。当容器收到一个Request后，调度线程从Worker线程池中选出一个Worker线程，将请求传递给该线程，然后由该线程来执行Servlet的service()方法。且该worker线程只能同时处理一个Request请求，如果过程中发生了阻塞，那么该线程就会被阻塞，而不能去处理其他任务。 Servlet默认情况下一个单例多线程。

<img src="/images/mw/gw/janus-04.png" width="650px" height="450px">

回到zuul，zuul逻辑的入口是`ZuulServlet`.service(ServletRequest servletRequest, ServletResponse servletResponse)，ZuulServlet本质就是一个Servlet。

`RequestContext`提供了执行filter Pipeline所需要的Context，因为Servlet是`单例多线程`，这就要求RequestContext即要线程安全又要Request安全。context使用ThreadLocal保存，这样每个worker线程都有一个与其绑定的RequestContext，因为worker仅能同时处理一个Request，`这就保证了Request Context 即是线程安全的，又是Request安全的`。所谓Request 安全，即该Request的Context不会与其他同时处理Request冲突。 RequestContext继承了ConcurrentHashMap。

三个核心的方法preRoute(),route(), postRoute()，zuul对request处理逻辑都在这三个方法里，`ZuulServlet交给ZuulRunner去执行`。由于`ZuulServlet是单例`，因此`ZuulRunner也仅有一个实例`。

>因此综上所述，Spring Cloud Zuul的Qps在`1000-2000 `Qps之间是有原因的，网关作为如此重要的组件，基于如上所述的需求，觉得自研网关中间件纳管Spring Cloud很有必要。


## 二.自研网关纳管Spring Cloud

### 2.1 网关整合Spring Cloud服务治理体系

#### 2.1.1 整合服务治理体系思路

* 如果服务注册中心使用的是Eureka，可以不引入Spring Cloud Eureka相关的依赖，直接通过定时任务发起Eureka REST请求，网关自身维护一个缓存列表，自己写LB，找到服务列表转发。

> 优点：不需要引入Spring Cloud，对网关Server进行瘦身，洁癖讨厌各种引入无用的jar；
> 缺点: 注册中心使用Eureka，可以通过Eureka REST接口获取服务注册列表，但是换成ZK，Consul，或者Etcd，直接歇菜。


-------


* 通过集成Spring Cloud Common中高度抽象的DiscoveryClient。
> 优点: 通过高度抽象的DiscoveryClient，无需关心实现细节和定时任务去刷新注册列表。
> 缺点：换注册中心，需要相应的更换对应配置和依赖，一堆有些无关紧要的jar，需要自己对其瘦身。

#### 2.1.2 网关整合Spring Cloud Eureka
 
1.引入Spring Cloud Eureka Starter，排除不用的依赖，还需要努力瘦身ing。

```xml
  <dependency>
	   <groupId>org.springframework.cloud</groupId>
	  <artifactId>spring-cloud-starter-eureka</artifactId>
	   <version>1.3.1.RELEASE</version>
		 <exclusions>
				<exclusion>
					<groupId>org.springframework.cloud</groupId>
					<artifactId>spring-cloud-netflix-core</artifactId>
				</exclusion>
				<exclusion>
					<groupId>com.netflix.ribbon</groupId>
					<artifactId>ribbon-eureka</artifactId>
				</exclusion>
				<exclusion>
					<groupId>org.springframework.cloud</groupId>
					<artifactId>spring-cloud-starter-ribbon</artifactId>
				</exclusion>
				<exclusion>
					<groupId>org.springframework.cloud</groupId>
					<artifactId>
						spring-cloud-starter-archaius
					</artifactId>
				</exclusion>
				<exclusion>
				    <artifactId>hibernate-validator</artifactId>
                      <groupId>org.hibernate</groupId>
				</exclusion>
			</exclusions>
  </dependency>
```

2、同Zuul一样，把网关自身注册到Eureka Server上，目的是为了获取服务注册列表。

```
server.port=8082

spring.application.name=janus-server


eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
```

>PS:鄙视的一点就是，Spring Cloud应该提供一个轻量级的java client，配置注册中心的地址，还不需要把网关自身注册到注册中心上。原因是：网关中间件，不需要和服务治理框架耦合的很深。

#### 2.1.3 Netty Server与Spring Cloud内置的Server的整合
  
Netty Http Servert提供端口用于接收网关对外的请求，Spring Boot内置的server提供端口用于和Gateway-console交互，目前没找到Spring Boot内置Server和Netty Server合二为一的方法，但是一个服务暴露两个端口，很有必要。

```java
@SpringBootApplication
@EnableDiscoveryClient
public class JanusServerAppliaction {

	private static Logger logger = LoggerFactory.getLogger(JanusServerAppliaction.class);

	// 非SSL的监听HTTP端口
	public static int httpPort = 8081;

	public static void main(String[] args) throws Exception {

		//①先启动Spring Boot内置Server
		SpringApplication.run(JanusServerAppliaction.class, args);

		// logger.info("services: {}", context.getBean("discoveryClient",
		// DiscoveryClient.class).getServices());

		logger.info("Gateway Server Application Start...");
		// 解析启动参数
		parseArgs(args);

		// 初始化网关Filter和配置
		logger.info("init Gateway Server ...");
		JanusBootStrap.initGateway();

		logger.info("start netty  Server...");
		final JanusNettyServer gatewayServer = new JanusNettyServer();
		// ②启动HTTP容器
		gatewayServer.startServer(httpPort);

	}
}
```
> NettyServer服务启动后，阻塞监听端口,会导致集成spring boot内置Server启动无日志打印，spring Boot容器也没启动。因此注意启动顺序。

### 2.2 提高自研网关的QPS必杀技

#### 2.2.1 NettyServer初始化及启动代码

自研网关使用netty自带的线程池，共有三组线程池，分别为bossGroup、workerGroup和executorGroup，bossGroup用于接收客户端的TCP连接，workerGroup用于处理I/O等，executorGroup用于处理网关作业(执行Filter链)。

```java
public void startServer(int noSSLPort) throws InterruptedException {

		// http请求ChannelInbound
		final HttpInboundHandler httpInboundHandler = new HttpInboundHandler();

		ServerBootstrap insecure = new ServerBootstrap();
		insecure.group(bossGroup, workerGroup).channel(NioServerSocketChannel.class)
				// SO_REUSEADDR,表示允许重复使用本地地址和端口
				.option(ChannelOption.SO_REUSEADDR, Boolean.TRUE)
				.option(ChannelOption.ALLOCATOR, ByteBufManager.byteBufAllocator)
				/**
				 * SO_KEEPALIVE
				 * 该参数用于设置TCP连接，当设置该选项以后，连接会测试链接的状态，这个选项用于可能长时间没有数据交流的连接。当设置该选项以后，
				 * 如果在两小时内没有数据的通信时，TCP会自动发送一个活动探测数据报文。
				 */
				.childOption(ChannelOption.SO_KEEPALIVE, Boolean.TRUE)
				.childOption(ChannelOption.TCP_NODELAY, Boolean.TRUE)
				.childOption(ChannelOption.ALLOCATOR, ByteBufManager.byteBufAllocator)
				.childHandler(new ChannelInitializer<SocketChannel>() {
					@Override
					public void initChannel(SocketChannel ch) throws Exception {
						ChannelPipeline pipeline = ch.pipeline();
						// 对channel监控的支持 暂不支持
						// keepalive_timeout 的支持
						pipeline.addLast(
								new IdleStateHandler(ProperityConfig.keepAliveTimeout, 0,
										0, TimeUnit.MILLISECONDS));
						// pipeline.addLast(new JanusHermesHandler());
						pipeline.addLast(new HttpResponseEncoder());
						// 经过HttpRequestDecoder会得到N个对象HttpRequest,first HttpChunk,second
						// HttpChunk,....HttpChunkTrailer
						pipeline.addLast(new HttpRequestDecoder(
								ProperityConfig.maxInitialLineLength,
								ProperityConfig.maxHeaderSize, 8192,
								ProperityConfig.validateHeaders));
						// 把HttpRequestDecoder得到的N个对象合并为一个完整的http请求对象
						pipeline.addLast(new HttpObjectAggregator(
								ProperityConfig.httpAggregatorMaxLength));

						// gzip的支持
						if (ProperityConfig.gzip) {
							pipeline.addLast(new JanusHttpContentCompressor(
									ProperityConfig.gzipLevel,
									ProperityConfig.gzipMinLength));
						}

						pipeline.addLast(httpInboundHandler);
					}
				});

		ChannelFuture insecureFuture = insecure.bind(noSSLPort).sync();

		logger.info("[listen HTTP NoSSL][" + noSSLPort + "]");

		/**
		 * Wait until the server socket is closed.</br>
		 * 找到之前的无日志打印spring 容器也没启动的原因了，集成spring boot
		 * 和eureka放上放下并不是问题，是因为JanusNettyServer服务启动后，阻塞监听端口导致的
		 **/
		insecureFuture.channel().closeFuture().sync();
		logger.info("[stop HTTP NoSSL success]");

 }

```
#### 2.2.2 基于Netty Channel Pool实现REST的异步转发

RestInvokerFilter异步转发Filter，基于Netty Channel Pool实现REST的异步转发,提高自网关的性能的必杀技。

```java
public class RestInvokerFilter extends AbstractFilter {

	
	@Override
	public void run(final AbstractFilterContext filterContext,
			final JanusHandleContext janusHandleContext) throws Exception {
        
        // 自定义LB从Spring Cloud中服务注册缓存列表中获取服务实例
		ServiceInstance serviceInstance = SpringCloudHelper.getServiceInstanceByLB(
				janusHandleContext, janusHandleContext.getAPIInfo().getRouteServiceId());
		// 生成发送的Request对象
		FullHttpRequest outBoundRequest = getOutBoundHttpRequest(janusHandleContext);

		// 转发的时候设置LB获取到的主机IP和端口即可
		AsyncHttpRequest.builder()
				.remoteAddress(
						serviceInstance.getHost() + ":" + serviceInstance.getPort())
				.sessionContext(janusHandleContext)
				/**
				 * connection holding 500ms
				 */
				.holdingTimeout(ProperityConfig.janusHttpPoolOauthMaxHolding).build()
				.execute(new SimpleHttpCallback(janusHandleContext) {
					@Override
					public void onSuccess(FullHttpResponse result) {
						// testResult(result);
						janusHandleContext.setRestFullHttpResponse(result);
						// 跳转到下一个Filter
						filterContext.skipNextFilter(janusHandleContext);
					}

					@Override
					public void onError(Throwable e) {
						//省略
					}

					@Override
					public void onTimeout() {
						//省略
					}
				}, outBoundRequest);

	}
    
    //其余省略
}

```

## 三.自研网关Filter链的设计

一层接口，一层 abstract类，
一层基于Event观察者模式的抽象类，一个基于观察者模式的接口，
 自定义Filter根据需要继承处理，在这里不做过多介绍。

## 四.自研网关纳管Spring Cloud的结果

### 4.1 自研网关注册到Eureka Server上

把自研网关注册到Eureka Server上，用于获取服务列表，如下图所示。
<img src="/images/mw/gw/janus-01.png">

>上图中有两个服务提供者1，2，以及一个网关Server。

### 4.2 无缝支持REST转REST的GET和POST的转发

自定义LB，基于Netty Channel Pool实现了GET，POST的协议适配和异步转发，如下所示。

<img src="/images/mw/gw/janus-02.png" >

>http://localhost:8081/，是本地网关Server的主机和端口。

## 五.参考文章

[Netty系列之Netty线程模型](http://www.infoq.com/cn/articles/netty-threading-model/)












