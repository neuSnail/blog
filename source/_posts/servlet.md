---
title: servlet
date: 2019-03-28 21:24:37
tags: servlet filter interceptor
categories: spring
---

```java       

  Stream.of(Thread.currentThread().getStackTrace())
                .forEach(stackTraceElement -> System.out.println(stackTraceElement.toString()));
```

```java
test.TestController.index(TestController.java:24)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.base/java.lang.reflect.Method.invoke(Method.java:566)
org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:215)
org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:142)
org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:102)
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:895)
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:800)
org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87)
org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1038)
org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:942)
org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:998)
org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:890)
javax.servlet.http.HttpServlet.service(HttpServlet.java:634)
org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:875)
javax.servlet.http.HttpServlet.service(HttpServlet.java:741)
org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:231)
org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:53)
org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:99)
org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
org.springframework.web.filter.FormContentFilter.doFilterInternal(FormContentFilter.java:92)
org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
org.springframework.web.filter.HiddenHttpMethodFilter.doFilterInternal(HiddenHttpMethodFilter.java:93)
org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:200)
org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:199)
org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:96)
org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:490)
org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:139)
org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:92)
org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:74)
org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:343)
org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:408)
org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:66)
org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:770)
org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1415)
org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
java.base/java.lang.Thread.run(Thread.java:834)
```
