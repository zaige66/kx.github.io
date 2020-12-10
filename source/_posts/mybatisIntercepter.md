---
title: 通过mybatis拦截器修改sql内容，实现不同的内容存入相同的字段中
date: 2019-12-19 17:28:46
tags: [java,mybatis]
categories: [java,mybaits]
---

@[toc]
# 场景
> 系统需要记录日志然后保存到数据库，保存的字段需要从请求信息中提取字段字段，不同的请求需要提取的字段不一样
# 实现
> 请求a  的请求信息 {"aaa":"aaa","bbbb":"bbbb"}，需要提取的字段为 aaa，代码实体中属性名也为aaa，
> 存入扩展字段一中，扩展字段的名为 PARAM1，则代码中需要将aaa与PARAM1对应起来
> 正常逻辑可以这么做：
>> 1.插入数据的时候可以 insert into log(PARAM1) values (#{aaa})
>> 2.查询的时候 select * from log where PARAM1 like "%XXX%" ,然后编写 resultMap 进行字段映射
这样的弊端：当有多个字段时，对应起来会让人奔溃，多个扩展字段与抽取字段对应操作繁琐，很容易出错

> 解决办法：
>>1.在属性 aaa 上加入自定义注解 ，注解中标明扩展字段名为 PARAM1
>> 2.mybatis拦截器中对sql语句中aaa内容替换成注解上注明的PARAM1
> 效果：
>> 1.插入数据的时候可以 insert into log(aaa) values (#{aaa})
>> 2.查询的时候 select * from log where aaa like "%XXX%" , 然后编写 resultMap 进行字段映射
>需要注意的是：查询时sql语句不需要写扩展字段名，但是 要写 resultMap 进行字段映射
## 注解
```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface ExtendColumnName {
    String value() ;
}
```
## 数据库定义
```sql
CREATE TABLE "RJGF_KMS"."KMS_REQUEST_LOG" 
   (	"ID" NUMBER, 
	"REQUEST_URL" VARCHAR2(100), 
	"REQUEST_DATA" VARCHAR2(1000), 
	"IP" VARCHAR2(20), 
	"RESPONSE_DATA" VARCHAR2(1000), 
	"TYPE_CODE" VARCHAR2(50), 
	"RECORD_TIME" TIMESTAMP (6), 
	"PARAM1" VARCHAR2(255), 
	"PARAM2" VARCHAR2(255), 
	"PARAM3" VARCHAR2(255), 
	"PARAM4" VARCHAR2(255), 
	"PARAM5" VARCHAR2(255)
   ) SEGMENT CREATION IMMEDIATE 
  PCTFREE 10 PCTUSED 40 INITRANS 1 MAXTRANS 255 NOCOMPRESS LOGGING
  STORAGE(INITIAL 65536 NEXT 8192 MINEXTENTS 1 MAXEXTENTS 2147483645
  PCTINCREASE 0 FREELISTS 1 FREELIST GROUPS 1 BUFFER_POOL DEFAULT FLASH_CACHE DEFAULT CELL_FLASH_CACHE DEFAULT)
```
## 实体
```java
public class PsamRequestLogEntity extends KmsRequestLog  {

    @ExtendColumnName("PARAM1")
    private String psamNo;

    @ExtendColumnName("PARAM2")
    private String code;

    @ExtendColumnName("PARAM3")
    private String lanNo;
}
```
## 拦截器
```java
@Component
@Intercepts(
        {
                @Signature(type = Executor.class, method = "query",
                        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class, CacheKey.class, BoundSql.class}),
                @Signature(type = Executor.class, method = "query",
                        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}),
                @Signature(type = Executor.class, method = "update",
                        args = {MappedStatement.class, Object.class})
        }
)
public class RequestLogExtendColumnInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        // 获取sql
        String sql = getSqlByInvocation(invocation);
        if (StringUtils.isBlank(sql)) {
            return invocation.proceed();
        }

        // 找出需要替换的字段
        Map<String, ExtendColumnName> extendColumnNames = getExtendColumn(invocation);

        if (extendColumnNames.size() == 0){
            return invocation.proceed();
        }

        // sql交由处理类处理  对sql语句进行处理
        String newSql = getExtendSql(extendColumnNames,sql);

        // 包装sql后，重置到invocation中
        resetSql2Invocation(invocation, newSql);

        return invocation.proceed();
    }

    private String getExtendSql(Map<String, ExtendColumnName> extendColumnNames, String sql) {
        String retVal = sql;
        for (Map.Entry<String, ExtendColumnName> stringExtendColumnNameEntry : extendColumnNames.entrySet()) {
            String key = stringExtendColumnNameEntry.getKey();
            ExtendColumnName value = stringExtendColumnNameEntry.getValue();
            // 将sql中的属性名 替换成 注解上的名字
            retVal = retVal.replace(key,value.value());
        }
        return retVal;
    }

    /**
     * 获取需要替换字段名的属性
     * @param invocation
     * @return
     */
    private Map<String, ExtendColumnName> getExtendColumn(Invocation invocation) {
        Map<String, ExtendColumnName> retVal = new HashMap<>();

        Object arg = invocation.getArgs()[1];
        // 当接口的方法有多个参数时，arg[1] 的对象为 MapperMethod.ParamMap，只有一个参数时，arg[1] 就是参数本身
        if (arg instanceof MapperMethod.ParamMap){
            MapperMethod.ParamMap paramMap = (MapperMethod.ParamMap) arg;
            Collection values = paramMap.values();
            for (Object value : values) {
                if (value != null) {
                    getExtendColumn(retVal, value);
                }
            }
        } else {
            getExtendColumn(retVal,arg);
        }

        return retVal;
    }

    private void getExtendColumn(Map<String, ExtendColumnName> retVal, Object arg) {
        Field[] declaredFields = getAllFields(arg);
        for (Field declaredField : declaredFields) {
            ExtendColumnName annotation = declaredField.getAnnotation(ExtendColumnName.class);
            if (null != annotation){
                // 如果属性上有 ExtendColumnName 该注解，就放入返回值中
                retVal.put(declaredField.getName(),annotation);
            }
        }
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target,this);
    }

    @Override
    public void setProperties(Properties properties) {

    }

    /**
     * 获取sql语句
     * @param invocation
     * @return
     */
    private String getSqlByInvocation(Invocation invocation) {
        final Object[] args = invocation.getArgs();
        MappedStatement ms = (MappedStatement) args[0];
        Object parameterObject = args[1];
        BoundSql boundSql = ms.getBoundSql(parameterObject);
        return boundSql.getSql();
    }

    /**
     * 包装sql后，重置到invocation中
     * @param invocation
     * @param sql
     * @throws SQLException
     */
    private void resetSql2Invocation(Invocation invocation, String sql) throws SQLException {
        final Object[] args = invocation.getArgs();
        MappedStatement statement = (MappedStatement) args[0];
        Object parameterObject = args[1];
        BoundSql boundSql = statement.getBoundSql(parameterObject);
        MappedStatement newStatement = newMappedStatement(statement, new BoundSqlSqlSource(boundSql));
        MetaObject msObject =  MetaObject.forObject(newStatement, new DefaultObjectFactory(), new DefaultObjectWrapperFactory(),new DefaultReflectorFactory());
        msObject.setValue("sqlSource.boundSql.sql", sql);
        args[0] = newStatement;

        // 如果参数个数为6，还需要处理 BoundSql 对象
        if (6 == args.length){
            BoundSql boundSqlArg = (BoundSql) args[5];
            // 该对象没有提供对sql属性的set方法，只能通过反射进行修改
            Class<? extends BoundSql> aClass = boundSql.getClass();
            try {
                Field field = aClass.getDeclaredField("sql");
                field.setAccessible(true);
                field.set(boundSqlArg,sql);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

    }

    private MappedStatement newMappedStatement(MappedStatement ms, SqlSource newSqlSource) {
        MappedStatement.Builder builder =
                new MappedStatement.Builder(ms.getConfiguration(), ms.getId(), newSqlSource, ms.getSqlCommandType());
        builder.resource(ms.getResource());
        builder.fetchSize(ms.getFetchSize());
        builder.statementType(ms.getStatementType());
        builder.keyGenerator(ms.getKeyGenerator());
        if (ms.getKeyProperties() != null && ms.getKeyProperties().length != 0) {
            StringBuilder keyProperties = new StringBuilder();
            for (String keyProperty : ms.getKeyProperties()) {
                keyProperties.append(keyProperty).append(",");
            }
            keyProperties.delete(keyProperties.length() - 1, keyProperties.length());
            builder.keyProperty(keyProperties.toString());
        }
        builder.timeout(ms.getTimeout());
        builder.parameterMap(ms.getParameterMap());
        builder.resultMaps(ms.getResultMaps());
        builder.resultSetType(ms.getResultSetType());
        builder.cache(ms.getCache());
        builder.flushCacheRequired(ms.isFlushCacheRequired());
        builder.useCache(ms.isUseCache());

        return builder.build();
    }

    public static Field[] getAllFields(Object object){
        Class clazz = object.getClass();
        List<Field> fieldList = new ArrayList<>();
        while (clazz != null){
            fieldList.addAll(new ArrayList<>(Arrays.asList(clazz.getDeclaredFields())));
            clazz = clazz.getSuperclass();
        }
        Field[] fields = new Field[fieldList.size()];
        fieldList.toArray(fields);
        return fields;
    }


}

//    定义一个内部辅助类，作用是包装sq
class BoundSqlSqlSource implements SqlSource {
    private BoundSql boundSql;
    public BoundSqlSqlSource(BoundSql boundSql) {
        this.boundSql = boundSql;
    }
    @Override
    public BoundSql getBoundSql(Object parameterObject) {
        return boundSql;
    }
}
```
## xml
```xml
<resultMap id="PsamRequestLog" type="net.rjgf.bg.modules.sys.entity.bean.PsamRequestLogEntity">
        <id column="id" property="id"></id>
        <result column="record_time" property="recordTime"></result>
        <result column="ip" property="ip"></result>
        <result column="request_data" property="requestData"></result>
        <result column="response_data" property="responseData"></result>
        <result column="request_url" property="requestUrl"></result>
        <result column="type_code" property="typeCode"></result>
        <result column="PARAM1" property="psamNo"></result>
        <result column="PARAM2" property="code"></result>
        <result column="PARAM3" property="lanNo"></result>
    </resultMap>

    <!--查询-->
    <select id="selectPSAMLogByConditions" parameterType="net.rjgf.bg.modules.sys.entity.so.PsamRequestLogSo" resultMap="PsamRequestLog">
        select * from KMS_REQUEST_LOG
        <where>
            <if test="so.ip != null and so.ip != ''">
                and ip like concat('%',concat(#{so.ip},'%'))
            </if>
            <if test="so.typeCode != null and so.typeCode != ''">
                and type_code like concat('%',concat(#{so.typeCode},'%'))
            </if>
            <if test="so.psamNo != null and so.psamNo != ''">
                and psamNo like concat('%',concat(#{so.psamNo},'%'))
            </if>
            <if test="so.code != null and so.code != ''">
                and code like concat('%',concat(#{so.code},'%'))
            </if>
            <if test="so.lanNo != null and so.lanNo != ''">
                and lanNo like concat('%',concat(#{so.lanNo},'%'))
            </if>
            <if test="so.startTime != null and so.startTime != ''">
                and record_time >= to_date(#{so.startTime},'yyyy-mm-dd')
            </if>
            <if test="so.endTime != null and so.endTime != ''">
                and to_date(#{so.endTime},'yyyy-mm-dd') >= record_time
            </if>
            <if test="so.requestUrl != null and so.requestUrl != ''">
                and request_url like concat('%',concat(#{so.requestUrl},'%'))
            </if>
        </where>
    </select>
    
<!--新增Psam请求日志-->
    <insert id="insertPsamLog" parameterType="net.rjgf.kms.forward.entity.bean.PsamRequestLogEntity">
        INSERT INTO KMS_REQUEST_LOG(ID,REQUEST_URL, REQUEST_DATA, IP, RESPONSE_DATA, TYPE_CODE, RECORD_TIME, psamNo,lanNo,code)
        VALUES ( KMS_REQUEST_LOG_SEQ.NEXTVAL,#{requestUrl,jdbcType=VARCHAR}, #{requestData, jdbcType=VARCHAR}, #{ip, jdbcType=VARCHAR}, #{responseData, jdbcType=VARCHAR}, #{typeCode, jdbcType=VARCHAR}, #{recordTime, jdbcType=VARCHAR}, #{psamNo, jdbcType=VARCHAR}, #{lanNo, jdbcType=VARCHAR}, #{code, jdbcType=VARCHAR})
    </insert>
```
# 效果
这样就不需要管数据库「扩展字段名」与代码实体类中「属性名」的对应关系了

## 拦截器中需要注意的地方
sql修改后要更新两个对象 「MappedStatement」与「BoundSql」都修改下最好，因为源码中有时候用到了BoundSql对象来获取sql语句
