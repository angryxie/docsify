# 前文提要：
上一篇我们简单演示了Feign在SpringCloud和非SpringCloud下的使用。但是眼尖的朋友可能会发现了两种使用方式有一些不一样。SpringCloud下使用明显要简单的多，这其中的差距体现在下面这段代码中。
```java
ItemService itemService= Feign.builder()
            .decoder(new SpringDecoder(objectFactory()))
            .encoder(new SpringEncoder(objectFactory()))
            .client(new Client.Default(null, null))
            .target(ItemService.class,serviceUrl); 
```
在这篇中，我们就来聊一下Feign的基础组件。

# Feign的基础组件构成：
根据上面的代码我们可以看到FeignClient的构建使用的是建造者模式，查看FeignBuilder的源码：
```java
 private final List<RequestInterceptor> requestInterceptors =
        new ArrayList<RequestInterceptor>();
    private Logger.Level logLevel = Logger.Level.NONE;
    private Contract contract = new Contract.Default();
    private Client client = new Client.Default(null, null);
    private Retryer retryer = new Retryer.Default();
    private Logger logger = new NoOpLogger();
    private Encoder encoder = new Encoder.Default();
    private Decoder decoder = new Decoder.Default();
    private ErrorDecoder errorDecoder = new ErrorDecoder.Default();
    private Options options = new Options();
    private InvocationHandlerFactory invocationHandlerFactory =
        new InvocationHandlerFactory.Default();
    private boolean decode404;
```
这里包含了FeignClient的大部分组件，大部分组件大多数时候我们并不需要自定义，以下组件是我们需要关系的：
* Contract
* Decoder
* Encoder
* Client

下面我们来一一聊一聊这些组件。
## Contract
顾名思义这个组件和约定有关，回看上一篇我们使用Feign的方式，在方法注解上，是不一样的，SpringCloud上使用的是`@RequestMapping`，非SpringCloud上使用的是`@RequestLine`。其实Contract就是Feign支持的注解。Contract接口只有一个方法：

```java
List<MethodMetadata> parseAndValidatateMetadata(Class<?> targetType);
```
格式化并且校验注解以链接到对应的HTTP请求。其实`@RequestLine`才是Feign默认的注解，那为什么SpringCloud可以支持SpringMvc的注解呢。我们查看Feign在SpringCloud下的自动配置文件，可以看到一行`org.springframework.cloud.openfeign.FeignClientsConfiguration.Configuration=`
查看这里指向的类，里面有一个类被注册为bean：

```java
@Bean
@ConditionalOnMissingBean
public Contract feignContract(ConversionService feignConversionService) {
    return new SpringMvcContract(this.parameterProcessors, feignConversionService);
}
```
可以看到SpringCloud默认配置的是SpringMvcContract，这就解释了为什么我们在SpringCloud的项目中可以直接使用SpringMvc的注解了，我想也是Spring的工程师也是因为SpringMvc的注解大家更熟悉才会放弃Feign自有的注解解析器吧。如果要在非Spring项目中的Feign使用SpringMvc的注解，只需要自定义Contract即可，如下：
```java
Feign.builder().contract(new SpringMvcContract())
```

## Encoder和Decoder
这两个组件很容易理解，编码器和解码器。如果你有了解过Netty的话，应该会对这两个组件很熟悉。其中Encode和Decode接口都只有一个方法：
```java
void encode(Object object, Type bodyType, RequestTemplate template) throws EncodeException;
```
和
```java
Object decode(Response response, Type type) throws DecodeException, FeignException;
```
我们可以用不同的实现来自定义自己的解编码器，不过大多数时候我们可以使用现有的实现。比如上一篇我们使用的SpringEnDecoder。查看SpringEncoder的源码，主要代码如下:
```java
for (HttpMessageConverter<?> messageConverter : this.messageConverters
					.getObject().getConverters()) {
    if (messageConverter.canWrite(requestType, requestContentType)){
        if (log.isDebugEnabled()) {
            if (requestContentType != null) {
                log.debug("Writing [" + requestBody + "] as \""
                        + requestContentType + "\" using ["
                        + messageConverter + "]");
            }
            else {
                log.debug("Writing [" + requestBody + "] using ["
                        + messageConverter + "]");
            }

        }
    FeignOutputMessage outputMessage = new FeignOutputMessage(request);
        try {
            @SuppressWarnings("unchecked")
            HttpMessageConverter<Object> copy =  
                                    (HttpMessageConverter<Object>) messageConverter;
            copy.write(requestBody, requestContentType, outputMessage);
        }
        catch (IOException ex) {
            throw new EncodeException("Error converting request body", ex);
        }
    }
```
很容易理解，循环解码器内所有的消息转换器，如果改转换器支持此类消息，则进行转换。
`messageConverter.canWrite(requestType, requestContentType)`，关于这里，以后有机会可以再详细聊，比如Feign默认支持的格式是json的，但是如果我们要对接老旧项目的话可能需要支持form-data的格式等。

所以我们可以选择根据需要选择合适的messageConverter，默认Feign对于json格式的转换器是Jackson，对于很多的json解析工具，有它自己默认的实现，比如，我们可以使用Fastjson消息转换器来覆盖掉默认的配置，代码如下：

```java
@Bean
public ResponseEntityDecoder feignDecode() {
    HttpMessageConverter httpMessageConverter =    fastJsonHttpMessageConverter();
    return new ResponseEntityDecoder(new SpringDecoder(() -> new HttpMessageConverters(httpMessageConverter)));
}

@Bean
public FastJsonHttpMessageConverter fastJsonHttpMessageConverter() {
    FastJsonHttpMessageConverter fastJsonConverter = new FastJsonHttpMessageConverter();
    List<MediaType> supportedMediaTypes = new ArrayList<>();
    supportedMediaTypes.add(MediaType.APPLICATION_JSON);
    supportedMediaTypes.add(MediaType.APPLICATION_JSON_UTF8);
    supportedMediaTypes.add(MediaType.APPLICATION_ATOM_XML);
    supportedMediaTypes.add(MediaType.APPLICATION_FORM_URLENCODED);
    supportedMediaTypes.add(MediaType.APPLICATION_OCTET_STREAM);
    supportedMediaTypes.add(MediaType.APPLICATION_PDF);
    supportedMediaTypes.add(MediaType.APPLICATION_RSS_XML);
    supportedMediaTypes.add(MediaType.APPLICATION_XHTML_XML);
    supportedMediaTypes.add(MediaType.APPLICATION_XML);
    supportedMediaTypes.add(MediaType.IMAGE_GIF);
    supportedMediaTypes.add(MediaType.IMAGE_JPEG);
    supportedMediaTypes.add(MediaType.IMAGE_PNG);
    supportedMediaTypes.add(MediaType.TEXT_EVENT_STREAM);
    supportedMediaTypes.add(MediaType.TEXT_HTML);
    supportedMediaTypes.add(MediaType.TEXT_MARKDOWN);
    supportedMediaTypes.add(MediaType.TEXT_PLAIN);
    supportedMediaTypes.add(MediaType.TEXT_XML);
    fastJsonConverter.setSupportedMediaTypes(supportedMediaTypes);
    FastJsonConfig config = new FastJsonConfig();
    config.setSerializerFeatures(SerializerFeature.WriteNullStringAsEmpty, SerializerFeature.WriteNullListAsEmpty, SerializerFeature.WriteNullBooleanAsFalse, SerializerFeature.DisableCircularReferenceDetect);
    fastJsonConverter.setFastJsonConfig(config);
    return fastJsonConverter;
}

@Bean
public HttpMessageConverters httpMessageConverters() {
    return new HttpMessageConverters(fastJsonHttpMessageConverter());
}

```

Decoder的实现类似，这里不再进行赘述。

## Client
client是Feign最重要的组件，也是最基础的组件，它的作用很明显是用于HTTP交互的客户端。查看Feign的源码，client接口只有一个`Response execute(Request request, Options options) throws IOException;`方法，其中有个默认的实现
```java
class Default implements Client {

    private final SSLSocketFactory sslContextFactory;
    private final HostnameVerifier hostnameVerifier;

    /**
     * Null parameters imply platform defaults.
     */
    public Default(SSLSocketFactory sslContextFactory, HostnameVerifier hostnameVerifier) {
      this.sslContextFactory = sslContextFactory;
      this.hostnameVerifier = hostnameVerifier;
    }

    @Override
    public Response execute(Request request, Options options) throws IOException {
      HttpURLConnection connection = convertAndSend(request, options);
      return convertResponse(connection, request);
    }
```
查看`excute`方法可以看到，Feign默认使用`HttpURLConnection`来进行HTTP请求。`HttpURLConnection`的使用api非常不友好，并且配置项少，协议也只支持HTTP1.0/1.1,所以大部分时候我们会选择使用其它的client，比如[OkHttp](https://square.github.io/okhttp/),这里尝试使用`OkHttp3`来替换默认的client。

代码如下：
```java
@Bean
public HystrixFeign.Builder feignClientBuilder() {
    return HystrixFeign.builder().client(feignOkHttpClient()).encoder(new SpringEncoder(objectFactory())).decoder(new SpringDecoder(objectFactory()));
}


@Bean
public feign.okhttp.OkHttpClient feignOkHttpClient() {
    return new feign.okhttp.OkHttpClient(new OkHttpClient.Builder().readTimeout(30, TimeUnit.SECONDS).connectTimeout(30, TimeUnit.SECONDS).build());
}
```

如果你直接使用OkHttp3来实现Client的话，会发现它们两者的request和response会有些许差别，转换起来稍有些麻烦，不过这点OkHttp已经帮你想好了所以提供了现有的FeignOkHttpClient，只需要引入jar包即可，maven依赖如下：
```maven
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-okhttp</artifactId>
    <version>10.2.3</version>
</dependency>
```
# 总结
到这里对Feign的基础组件就介绍完了，其实Feign的组件做的事情就是HTTP请求应用层所做的事情。但是非SpringCloud使用Feign会有一个很大的缺陷，因为应用没有注册到eureka，所以无法直接从eureka获取实例的服务地址，所以在生成FeignClient的时候我们传入的是url，即实例的地址，这样的缺点就是负载均衡依赖外部实现，比如nginx，zuul等。

其实非Spring项目也可以注册到eureka并且从eureka获取实例的服务地址，下一篇我们就来聊聊这里怎么实现。
