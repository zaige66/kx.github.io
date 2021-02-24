---
title: 通过spring Registrar扩展点，实现同一个接口根据配置文件来加载不同的实现类
date:  2021-02-24 17:17:02
tags: [java,spring]
categories: [java,spring]
---

@[toc]
# 写在前面
```
现有场景：
	模块业务有多种实现情况，需要有多个实现类，并且多个实现类需要根据配置文件来进行切换
```
# 排除已有的类的加载
- 通过@ComponentScan注解来排除扫描的包
- 注意排除规则要加上 「.*」
```java
@SpringBootApplication
@ComponentScan(basePackages = {"net.rjgf"},
        excludeFilters = {
        @ComponentScan.Filter(type = FilterType.REGEX,pattern = "net.rjgf.starter.test.*")
})
```
# 设置配置文件
- 在application-dev.yml文件中加入配置文件
```yml
packageName: net.rjgf.starter.test.one
```
- 在application-prod.yml文件中加入配置文件
```yml
packageName: net.rjgf.starter.test.two
```
# 编写Registrar扩展点类进行自定义加载
```java
public class Test implements ImportBeanDefinitionRegistrar, EnvironmentAware {
    
    private Environment environment;

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

        // 获取配置文件的值
        String packageName = environment.getProperty("packageName");
        
        // 手动扫描包
        ClassPathBeanDefinitionScanner classPathBeanDefinitionScanner = new ClassPathBeanDefinitionScanner(registry);
        classPathBeanDefinitionScanner.scan(packageName);
    }

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }
}
```

# 激活扩展点类
- 在启动类上通过@Import注解引入
```java
@Import(Test.class)
@SpringBootApplication
@ComponentScan(basePackages = {"net.rjgf"},
        excludeFilters = {
        @ComponentScan.Filter(type = FilterType.REGEX,pattern = "net.rjgf.starter.test.*")
})
public class StarterApplication {
```
# 测试类
- 测试包名：net.rjgf.starter.test
- 包结构如下
```
net.rjgf.starter.test
	one
		OneClass.java
	two
		TwoClass.java
	TestInterFace.java
```
- OneClass.java
```java
@Component
public class OneClass implements TestInterFace {
    @Override
    public void say() {
        System.out.println("one");
    }
}
```
- TwoClass.java
```java
@Component
public class TwoClass implements TestInterFace {
    @Override
    public void say() {
        System.out.println("two");
    }
}
```
- TestInterFace.java
```java
public interface TestInterFace {
    void say();
}
```
- 启动类
```java
@Import(Test.class)
@SpringBootApplication
@ComponentScan(basePackages = {"net.rjgf"},
        excludeFilters = {
        @ComponentScan.Filter(type = FilterType.REGEX,pattern = "net.rjgf.starter.test.*")
})
public class StarterApplication {
	
	public static void main(String[] args) {
        SpringApplication app = new SpringApplication(StarterApplication.class);
        ConfigurableApplicationContext run = app.run(args);
		// 测试方法的调用
        run.getBean(TestInterFace.class).say();
    }
}
```
- 测试结果
```
当active.profiles=dev时，输出：one
当active.profiles=prod时，输出：two
```

# 写在后面
```
该方式适合对已有项目进行改造，这种方式只需要把现有的包复制一份出来，然后修改相应的逻辑即可，如果用spring提供的@Conditional注解来进行改造，需要在每个类上都加注解，改动比较大
当第一次开发时，如果有改类场景，可以考虑用spring提供的@Conditional进行自定义激活条件，或者采用spring提供的各类 @OnxxxxConditional 注解 来实现该效果
```
