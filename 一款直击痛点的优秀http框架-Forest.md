---
title: 一款直击痛点的优秀http框架-Forest
categories: Java
date: 2021-08-22 20:25:59
tags: Forest
---

# 一、背景

在涉及投放和变现业务时，时常需要对接第三方公司的数据接口获取我们所需的数据。这些公司都有提供基于http的api。但是每家公司提供api具体细节差别很大。有的基于`RESTFUL`规范，有的基于传统的http规范；有的需要在`header`里放置签名，有的需要`SSL`的双向认证，有的只需要`SSL`的单向认证；有的以`JSON` 方式进行序列化，有的以`XML`方式进行序列化。类似于这样细节的差别太多了。

<!-- more -->

不同的公司API规范不一样，这很正常。但是对于我来说，我如果想要代码变得优雅。我就必须解决一个痛点：

**不同服务商API那么多的差异点，如何才能维护一套不涉及业务的公共http调用套件。最好通过配置或者简单的参数就能区分开来。进行方便的调用？**

目前Java领域已经有很多优秀的大名鼎鼎的http开源框架可以实现任何形式的http调用，比如apache的`httpClient`包，非常优秀的`Okhttp`，JDK 自带的 `HttpURLConnection`标准库。

这些http开源框架的接口使用相对来说，都不太一样。不管选哪个在我这个场景里来说，我都不希望在调用每个第三方的http api时写上一堆http调用代码。我希望的是业务代码和http调用逻辑耦合度低。

# 二、传统的HTTP组件库

## HttpURLConnection 

使用标准库的最大好处就是不需要引入额外的依赖，但使用起来比较繁琐，就像直接使用 JDBC 连接数据库那样，需要很多模板代码。

```java
public class HttpUrlConnectionDemo {
    public static void main(String[] args) throws IOException {
        String urlString = "https://xxx.org/post";
        String bodyString = "password=123";

        URL url = new URL(urlString);
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();
        conn.setRequestMethod("POST");
        conn.setDoOutput(true);

        OutputStream os = conn.getOutputStream();
        os.write(bodyString.getBytes("utf-8"));
        os.flush();
        os.close();

        if (conn.getResponseCode() == HttpURLConnection.HTTP_OK) {
            InputStream is = conn.getInputStream();
            BufferedReader reader = new BufferedReader(new InputStreamReader(is));
            StringBuilder sb = new StringBuilder();
            String line;
            while ((line = reader.readLine()) != null) {
                sb.append(line);
            }
            System.out.println("响应内容:" + sb.toString());
        } else {
            System.out.println("响应码:" + conn.getResponseCode());
        }
    }
}
```

HttpURLConnection 发起的 HTTP 请求比较原始，基本上算是对网络传输层的一次浅层次的封装；有了 HttpURLConnection 对象后，就可以获取到输出流，然后把要发送的内容发送出去；再通过输入流读取到服务器端响应的内容；

## HttpClient

不过 HttpURLConnection 不支持 HTTP/2.0，为了解决这个问题，Java 9 的时候官方的标准库增加了一个更高级别的 HttpClient，再发起 POST 请求就显得高大上多了，不仅支持异步，还支持顺滑的链式调用。

```java
public class HttpClientDemo {
    public static void main(String[] args) throws URISyntaxException {
        HttpClient client = HttpClient.newHttpClient();
        HttpRequest request = HttpRequest.newBuilder()
                .uri(new URI("https://xxx.com/post"))
                .headers("Content-Type", "text/plain;charset=UTF-8")
                .POST(HttpRequest.BodyPublishers.ofString("牛逼"))
                .build();
        client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
                .thenApply(HttpResponse::body)
                .thenAccept(System.out::println)
                .join();
    }
}
```



## OkHttp

OkHttp 是一个执行效率比较高的 HTTP 客户端：

- 支持 HTTP/2.0，当多个请求对应同一个 Host 地址时，可共用同一个 Socket；
- 连接池可减少请求延迟；
- 支持 GZIP 压缩，减少网络传输的数据大小；
- 支持 Response 数据缓存，避免重复网络请求；

```java
public class OkHttpPostDemo {
    public static final MediaType JSON = MediaType.get("application/json; charset=utf-8");

    OkHttpClient client = new OkHttpClient();

    String post(String url, String json) throws IOException {
        RequestBody body = RequestBody.create(json, JSON);
        Request request = new Request.Builder()
                .url(url)
                .post(body)
                .build();
        try (Response response = client.newCall(request).execute()) {
            return response.body().string();
        }
    }

    public static void main(String[] args) throws IOException {
        OkHttpPostDemo example = new OkHttpPostDemo();
        String json = "{'name':'牛逼'}";
        String response = example.post("https://xxx.org/post", json);
        System.out.println(response);
    }
}
```



# 三、Forest

那今天介绍的这款轻量级的 HTTP 客户端框架 Forest，正是基于 Httpclient和OkHttp 的，屏蔽了不同细节的 HTTP 组件库所带来的所有差异。

Forest 的字面意思是森林的意思，更内涵点的话，可以拆成For和Rest两个单词，也就是“为了Rest”（Rest为一种基于HTTP的架构风格）。 而合起来就是森林，森林由很多树木花草组成（可以理解为各种不同的服务），它们表面上看独立，实则在地下根茎交错纵横、相互连接依存，这样看就有点现代分布式服务化的味道了。 最后，这两个单词反过来读就像是Resultful。

## Forest特点

 ![1.png](https://p.pstatp.com/origin/pgc-image/2a8d0d676c704bf28b17e1e6707a8d8c)

 ![2.png](https://p.pstatp.com/origin/pgc-image/6998054c49f7442e91da2dce57f62289)

## Forest使用

### pom添加一个 maven 依赖便可

```xml
<dependency>
    <groupId>com.dtflys.forest</groupId>
    <artifactId>forest-spring-boot-starter</artifactId>
    <version>1.5.0-RC7</version>
</dependency>
```

### 定义接口方法

在 Forest 依赖加入好之后，就可以构建 HTTP 请求的接口了。

在 Forest 中，所有的 HTTP 请求信息都要绑定到某一个接口的方法上，不需要编写具体的代码去发送请求。请求发送方通过调用事先定义好 HTTP 请求信息的接口方法，自动去执行 HTTP 发送请求的过程，其具体发送请求信息就是该方法对应绑定的 HTTP 请求信息。

```java
/**
 * 高德地图服务客户端接口
 */
@BaseRequest(baseURL = "http://ditu.amap.com")
public interface Amap {

    /**
     * 根据经纬度获取详细地址
     * @param longitude 经度
     * @param latitude 纬度
     * @return
     */
    @Request(url = "/service/regeo")
    Map getLocation(@Query("longitude") String longitude, @Query("latitude") String latitude);

    /**
     * 根据经纬度获取详细地址
     * @param coordinate 经纬度对象
     * @return
     */
    @Get(url = "/service/regeo")
    Map getLocation(@Query Coordinate coordinate);

    /**
     * 根据经纬度获取详细地址
     * @param coordinate 经纬度对象
     * @return
     */
    @Get(
        url = "/service/regeo",
        data = {
            "longitude=${coord.longitude}",
            "latitude=${coord.latitude}"
        }
    )
    Map getLocationByCoordinate(@DataVariable("coord") Coordinate coordinate);
}
```

### 扫描接口包

只要在`Spring Boot`的配置类或者启动类上加上`@ForestScan`注解，并在`basePackages`属性里填上远程接口的所在的包名

```java
@SpringBootApplication
@ForestScan(basePackages = "com.dtflys.forest.example.client")
public class ForestExampleApplication {
    public static void main(String[] args) {
        SpringApplication.run(ForestExampleApplication.class, args);
    }
}
```

Forest 会扫描`@ForestScan`注解中`basePackages`属性指定的包下面所有的接口，然后会将符合条件的接口进行动态代理并注入到 Spring 的上下文中。

### 调用接口

然后便能在其他代码中从 Spring 上下文注入接口实例，然后如调用普通接口那样调用即可。

```java
@RestController
public class ForestExampleController {

    @Autowired
    private Amap amap;

    @GetMapping("/amap/location")
    public Map amapLocation(@RequestParam BigDecimal longitude, @RequestParam BigDecimal latitude) {
        Map result = amap.getLocation(longitude.toEngineeringString(), latitude.toEngineeringString());
        return result;
    }

    @GetMapping("/amap/location2")
    public Map amapLocation2(@RequestParam BigDecimal longitude, @RequestParam BigDecimal latitude) {
        Coordinate coordinate = new Coordinate(
                longitude.toEngineeringString(),
                latitude.toEngineeringString());
        Map result = amap.getLocation(coordinate);
        return result;
    }

    @GetMapping("/amap/location3")
    public Map amapLocation3(@RequestParam BigDecimal longitude, @RequestParam BigDecimal latitude) {
        Coordinate coordinate = new Coordinate(
                longitude.toEngineeringString(),
                latitude.toEngineeringString());
        Map result = amap.getLocationByCoordinate(coordinate);
        return result;
    }
}
```

## 效果演示

启动ForestExample项目 ![3.png](https://p.pstatp.com/origin/pgc-image/fce284cbceb9444db5c054cfda8e260a)

调用http://127.0.0.1:8080/amap/location?longitude=114.085947&latitude=22.547传入深圳的经纬度，获取深圳的地图信息

![4.png](https://p.pstatp.com/origin/pgc-image/07515a9a985a42e39a70c3582c45862b)

详细的输出了接口的调用日志

参考项目地址https://gitee.com/dt_flys/forest-example.git

# 四、两个很棒的功能

这里分享这个框架2个我认为比较好的功能

## **模板表达式和参数的映射绑定功能**

模板表达式在使用的时候特别方便，举个栗子

```java
@Request(
    url = "${0}/send?un=${1}&pw=${2}&ph=${3}&ct=${4}",
    type = "get",
    dataType = "json"
)
public Map send(
    String base,
    String userName,
    String password,
    String phone,
    String content
);
```

上述是用序号下标进行取值，也可以通过名字进行取值：

```java
@Request(
    url = "${base}/send?un=${un}&pw=${pw}&ph=${3}&ct=${ct}",
    type = "get",
    dataType = "json"
)
public Map send(
    @DataVariable("base") String base,
    @DataVariable("un") String userName,
    @DataVariable("pw") String password,
    @DataVariable("ph") String phone,
    @DataVariable("ct") String content
);
```

甚至于可以这样简化写：

```java
@Request(
    url = "${base}/send",
    type = "get",
    dataType = "json"
)
public Map send(
    @DataVariable("base") String base,
    @DataParam("un") String userName,
    @DataParam("pw") String password,
    @DataParam("ph") String phone,
    @DataParam("ct") String content
);
```

**以上三种写法是等价的**

当然你也可以把参数绑定到header和body里去，你甚至于可以用一些表达式简单的把对象序列化成json或者xml：

```java
@Request(
    url = "${base}/pay",
      contentType = "application/json",
    type = "post",
    dataType = "json",
    headers = {"Authorization: ${1}"},
    data = "${json($0)}"
)
public PayResponse pay(PayRequest request, String auth);
```

## 对`HTTPS`的支持

以前用其他http框架处理https的时候，总觉得特别麻烦，尤其是双向证书。每次碰到问题也只能去baidu。然后根据别人的经验来修改自己的代码。

`Forest`对于这方面也想的很周到，底层完美封装了对https单双向证书的支持。也是只要通过简单的配置就能迅速完成。举个双向证书栗子：

```java
@Request(
    url = "${base}/pay",
      contentType = "application/json",
    type = "post",
    dataType = "json",
      keyStore = "pay-keystore",
      data = "${json($0)}"
)
public PayResponse pay(PayRequest request);
```

其中`pay-keystore`对应着`application.yml`里的`ssl-key-stores`

```yaml
forest:
  ...
  ssl-key-stores:
    - id: pay-keystore
      file: test.keystore
      keystore-pass: 123456
      cert-pass: 123456
      protocols: SSLv3
```

`Forest`有很多其他的功能设定，如果感兴趣的同学还请仔细去阅读文档和示例。

但是我想说的是，相信看到这里，很多人一定会说，这不就是`Feign`吗？

我在开发`Spring Cloud`项目的时候，也用过一段时间`Feign`，个人感觉`Forest`的确在配置和用法上和`Feign`的设计很像，但`Feign`的角色更多是作为`Spring Cloud`生态里的一个成员。充当RPC通信的角色，其承担的不仅是http通讯，还要对注册中心下发的调用地址进行负载均衡。

而`Forest`这个开源项目其定位则是一个高阶的http工具，主打友好和易用性。从使用角度出发，个人感觉`Forest`配置性更加简单直接。提供的很多功能也能解决很多人的痛点。

# 五、参考资料

https://github.com/dromara/forest

http://forest.dtflyx.com/

