## **1.1 通过时间匹配**

配置：

```
1.server:  
2.  port: 8080  
3.spring:  
4.  application:  
5.    name: api-gateway  
6.  cloud:  
7.    gateway:  
8.      routes:  
9.        - id: gateway-service  
10.          uri: https://www.baidu.com  
11.          order: 0  
12.          predicates:  
13.            - After=2019-01-01T00:00:00+08:00[Asia/Shanghai]
```

上面的示例是指，请求时间在 2019年1月1日0点0分0秒之后的所有请求都转发到地址https://blog.csdn.net。+08:00是指时间和UTC时间相差八个小时，时间地区为Asia/Shanghai。

配置：

```
1.server:  
2.  port: 8080  
3.spring:  
4.  application:  
5.    name: api-gateway  
6.  cloud:  
7.    gateway:  
8.      routes:  
9.        - id: gateway-service  
10.          uri: https://www.baidu.com  
11.          order: 0  
12.          predicates:  
13.            - Before=2019-01-01T00:00:00+08:00[Asia/Shanghai]
```

表示在这个时间之前可以进行路由，在这时间之后停止路由，修改完之后重启项目再次访问地址[http://localhost:8080，页面会报](http://localhost:8080，页面会报) 404 没有找到地址。

配置：

```
1.server:  
2.  port: 8080  
3.spring:  
4.  application:  
5.    name: api-gateway  
6.  cloud:  
7.    gateway:  
8.      routes:  
9.        - id: gateway-service  
10.          uri: https://www.baidu.com  
11.          order: 0  
12.          predicates:  
13.            - Between=2019-01-01T00:00:00+08:00[Asia/Shanghai], 2019-07-01T00:00:00+08:00[Asia/Shanghai]
```

在这个时间段内可以匹配到此路由，超过这个时间段范围则不会进行匹配

### **1.2 通过Cookie匹配**

配置：Cookie Route Predicate 可以接收两个参数，一个是 Cookie name ,一个是正则表达式，路由规则会通过获取对应的 Cookie name 值和正则表达式去匹配，如果匹配上就会执行路由，如果没有匹配上则不执行。

```
1.server:  
2.  port: 8080  
3.spring:  
4.  application:  
5.    name: api-gateway  
6.  cloud:  
7.    gateway:  
8.      routes:  
9.        - id: gateway-service  
10.          uri: https://www.baidu.com  
11.          order: 0  
12.          predicates:  
13.            - Cookie=sessionId, test
```

使用 curl 测试，命令行输入:

```
curl http://localhost:8080 --cookie "sessionId=test"
```

则会返回页面代码，如果去掉–cookie "sessionId=test"，后台汇报 404 错误。

### **1.3 通过Header属性匹配**

配置：接收 2 个参数，一个 header 中属性名称和一个正则表达式，这个属性值和正则表达式匹配则执行

```
1.server:  
2.  port: 8080  
3.spring:  
4.  application:  
5.    name: api-gateway  
6.  cloud:  
7.    gateway:  
8.      routes:  
9.        - id: gateway-service  
10.          uri: https://www.baidu.com  
11.          order: 0  
12.          predicates:  
13.            - Header=X-Request-Id, \d+
```

使用 curl 测试，命令行输入:

```
curl http://localhost:8080  -H "X-Request-Id:88"
```

则返回页面代码证明匹配成功。将参数-H "X-Request-Id:88"改为-H "X-Request-Id:spring"再次执行时返回404证明没有匹配。

### **1.4 通过Host匹配**

配置：

```
1.server:  
2.  port: 8080  
3.spring:  
4.  application:  
5.    name: api-gateway  
6.  cloud:  
7.    gateway:  
8.      routes:  
9.        - id: gateway-service  
10.          uri: https://www.baidu.com  
11.          order: 0  
12.          predicates:  
13.            - Host=**.baidu.com
```

使用 curl 测试，命令行输入:

```
1.curl http://localhost:8080  -H "Host: www.baidu.com"   
2.curl http://localhost:8080  -H "Host: md.baidu.com"
```

经测试以上两种 host 均可匹配到 host\_route 路由，去掉 host 参数则会报 404 错误。

### **1.5 通过请求方式匹配**

配置：

```
1.server:  
2.  port: 8080  
3.spring:  
4.  application:  
5.    name: api-gateway  
6.  cloud:  
7.    gateway:  
8.      routes:  
9.        - id: gateway-service  
10.          uri: https://www.baidu.com  
11.          order: 0  
12.          predicates:  
13.            - Method=GET
```

使用 curl 测试，命令行输入:

```
1.# curl 默认是以 GET 的方式去请求  
curl http://localhost:8080
```

测试返回页面代码，证明匹配到路由，我们再以 POST 的方式请求测试。

```
1.# curl 默认是以 GET 的方式去请求  
2.curl -X POST http://localhost:8080
```

返回 404 没有找到，证明没有匹配上路由

### **1.6 通过请求路径匹配**

配置：

```
1.server:  
2.  port: 8080  
3.spring:  
4.  application:  
5.    name: api-gateway  
6.  cloud:  
7.    gateway:  
8.      routes:  
9.        - id: gateway-service  
10.          uri: http://ityouknow.com  
11.          order: 0  
12.          predicates:  
13.            - Path=/foo/{segment}
```

如果请求路径符合要求，则此路由将匹配，例如：/foo/1 或者 /foo/bar。

使用 curl 测试，命令行输入:

```
1.curl http://localhost:8080/foo/1  
2.curl http://localhost:8080/foo/xx  
3.curl http://localhost:8080/boo/xx
```

经过测试第一和第二条命令可以正常获取到页面返回值，最后一个命令报404，证明路由是通过指定路由来匹配。

### **1.7 通过请求参数匹配**

配置：

```
1.server:  
2.  port: 8080  
3.spring:  
4.  application:  
5.    name: api-gateway  
6.  cloud:  
7.    gateway:  
8.      routes:  
9.        - id: gateway-service  
10.          uri: https://www.baidu.com  
11.          order: 0  
12.          predicates:  
13.            - Query=smile
```

这样配置，只要请求中包含 smile 属性的参数即可匹配路由。

使用 curl 测试，命令行输入:

```
curl localhost:8080?smile=x&id=2
```

经过测试发现只要请求汇总带有 smile 参数即会匹配路由，不带 smile 参数则不会匹配。

还可以将 Query 的值以键值对的方式进行配置，这样在请求过来时会对属性值和正则进行匹配，匹配上才会走路由。

```
1.server:  
2.  port: 8080  
3.spring:  
4.  application:  
5.    name: api-gateway  
6.  cloud:  
7.    gateway:  
8.      routes:  
9.        - id: gateway-service  
10.          uri: https://www.baidu.com  
11.          order: 0  
12.          predicates:  
13.            - Query=keep, pu.
```

这样只要当请求中包含 keep 属性并且参数值是以 pu 开头的长度为三位的字符串才会进行匹配和路由。

使用 curl 测试，命令行输入:

```
curl localhost:8080?keep=pub
```

测试可以返回页面代码，将 keep 的属性值改为 pubx 再次访问就会报 404,证明路由需要匹配正则表达式才会进行路由。

### **1.8 通过请求ip地址进行匹配**

配置：

```
1.server:  
2.  port: 8080  
3.spring:  
4.  application:  
5.    name: api-gateway  
6.  cloud:  
7.    gateway:  
8.      routes:  
9.        - id: gateway-service  
10.          uri: https://www.baidu.com  
11.          order: 0  
12.          predicates:  
13.            - RemoteAddr=192.168.1.1/24
```

可以将此地址设置为本机的 ip 地址进行测试。

```
curl localhost:8080
```

如果请求的远程地址是 192.168.1.10，则此路由将匹配。

### **1.9 组合使用**

```
1.server:  
2.  port: 8080  
3.spring:  
4.  application:  
5.    name: api-gateway  
6.  cloud:  
7.    gateway:  
8.      routes:  
9.        - id: gateway-service  
10.          uri: https://www.baidu.com  
11.          order: 0  
12.          predicates:  
13.            - Host=**.foo.org  
14.            - Path=/headers  
15.            - Method=GET  
16.            - Header=X-Request-Id, \d+  
17.            - Query=foo, ba.  
18.            - Query=baz  
19.            - Cookie=chocolate, ch.p  
20.            - After=2018-01-20T06:06:06+08:00[Asia/Shanghai]
```

### **1.10 源码下载**

[示例代码-Github](https://github.com/meteor1993/SpringCloudLearning/tree/master/chapter12)

