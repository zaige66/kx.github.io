---
title: 通过aop 解决微服务中 跨服连表查询
date: 2019-01-18 16:35:10
tags: [java,spring,aop]
categories: [java,spring]
---


# 问题
```txt
在微服务架构中，我们需要对模块进行较细的拆分，但是对应到具体业务时，
又需要这些服务一起提供数据，这时可能就需要跨服务进行关联查询。
具体例子：
	把数据库层划分为：
					基础数据服
				    订单数据服
	现有一个具体业务：
					查询订单信息
	分析：
				订单主表信息是在订单数据服中进行查询
				订单主表中包含有商品信息，商品信息属于基础数据
			在以往的架构中：两个服务是在同一个数据库中，我们可以直接写sql语句，进行链接查询
			在微服务架构中：由于两张表归不同的服务管理，那么很有可能两张表不在同一个数据库中，我们无法进行连表查询
						 我们只能先查询出订单表信息，然后再拿出订单表中保存的商品主键，去基础数据服中查询商品信息。**本文解决的问题就是，简化这个过程**
```
# 预期的结果
```txt
当需要进行连表查询时，只需要配置下 **服务名称** **表名**，自动会去其他服进行查询，并把结果进行填充，不需要自己再写代码去处理。
```
# 实现过程
> 通过aop来实现这一功能

计划
```txt
1.在需要连表查询的 mapper 接口方法中加上 @NeedJoinQuery 自定义接口
2.通过 aop 对 @NeedJoinQuery 该注解标识的方法进行拦截
3.在接收查询结果的实体类中需要关联查询的属性上面 加上自定义注解 @JoinQuery，并配置上 服务名，表名
4.在切面实现中进行处理
```
# Aop切面实现类
```java
@Aspect
@Configuration
@EnableDiscoveryClient
@EnableAspectJAutoProxy(proxyTargetClass = true)
public class DifferentServerJoinQueryAspect {

    /**
     * 处理需要在服务间进行连表查询
     * @param joinPoint
     * @return
     */
    @Around(value = "@annotation(com.csbr.common.annotation.NeedJoinQuery)")
    public  Object joinQueryAfterAdvice(ProceedingJoinPoint joinPoint){
        Object returnValue = null;
        try {
            returnValue = joinPoint.proceed();
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }

        if (returnValue == null){
            return returnValue;
        }

        // 处理结果
        handlerJoinQuery(returnValue);

        return returnValue;
    }

    private <T extends ResponseModel> void handlerJoinQuery(Object returnValue){
        // 返回实体的 class对象
        Class<T> aClass ;
        List<T> listValue = new ArrayList<T>() ;
        if (returnValue instanceof List){
             listValue = (List<T>) returnValue;
            // 获取泛型的 class 对象
            if (listValue.size() == 0){
                return;
            }else {
                aClass = (Class<T>) listValue.get(0).getClass();
            }
        }else {
            aClass = (Class<T>)returnValue.getClass();
        }

        // 找出需要连表查询的属性
        Map<Field, JoinQuery> fieldMap = CommonUtil.getFieldsAnnotation(aClass,JoinQuery.class);
        // 对每个属性进行处理
        for (Map.Entry<Field, JoinQuery> stringAnnotationEntry : fieldMap.entrySet()) {
            Field field = stringAnnotationEntry.getKey();
            // 属性名
            String fieldName = field.getName();
            // 注解对象
            JoinQuery annotation =  stringAnnotationEntry.getValue();
            String serverId = annotation.serverId();
            String tableName = annotation.tableName();
            String guidName = annotation.guidName();

            /**
             * 根据serverId获取feign实例，
             * 1.如果是用@FeignClient注解生成的代理类，那么实例在容器中的名字为 serverId + “FeignClient”
             * 2.如果是自定义的Feign 代理类，那么这里还需要改进。
             *      如果在容器中读取不到，应该到一个Feign集合中去读取，如果还是读取不到，抛出异常
             *      Feign集合应该是开发人员自定义代理类时加入进去
             */
            CrudCommonFeignClient crudCommonFeignClient = (CrudCommonFeignClient)GlobalSpringUtil.getBeanByName(serverId + "FeignClient");
            if (null == crudCommonFeignClient){
                throw new RuntimeException("spring容器中无法获取到 "+serverId+" 服务的基础feign调用实例！");
            }

            // 返回集合
            if (returnValue instanceof List){
                /** guid集合 */
                List<String> guidList = new ArrayList<>();
                /** key：主键值，value：对应的对象 */
                Map<String,Object> guidAndObj = new HashMap<>();
                for (T obj : listValue) {
                    // 获取需要关联查询的对象
                    Object get = CommonUtil.optFieldValue(obj, fieldName, "get");
                    // 获取关联查询对象的主键值
                    String o = (String)CommonUtil.optFieldValue(get, guidName, "get");
                    guidList.add(o);
                    guidAndObj.put(o,get);
                }
                String s;
                try {
                    // 去其他服务进行数据请求
                    s = crudCommonFeignClient.selectByGuids(tableName, CommonUtil.concatBySeparator(guidList,","));
                }catch (Exception e){
                    e.printStackTrace();
                    // 发生错误时，进行下一个属性的查询
                    continue;
                }


                // 处理数据
                ResponseList<T> responseBean = JSON.parseObject(s, new TypeReference<ResponseList<T>>(field.getType()) {});
                if (responseBean.isSuccess()){
                    List<T> datas = responseBean.getDatas();
                    if (null != datas){
                        for (T data : datas) {
                            // 获取关联查询对象的主键值，这是获得的guid
                            String retGuid = (String)CommonUtil.optFieldValue(data, guidName, "get");
                            // 根据guid 获取对应的返回值中的对象
                            Object queryObj = guidAndObj.get(retGuid);
                            if (null != queryObj){
                                // 把服务调用获得的值 复制到 查询得到的对象中
                                BeanUtils.copyProperties(data,queryObj);
                            }
                        }
                    }
                }
            }

            // 返回对象
            else {
                // 获取需要关联查询的对象
                Object get = CommonUtil.optFieldValue(returnValue, fieldName, "get");
                // 获取关联查询对象的主键值
                Object o = CommonUtil.optFieldValue(get, guidName, "get");
                // 去其他服务进行数据请求
                String s = crudCommonFeignClient.selectByGuid(tableName, o.toString());

                // 处理获取的结果
                ResponseBean<T> responseBean = JSON.parseObject(s, new TypeReference<ResponseBean<T>>(field.getType()) {});
                T data = responseBean.getData();
                if (responseBean.isSuccess() && null != data){
                    // 把获取到的数据设置到返回值中
                    CommonUtil.optFieldValue(returnValue, fieldName, "set",data);
                }
            }

        }
    }

}
```
## Mapper文件
```java
@Mapper
@Repository
public interface TrprescriptionMapper {
    @NeedJoinQuery
    TrprescriptionResponse getTrprescriptionById(String id);
    @NeedJoinQuery
    List<TrprescriptionResponse> searchTrprescriptions(TrprescriptionSO so);
}
```
## 实体类
```java
package com.csbr.cloud.ordercenter.dto.response;

import com.csbr.common.annotation.JoinQuery;
import com.csbr.common.service.ResponseModel;
import io.swagger.annotations.ApiModel;
import io.swagger.annotations.ApiModelProperty;

import java.sql.Timestamp;
import java.util.List;

/**
 * @author kangxuan
 * @since 2019-01-16
 */
public class TrprescriptionResponse extends ResponseModel {

    private String guid;
  
    @JoinQuery(serverId = "basic-data",tableName = "Mfmed")
    private MfmedResponse med;

    private List<TrprescriptiondetailResponse> details;

    public MfmedResponse getMed() {
        return med;
    }

    public void setMed(MfmedResponse med) {
        this.med = med;
    }
}
```
## 自定义注解
```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface JoinQuery {

    /** 该属性需要访问的微服务名称 */
    String serverId();

    /** 表名 */
    String tableName();

    /** 主键名称，默认是 guid */
    String guidName() default "guid";
}
```

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface NeedJoinQuery {
}
```
## aop中用到的工具类CommonUtils
```java
 /**
     * 反射获操作对象中某属性的值
     * @param obj
     * @param fieldName
     * @return
     */
    public static  Object optFieldValue(Object obj,String fieldName,String methodPrefix,Object... args){
        Class aClass = obj.getClass();
        String methodName = methodPrefix + fieldName.substring(0,1).toUpperCase()+fieldName.substring(1,fieldName.length());
        try {

            if ("get".equals(methodPrefix)){
                Method method = aClass.getMethod(methodName);
                Object invoke = method.invoke(obj);
                return invoke;
            }else if ("set".equals(methodPrefix)){
                List<Class> classList = new ArrayList<>();
                for (Object arg : args) {
                    classList.add(arg.getClass());
                }
                Method method = aClass.getMethod(methodName,classList.toArray(new Class[]{}));
                method.invoke(obj,args);
            }

        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }

        return null;
    }

    /**
     * 获取 某字节码对象中，拥有某注解的属性集合
     * @param aClass class对象
     * @param annotationClass 注解class对象
     * @return map key:属性对象  value：注解对象
     */
    public static <T extends Annotation>  Map<Field,T> getFieldsAnnotation(Class aClass,Class<T> annotationClass ){
        Field[] declaredFields = aClass.getDeclaredFields();
        Map<Field, T> fieldMap = new HashMap<>();
        for (Field declaredField : declaredFields) {
            T annotation = declaredField.getAnnotation(annotationClass);
            if (annotation != null){
                fieldMap.put(declaredField,annotation);
            }
        }
        return fieldMap;
    }

    /**
     * 把list中的值根据分隔符进行拼接
     * @param strings
     * @param separator
     * @return
     */
    public static String concatBySeparator(List<String> strings,String separator){
        StringBuffer buffer = new StringBuffer();
        for (String string : strings) {
            buffer.append(string).append(separator);
        }
        String substring = buffer.substring(0, buffer.length() - 1);

        return substring;
    }

    /**
     * 把 字符串根据 分隔符 分割，返回list集合
     * @param content
     * @param separator 分隔符
     * @return
     */
    public static List<String> splitBySeparator(String content,String separator){
        String[] split = content.split(separator);
        return Arrays.asList(split);
    }
```
----------------------------------
## aop逻辑修改
> 分成两个步骤
> 	1.递归收集所有加了注解的字段，并进行汇总
>  2.对手收集到的结果进行处理
>  结果：可以自定义处理结果

```java
/**
 * @date 2019/1/18 0018 13:57.
 * @Description: 处理不同服务间的关联查询
 */
@Aspect
@Configuration
@EnableDiscoveryClient
@EnableAspectJAutoProxy(proxyTargetClass = true)
public class DifferentServerJoinQueryAspect2 {

    /**
     * 处理需要在服务间进行连表查询
     * @param joinPoint
     * @return
     */
    @Around(value = "@annotation(com.csbr.common.annotation.NeedJoinQuery)")
    public  Object joinQueryAfterAdvice(ProceedingJoinPoint joinPoint){
        Object returnValue = null;
        try {
            returnValue = joinPoint.proceed();
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }

        if (returnValue == null){
            return returnValue;
        }

        // 把注解信息收集起来,方便后面统一去请求获取数据，减少服务间请求次数
        Map<JoinQuery, Map<String, List<Object>>> annotationInfo = new HashMap<>();
        annotationInfoCollect(returnValue,annotationInfo);

        // 处理结果
        handlerJoinQuery(annotationInfo);

        return returnValue;
    }

    /**
    * feign 处理方式
    */
    private <T extends ResponseModel> void handlerJoinQuery(Map<JoinQuery, Map<String, List<Object>>> annotationInfo){
        if (annotationInfo.size() == 0){
            return;
        }
        for (Map.Entry<JoinQuery, Map<String, List<Object>>> joinQueryMapEntry : annotationInfo.entrySet()) {
            JoinQuery annotation = joinQueryMapEntry.getKey();
            Map<String, List<Object>> guidAndObjs = joinQueryMapEntry.getValue();
            if (guidAndObjs.size() == 0){
                return;
            }
            // 获取请求的一批guid
            Set<String> strings = guidAndObjs.keySet();
            String guids = CommonUtil.concatBySeparator(strings, ",");

            // 进行请求
            String serverId = annotation.serverId();
            String tableName = annotation.tableName();
            String guidName = annotation.guidName();

            CrudCommonFeignClient crudCommonFeignClient = (CrudCommonFeignClient)GlobalSpringUtil.getBeanByName(serverId + "FeignClient");
            if (null == crudCommonFeignClient){
                throw new RuntimeException("spring容器中无法获取到 "+serverId+" 服务的基础feign调用实例！");
            }

            String s;
            try {
                // 去其他服务进行数据请求
                s = crudCommonFeignClient.selectByGuids(tableName, guids);
            } catch (Exception e) {
                e.printStackTrace();
                // 发生错误时，进行下一个属性的查询
                continue;
            }

            // 处理返回值
            Collection<List<Object>> values1 = guidAndObjs.values();
            Object o = new Object();
            for (List<Object> list : values1) {
                o = list.get(0);
            }
            ResponseList<T> responseBean = JSON.parseObject(s, new TypeReference<ResponseList<T>>(o.getClass()) {
            });
            if (responseBean.isSuccess()) {
                List<T> datas = responseBean.getDatas();
                if (null != datas) {
                    for (T data : datas) {
                        // 获取关联查询对象的主键值，这是获得的guid
                        String retGuid = (String) CommonUtil.optFieldValue(data, guidName, "get");
                        // 根据guid 获取对应的返回值中的对象
                        List<Object> queryObj = guidAndObjs.get(retGuid);
                        if (null != queryObj) {
                            for (Object realObj : queryObj) {
                                // 把服务调用获得的值 复制到 查询得到的对象中
                                BeanUtils.copyProperties(data, realObj);
                            }
                        }
                    }
                }
            }
        }
    }

    /**
     * 收集注解信息，把需要对同一个服务同一张表的的guid汇总起来
     * @param returnValue
     * @param annotationInfo
     * @param <T>
     */
    private <T extends ResponseModel> void annotationInfoCollect(Object returnValue,Map<JoinQuery, Map<String, List<Object>>> annotationInfo){
        // 返回实体的 class对象
        Class<T> aClass ;
        if (returnValue instanceof List){
            List<T> listValue = (List<T>) returnValue;
            // 获取泛型的 class 对象
            if (listValue.size() == 0){
                return;
            }else {
                for (T t : listValue) {
                    // 递归调用
                    annotationInfoCollect(t,annotationInfo);
                }
            }
        }
        // 处理对象
        else {
            // ---------1.处理 添加了@JoinQuery的 单个属性--------
            aClass = (Class<T>)returnValue.getClass();
            // 找出需要连表查询的属性
            Map<Field, JoinQuery> fieldMap = CommonUtil.getFieldsAnnotation(aClass,JoinQuery.class);
            // 获取该属性实际的值，并取出 guid
            for (Map.Entry<Field, JoinQuery> stringAnnotationEntry : fieldMap.entrySet()) {
                Field field = stringAnnotationEntry.getKey();
                JoinQuery joinQuery = stringAnnotationEntry.getValue();

                // 找出实际对象
                String fieldName = field.getName();
                String guidName = joinQuery.guidName();
                Object get = CommonUtil.optFieldValue(returnValue, fieldName, "get");
                if (null == get){
                    continue;
                }
                // 获取guid，并把对应关系存入 annotaioninfo map中
                Object o = CommonUtil.optFieldValue(get, guidName, "get");
                if (null != o){
                    String guid = o.toString();
                    if (annotationInfo.containsKey(joinQuery)){
                        Map<String, List<Object>> stringListMap = annotationInfo.get(joinQuery);
                        if (stringListMap.containsKey(guid)){
                            stringListMap.get(guid).add(get);
                        }else {
                            List<Object> list = new ArrayList<>();
                            list.add(get);
                            stringListMap.put(guid,list);
                        }
                    }else {
                        Map<String, List<Object>> stringListMap = new HashMap<>();
                        List<Object> list = new ArrayList<>();
                        list.add(get);
                        stringListMap.put(guid,list);
                        annotationInfo.put(joinQuery,stringListMap);
                    }
                }
            }

            // ---------2.处理 对象里面的list属性--------
            Field[] declaredFields = aClass.getDeclaredFields();
            for (Field declaredField : declaredFields) {
                if (declaredField.getType().equals(List.class)){
                    String fieldName = declaredField.getName();
                    Object get = CommonUtil.optFieldValue(returnValue, fieldName, "get");
                    if (null != get && ((List)get).size() > 0){
                      annotationInfoCollect(get,annotationInfo);
                    }
                }
            }
        }
    }
}

```
# 写在后面
代码已经过调试，逻辑没问题。但是各位看官估计是无法复制过去直接用，因为这是我结合现有项目写的，所以要用的话得自己改改。
