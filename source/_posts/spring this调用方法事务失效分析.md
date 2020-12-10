---
title: "spring this调用方法事务失效分析"
date: 2018-08-14 21:56:13  
tags: [java,spring,aop]
categories: [java,spring]
---

问题

       a方法，b方法，都通过aop加上了事物控制，a中调用了b方法，那么一共几次事物

准备

      1.创建数据库

-- 创建数据库
use test;
-- 建表
create table account(
  id int not null auto_increment,
  name varchar(20) not null ,
  money double not null
);
INSERT into account(id,name,money) VALUE
  (1,"aaa",1000),
  (1,"bbb",1000),
  (1,"ccc",1000)

      2.xml配置，实例化了两个 service 类，一个dao类，开启事务注解驱动

<!--spring事务管理器-->
    <bean name="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"></property>
    </bean>

    <tx:annotation-driven transaction-manager="transactionManager"></tx:annotation-driven>


    <!--配置业务层-->
    <bean id="accountDaoImpl" class="com.test.demo4.AccountDaoImpl">
        <property name="dataSource" ref="dataSource"></property>
    </bean>
    <bean id="accountServiceImpl" class="com.test.demo4.AccountServiceImpl"></bean>
    <bean id="accountServiceImpl2" class="com.test.demo4.AccountServiceImpl2"></bean>
      3.dao 类

public class AccountDaoImpl extends JdbcDaoSupport implements AccountDao {

    @Override
    public void add(String name, double money) {
        String sql = "update `account` set money = money + "+money+" where `name` = '"+name+"'";
        getJdbcTemplate().execute(sql);
    }

    @Override
    public void reduce(String name, double money) {
        String sql = "update `account` set money = money - "+money+" where `name` =  '"+name+"'";
        getJdbcTemplate().execute(sql);
    }
}
     4.service 类

// service 1
public class AccountServiceImpl  implements AccountService {
    @Autowired
    private AccountDao accountDao;
    @Autowired
    private AccountServiceImpl2 accountServiceImpl2;

    @Override
    public void transfer(String from, String to, double money) {
        add(to,money);
        try {
            //accountServiceImpl2.reduce(from,money);
            reduce(from,money);
        }catch (Exception e){
            System.out.println("1111111");
        }

    }

    public void  add(String to, double money){
        accountDao.add(to,money);
    }

    public void  reduce(String to, double money){
       int i = 1/0;
        accountDao.reduce(to,money);
    }
}



// service 2
public class AccountServiceImpl2  {
    @Autowired
    private AccountDao accountDao;

   @Transactional(propagation = Propagation.REQUIRED, readOnly = false)
    public void  reduce(String to, double money){
       int i = 1/0;
        accountDao.reduce(to,money);
    }
}
    5.测试类

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration("classpath:applicationContext4.xml")
public class TestService {
    @Autowired
    private AccountService accountService;

    @Test
    public void test(){
        accountService.transfer("aaa","bbb",200);
    }
}
测试

      1.当 a , b 方法在同一个类中时，两个方法的 事务传播行为都设置为 Propagation.REQUIRED

          即 AccountServiceImpl 中transfer 的内容为

@Transactional(propagation = Propagation.REQUIRED, readOnly = false)
    @Override
    public void transfer(String from, String to, double money) {
        add(to,money);
        try {
            //accountServiceImpl2.reduce(from,money);
            reduce(from,money);
        }catch (Exception e){
            System.out.println("1111111");
        }

    }

         AccountServiceImpl 中reduce 的内容为

@Transactional(propagation = Propagation.REQUIRED, readOnly = false)
 public void  reduce(String to, double money){
    int i = 1/0;
     accountDao.reduce(to,money);
 }

测试结果：



当设置 AccountServiceImpl 中transfer 的传播行为是 Propagation.REQUIRED，AccountServiceImpl 中reduce 的传播行为是 REQUIRES_NEW 时，

测试结果：



2.当 a , b 方法不在同一个类中时，两个方法的 事务传播行为都设置为 Propagation.REQUIRED

即 AccountServiceImpl 中transfer 的内容为

@Transactional(propagation = Propagation.REQUIRED, readOnly = false)
    @Override
    public void transfer(String from, String to, double money) {
        add(to,money);
        try {
            accountServiceImpl2.reduce(from,money);
            //reduce(from,money);
        }catch (Exception e){
            System.out.println("1111111");
        }

    }
AccountServiceImpl2 中reduce 的内容为

@Transactional(propagation = Propagation.REQUIRED, readOnly = false)
 public void  reduce(String to, double money){
    int i = 1/0;
     accountDao.reduce(to,money);
 }

测试结果：，控制台抛出异常  org.springframework.transaction.UnexpectedRollbackException: Transaction rolled back because it has been marked as rollback-only


当设置 AccountServiceImpl 中transfer 的传播行为是 Propagation.REQUIRED，AccountServiceImpl2 中reduce 的传播行为是 Propagation.REQUIRES_NEW 时，

测试结果：，控制台正常



总结：

        1.当a,b方法在同一个类中时，不管 b 方法设置何种 传播行为，a,b方法都共用一个事务（具体实现方式还未深究）。

        2.当a,b方法不在同一个类中时，事务的传播行为看 b 方法的注解配置，b 方法配置 Propagation.REQUIRED 是，与a方法共用事务，

当 b 中产生异常，b 方法把 事务 标记 为回滚，然后把异常抛给 a 方法，a 方法对异常进行了捕获，所以 a 方法内无异常，a方法会 对事务 进行提交操作，但是b 方法已经把 该事务 标记为 回滚，所以 会抛出异常。解决这个的办法就是把 b 方法配置为 Propagation.REQUIRES_NEW，这样 b 方法会产生新的 事务，结果就是 b 的事务回滚，a 的事务正常提交



如有说的不对地方，欢迎大家拍砖！

------------------------------------------------------------------------------------------

上面总结中的第一点，经过一番百度，有一篇比较深入的讲解博客，附上连接：https://blog.csdn.net/jiesa/article/details/53438342。

但是对于该博客的分析中，我还是有点疑问 ：

博文中说 当调用本类方法时，此时的 this 是指向被代理的对象的



测试确实如此，那为什么不是 指向 spring 生成的代理类呢？

希望知道的朋友能知导一二，谢谢！

----------------------------------------------------------------------------------------------------------------------------------------

又找到了两个说的很透彻的链接了！

http://www.importnew.com/28793.html

https://my.oschina.net/guangshan/blog/1797461

JDK 动态代理

JdkDynamicAopProxy 类的invoke方法
Cglib动态代理

CglibAopProxy 类中设置了callback   为 DynamicAdvisedInterceptor 类


我们配置的 aop 切面，实际是以责任链的方式执行的，然后调用的*被代理的对象*执行响应的方法，所以在方法中断点查看this对象，发现不是代理对象，就是当前的类。








