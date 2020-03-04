---
layout: post
title: Spring Boot 程序的优雅停机[译文]
categories: [Spring Boot,spring-boot]
category: spring-boot
tags: [Spring Boot,spring-framework,spring-boot,tomcat,java,tutorial]
keywords: Java,spring-framework,spring-boot,tomcat,graceful-shutdown
---

今天，我读到 Marcos Barbero 在 2018 年的一篇旧文，介绍了 Spring Boot 程序的优雅停机。下面就是原文的翻译。

---

> 作者：Marcos Barbero<br/> 
> 原文网址：[https://blog.marcosbarbero.com/graceful-shutdown-spring-boot-apps/](https://blog.marcosbarbero.com/graceful-shutdown-spring-boot-apps/)

本文介绍 Spring Boot 程序的优雅停机处理过程

这篇文章起初是 [Andy Wilkinson](https://twitter.com/ankinson) 写的，后来我改为使用 Spring Boot 2 实现。 代码基于这段 [GitHub 评论](https://github.com/spring-projects/spring-boot/issues/4657#issuecomment-161354811)

## 介绍

许多开发者和架构师会讨论应用程序的设计、负载、框架以及设计模式，但是很少会有人讨论关于应用关闭的这个阶段。

我们考虑这种情况，某个应用程序的处理过程非常漫长并且需要关闭程序才能升级到新的版本。那么我们优雅地等待它处理完成之后再做关闭，而不是直接杀掉所有的连接，这样做会不会更好？

这就是本文所介绍的内容！

## 前置准备

- JDK 1.8
- 你所喜欢的 IDE
- Maven 3.0

## Spring Boot, Tomcat

实现此功能，首先要实现 `TomcatConnectorCustomizer`

```java
import org.apache.catalina.connector.Connector;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.web.embedded.tomcat.TomcatConnectorCustomizer;
import org.springframework.context.ApplicationListener;
import org.springframework.context.event.ContextClosedEvent;

import java.util.concurrent.Executor;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class GracefulShutdown implements TomcatConnectorCustomizer, ApplicationListener<ContextClosedEvent> {

    private static final Logger log = LoggerFactory.getLogger(GracefulShutdown.class);

    private static final int TIMEOUT = 30;

    private volatile Connector connector;

    @Override
    public void customize(Connector connector) {
        this.connector = connector;
    }

    @Override
    public void onApplicationEvent(ContextClosedEvent event) {
        this.connector.pause();
        Executor executor = this.connector.getProtocolHandler().getExecutor();
        if (executor instanceof ThreadPoolExecutor) {
            try {
                ThreadPoolExecutor threadPoolExecutor = (ThreadPoolExecutor) executor;
                threadPoolExecutor.shutdown();
                if (!threadPoolExecutor.awaitTermination(TIMEOUT, TimeUnit.SECONDS)) {
                    log.warn("Tomcat thread pool did not shut down gracefully within "
                            + TIMEOUT + " seconds. Proceeding with forceful shutdown");

                    threadPoolExecutor.shutdownNow();

                    if (!threadPoolExecutor.awaitTermination(TIMEOUT, TimeUnit.SECONDS)) {
                        log.error("Tomcat thread pool did not terminate");
                    }
                }
            } catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
            }
        }
    }

}
```

上面这段代码实现中，`ThreadPoolExecutor` 将会等待 30 秒后再关闭程序，很简单，对吧？现在还得把该 bean 注册到 `application context` 并且将其注入到 `Tomcat`

为此，只需要创建以下 Spring `@Bean`

```java
@Bean
public GracefulShutdown gracefulShutdown() {
    return new GracefulShutdown();
}

@Bean
public ConfigurableServletWebServerFactory webServerFactory(final GracefulShutdown gracefulShutdown) {
    TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
    factory.addConnectorCustomizers(gracefulShutdown);
    return factory;
}
```

## 如何测试？

为测试该实现，我创建了一个 `LongProcessController`，其中 `Thread.sleep` 10 秒钟

```java
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class LongProcessController {

    @RequestMapping("/long-process")
    public String pause() throws InterruptedException {
        Thread.sleep(10000);
        return "Process finished";
    }

}
```

现在，只要启动好你的 Spring Boot 程序，然后向 `/long-process` 发出一个请求，同时用 `SIGTERM` 杀掉程序

## 本地进程 ID

在你启动程序得时候，你可以启动日志中找到进程 ID 号，以我的为例 `6578`

```
2018-06-28 20:37:28.292  INFO 6578 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2018-06-28 20:37:28.296  INFO 6578 --- [           main] c.m.wd.gracefulshutdown.Application      : Started Application in 2.158 seconds (JVM running for 2.591)
```

## 请求和关闭

我使用 `curl` 来处理 `/long-process` 的请求：

```
$ curl -i localhost:8080/long-process
```

发送 `SIGTERM` 到进程：

```
$ kill 6578
```

以下是 `curl` 请求后的响应：

```
HTTP/1.1 200
Content-Type: text/plain;charset=UTF-8
Content-Length: 14
Date: Thu, 28 Jun 2018 18:39:56 GMT

Process finished
```

## 总结

恭喜，你已经学会如何在 Spring Boot 应用中优雅停机！

## 注脚

- 本文所使用的代码可以在 [Github](https://github.com/weekly-drafts/graceful-shutdown-spring-boot) 找到
- 本文实现基于该[评论](https://github.com/spring-projects/spring-boot/issues/4657#issuecomment-161354811)
