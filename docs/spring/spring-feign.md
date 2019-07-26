# 简介
最近接手了公司的一个旧项目，框架使用的是传统的SSM架构。正值公司推行SpringCloud微服务之际，所以研究了下Feign，着手在传统的Spring项目中接入Feign。

# 介绍
> Feign是一个受到[Retrofit](https://github.com/square/retrofit)，[JAXRS-2.0](https://github.com/jax-rs)和[WebSocket](http://www.oracle.com/technetwork/articles/java/jsr356-1937161.html)的启发而诞生的HTTP请求框架，旨在降低JAVA使用HTTP请求的复杂性，开源地址[OpenFeign](https://github.com/OpenFeign/feign)。

# Feign的基础使用
* ## SpringBoot中使用
如果你的项目是SpringBoot项目的话，那么接入Feign会变的非常简单切优雅，如下：

```java
@FeignClient(value = "serviceId")
public interface ItemService {

    /**
     * <p>通过itemID获取商品详情</p>
     *
     * @param itemId 商品的id
     * @return {@link GetItemResponse} 返回值，其中{@code GetItemResponse.code} 为200时，正常
     * @since 2019-04-22
     */
    @PostMapping(value = {"/item/query/getItemDetailsById"})
    GetItemResponse getItemDetailsById(@RequestParam(name = "id") Long itemId);

}
```
引入必要的依赖并且在启动类或配置类上加入`@EnableFeignClients`注解即可，一切都是那么的简单且优雅。
* ## Spring中使用
  如果你的项目是Spring项目的话。那么你可以按照下面的方法使用Feign
 
```java
public interface ItemService {

    /**
     * <p>通过itemID获取商品详情</p>
     *
     * @param itemId 商品的id
     * @return {@link GetItemResponse} 返回值，其中{@code GetItemResponse.code} 为200时，正常
     * @since 2019-04-22
     */
    @RequestLine("GET /item/query/getItemDetailsById?id={id}")
    GetItemResponse getItemDetailsById(@Param(name = "id") Long itemId);

}
```
和上面类似，但是由于注解`@FeignClient` 依赖`org.springframework.cloud.openfeign.FeignAutoConfiguration`类，而自动配置是SpringBoot的特性。关于SpringBoot的自动配置，感兴趣的朋友可以阅读[此处](https://docs.spring.io/spring-boot/docs/current/reference/html/using-boot-auto-configuration.html)，这里就不再赘述了。

虽然无法使用自动配置来生成FeignClient，但是我们可以手动配置生成`Service`对应的`FeignClient`, 代码如下:
```java
    public static void main(String[] args) {
        ItemService itemService= Feign.builder()
                .decoder(new SpringDecoder(objectFactory()))
                .encoder(new SpringEncoder(objectFactory()))
                .client(new Client.Default(null, null))
                .target(ItemService.class,serviceUrl);
        System.out.println(JSON.toJSONString(itemService.getItemDetailsById(3837919L)));
    }
```
执行成功，结果如下：
![运行结果](https://sheapic.oss-cn-shenzhen.aliyuncs.com/1563280323%281%29.jpg)


* ## 结尾
至此，Feign的基础使用就到这里了，但是这样的调用方式明显不够优雅，并且对于为什么要这样构建FeignClient也没有说明，encoder decoder client的作用是什么？下一篇就聊一聊FeignClient的几个基础组件吧。




