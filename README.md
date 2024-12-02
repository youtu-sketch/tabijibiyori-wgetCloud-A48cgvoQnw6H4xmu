
传统的http1\.0请求开发，已经满足了我们日常的web开发。一般请求就像下图这样子，客服端发起一个请求（触发），服务端做出一个响应（动作）：![](https://img2024.cnblogs.com/blog/704073/202412/704073-20241202111526759-1037869510.png)


有时会有诸如实时刷新，实时显示的场景，我们往往是客户端定时发起请求，不断的尝试获取最新的数据。但是每次请求都会创建并释放一个新的连接，这样对于需要频繁请求的场景，性能损耗太大，此外对于实时性响应的场景也很难评估轮询周期。轮询的周期短，很多查询结果其实并没有变化，增加了成本开销。轮询周期长，又不能实时的展示数据，周期值变成了一个经验值，而且不同场景都需要不断的调整。这属实不够友好。于是http1\.1协议对此进行了扩展，允许长连接的存在。今天要介绍的SSE协议，就属于http1\.1下的新协议。SSE全称为 Sever\-Sent Event指服务器端事件发送。当客户端请求成功后，服务端会依次将事件（其实就是响应信息），分多次发送到客户端。客户端只要接收事件（响应信息），做出相应的处理即可。就像下图的样子：


![](https://img2024.cnblogs.com/blog/704073/202412/704073-20241202111639038-282342583.png)


比如K线增长图，实时热力图，各种增长曲线等等，都可以实时的，由后端主动将事件推送到前端，不再需要前端每次建立一个新的连接来请求。这种方式也称之为长连接。除了SSE，像websocket 、TCP等都属于长连接的类型。依次连接可以多次交互。SSE其实最初并不受重视，甚至很多人都不知道这个协议。如果是简单一点的话，通常直接多轮询几遍就解决问题了，如果是复杂一点的话，直接就使用websocket这样的重协议来处理了，功能也相对来说比较强大。但是自从交互大模型问世以后，大模型的流式对话往往能更高效的输出，这种流式输出的用户体验也更好。这种主要是侧重大模型响应的交互模式，(防盗连接：本文首发自https://github.com/jilodream/ )反而使得SSE的优势又体现出来了。下面我们看下如何在springboot中使用sse来开发：由于springboot的封装，我们使用SSE开发变得异常简单，核心思路是：创建一个 SseEmitter 对象，返回给前端这个SseEmitter类似于一个socket，我们只管向里边塞数据即可，而前端在收到SseEmitter对象后，则只管从sseEmitter中取数据即可。（注意此处一般采用注册响应方式）


后端代码如下：pom文件新增依赖：




```
1         <dependency>
2             <groupId>org.springframework.bootgroupId>
3             <artifactId>spring-boot-starter-webartifactId>
4         dependency>
```


controller类：




```
 1 package com.example.demo.learnsse;
 2 
 3 import lombok.extern.slf4j.Slf4j;
 4 import org.springframework.http.MediaType;
 5 import org.springframework.web.bind.annotation.CrossOrigin;
 6 import org.springframework.web.bind.annotation.GetMapping;
 7 import org.springframework.web.bind.annotation.RequestParam;
 8 import org.springframework.web.bind.annotation.RestController;
 9 import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;
10 
11 import java.io.IOException;
12 import java.util.concurrent.TimeUnit;
13 
14 /**
15  * @discription
16  */
17 @Slf4j
18 @RestController
19 @CrossOrigin(origins = "*")
20 public class SseController {
21 
22 
23     @GetMapping(value = "/learn/sseChat" , produces = {MediaType.TEXT_EVENT_STREAM_VALUE})
24     public SseEmitter chat(@RequestParam String name) throws IOException {
25         SseEmitter sseEmitter = new SseEmitter(360000L);
26         sseEmitter.onCompletion(() -> log.warn("sse complete!!!" + Thread.currentThread().getName()));
27         sseEmitter.onError(throwable -> {
28             log.warn("sse error  " + Thread.currentThread().getName(), throwable);
29         });
30         sseEmitter.send("start");
31         Runnable r = () -> {
32             int i = 1;
33             try {
34                 while (i <= 10) {
35                     sseEmitter.send(Thread.currentThread().getName()+": the next index:" + i);
36                     log.warn(Thread.currentThread().getName() + ":" + i);
37                     i++;
38                     TimeUnit.SECONDS.sleep(3);
39                 }
40                 sseEmitter.complete();
41             } catch (Exception e) {
42                 log.warn("catch a ex", e);
43                 sseEmitter.completeWithError(e);
44             }
45         };
46         Thread t = new Thread(r);
47         t.start();
48         log.warn("start return sse");
49         return sseEmitter;
50     }
51 }
```


我们可以不写前端，直接用浏览器或者命令行访问，


浏览器效果如下：


真实效果是一行行输出的




```
data:start

data:Thread-2: the next index:1

data:Thread-2: the next index:2

data:Thread-2: the next index:3

data:Thread-2: the next index:4

data:Thread-2: the next index:5

data:Thread-2: the next index:6

data:Thread-2: the next index:7

data:Thread-2: the next index:8

data:Thread-2: the next index:9

data:Thread-2: the next index:10
```


日志输出如下：




```
2024-12-02 11:06:36.267  WARN 2032 --- [nio-8081-exec-4] com.example.demo.learnsse.SseController  : sse complete!!!http-nio-8081-exec-4
2024-12-02 11:06:38.440  WARN 2032 --- [       Thread-2] com.example.demo.learnsse.SseController  : Thread-2:2
2024-12-02 11:06:41.442  WARN 2032 --- [       Thread-2] com.example.demo.learnsse.SseController  : Thread-2:3
2024-12-02 11:06:44.450  WARN 2032 --- [       Thread-2] com.example.demo.learnsse.SseController  : Thread-2:4
2024-12-02 11:06:47.458  WARN 2032 --- [       Thread-2] com.example.demo.learnsse.SseController  : Thread-2:5
2024-12-02 11:06:50.468  WARN 2032 --- [       Thread-2] com.example.demo.learnsse.SseController  : Thread-2:6
2024-12-02 11:06:53.471  WARN 2032 --- [       Thread-2] com.example.demo.learnsse.SseController  : Thread-2:7
2024-12-02 11:06:56.475  WARN 2032 --- [       Thread-2] com.example.demo.learnsse.SseController  : Thread-2:8
2024-12-02 11:06:59.483  WARN 2032 --- [       Thread-2] com.example.demo.learnsse.SseController  : Thread-2:9
2024-12-02 11:07:02.495  WARN 2032 --- [       Thread-2] com.example.demo.learnsse.SseController  : Thread-2:10
2024-12-02 11:07:05.508  WARN 2032 --- [nio-8081-exec-5] com.example.demo.learnsse.SseController  : sse complete!!!http-nio-8081-exec-5
```


这样一个简单的单次连接，服务器多次推送的示例就写完了。


当然你也可以写一个简短的前端代码，查看效果，注意此时涉及到跨域了，因此我们的java代码要使用注解@CrossOrigin(origins \= "\*") 来解决跨域，请看controller代码中红色字体




```
 1 
 2 
 3 
 4     SSE Example
 5 
 6 
 7     
 8     
32 
33 
```


我们在创建好SSE示例时，一般会设置以下几个回调方法：


onCompletion(Runnable callback)：当异步请求完成时，我们会调用此方法注册的回调函数。onError(Consumer callback) 当异步处理期间发生错误时，会调用该方法设置的回调函数


服务端发现任务结束时，主动知会客户端关闭连接：complete()：表示已经完成推送，通知客户端不再有新的事件发送。completeWithError(Throwable ex) 表示由于发生了某个异常而结束推送。springmvc将通过异常处理机制传递该异常。一般在对接大模型时，(防盗连接：本文首发自https://github.com/jilodream/ )我们除了完成SSE相关的注册，还会设置与大模型的连接，一般的思路是这样的：1、当前端发送请求提问来后端时，2、我们首先创建一个SseEmitter，作为未来发送的套接字，3、接着启动一个http连接，来请求大模型，4、此时我们会使用Reactor\-Mono之类的响应式编程框架，来回调处理大模型推送回来的数据。（其中Reactor部分的代码实现，由于篇幅有限，我会在后边的文章中讲解）5、在Mono的每次回调到大模型推送回来的数据时，我们通过SseEmitter发送给前端6、将第二步创建好的SseEmitter，返回给前端。注意3/4/5步都是作为异步回调注册到mono中的。整体的结构图如下：


![](https://img2024.cnblogs.com/blog/704073/202412/704073-20241202112327569-1502944797.png)


 


 本博客参考[wgetcloud全球加速服务机场](https://wa7.org)。转载请注明出处！
