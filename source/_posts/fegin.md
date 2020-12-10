---
title: feign 同一个服务编写多个远程调用实例 解决办法
date: 2019-01-16 15:04:21
tags: [spring,feign]
categories: [java,spring]
---
# 问题
	在微服务架构中，当我们需要进行服务间调用时可以选择feign组件，现在遇到的问题是：
		当同一个服务，声明多个feign实例时，启动时直接报错
**错误信息**

```
Cannot define alias 'basic-dataFeignClient' for name 'com.csbr.pharmacy.chain.cloud.service.operation.OperationFeignClient': It is already registered for name 'com.csbr.pharmacy.chain.cloud.service.BasicDataCommonFeignClient'.
```
猜测是因为@FeignClient注解的处理类在生成代理类，把代理类放入spring工厂中时，指定的名字规则就是   服务名+FeignClient，所以当有多个实例时，直接实例化失败
# 解决办法
> 通过 Feign.builder() 手动生成代理类
> 
**配置类**
```java
package com.csbr.pharmacy.chain.cloud.configurations;

import com.csbr.pharmacy.chain.cloud.service.operation.OperationFeignClient;
import com.netflix.appinfo.InstanceInfo;
import com.netflix.discovery.EurekaClient;
import feign.Contract;
import feign.Feign;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.openfeign.FeignContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @author kangxuan
 * @date 2019/1/16 0016 11:13.
 * @Description: 自定义 feign实例 配置
 */
@Configuration
public class FeignClientConfig {
	/**
	* FeignClientFactoryBean 该工厂类中 设置builder属性时就是通过该对象，源码中可看到
	*/
    @Autowired
    private FeignContext feignContext;

	/**
	* 通过注入Eureka实例对象，就不用手动指定url，只需要指定服务名即可
	*/
    @Autowired
    private EurekaClient eurekaClient;


    private <T> T create(Class<T> clazz,String serverId){
        InstanceInfo nextServerFromEureka = eurekaClient.getNextServerFromEureka(serverId,false);
        return Feign.builder()
                .encoder(feignContext.getInstance(serverId,feign.codec.Encoder.class))
                .decoder(feignContext.getInstance(serverId,feign.codec.Decoder.class))
                .contract(feignContext.getInstance(serverId, Contract.class))
                .target(clazz, nextServerFromEureka.getHomePageUrl());

    }

	  @Bean
	   public OperationFeignClient getOperationFeignClient(){
	       return create(OperationFeignClient.class,"basic-data");
	   }
}
```
**OperationFeignClient 类**
```java
package com.csbr.pharmacy.chain.cloud.service.operation;

import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

/**
 * @author kangxuan
 * @date 2018/12/28 0028 10:15.
 * @Description: 运营端专用 feign组件
 */
public interface OperationFeignClient {

    /**
     * 添加数据
     * @param tableName 表名
     * @param data 添加的实体类
     * @return
     */
    @RequestMapping(value = "/data/{tableName}/add",method = RequestMethod.POST)
    String  create(
            @PathVariable("tableName") String tableName,
            @RequestBody Object data);

}
```
**@FeignClient注解类**
```java
package com.csbr.pharmacy.chain.cloud.service;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

/**
 * @author kangxuan
 * @date 2018/12/28 0028 10:15.
 * @Description: 基础数据服 调用 公共组件
 */
@FeignClient(serviceId = "basic-data",name = "base")
public interface  BasicDataCommonFeignClient  {

    /**
     * 添加数据
     * @param tableName 表名
     * @param data 添加的实体类
     * @return
     */
    @RequestMapping(value = "/data/{tableName}/add",method = RequestMethod.POST)
    String  create(
            @PathVariable("tableName") String tableName,
            @RequestBody Object data);

    /**
     * 根据id查询数据
     * @param tableName 表名
     * @param guid guid
     * @return
     */
    @RequestMapping(value = "/data/{tableName}/{guid}",method = RequestMethod.GET)
    String selectByGuid(
            @PathVariable("tableName") String tableName,
            @PathVariable("guid") String guid);

    /**
     * 根据条件查询
     * @param tableName
     * @param data
     * @return
     */
    @RequestMapping(value = "/data/{tableName}/search",method = RequestMethod.POST)
    String selectByConditions(
            @PathVariable("tableName") String tableName,
            Object data);


    /**
     * 更新数据
     * @param tableName 表名
     * @param guid guid
     * @return
     */
    @RequestMapping(value = "/data/{tableName}/{guid}",method = RequestMethod.PUT)
    String update(
            @PathVariable("tableName") String tableName,
            @PathVariable("guid") String guid,
            @RequestBody Object data);
}
```
**这样两个 feign 接口都能进行使用了**
