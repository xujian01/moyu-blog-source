---
title: 基于Apollo的配置自动刷新组件
date: 2020-06-22 10:33:29
tags: Apollo,SpringBoot
---
> 本文介绍在[《Apollo配置中心二：环境隔离方案》](https://note.youdao.com/)的基础上开发基于注解自动添加配置更新监听器的组件，也是对[《Apollo配置中心二：环境隔离方案》](https://note.youdao.com/)第6章的补充。

## 1. 定义注解
定义需要使用的注解，其中一个注解需要起到标注作用，对于@ConfigurationProperties定义的配置bean加了该注解会自动为其添加listener；
另一个注解起到开关作用，类似@EnableFeignClients，整体控制上一个注解是否生效

注解1：标注注解
```java
/**
 * @author xujian03
 * @date 2020/4/15 10:44
 * @description 开启自动添加listener，用于自动刷新@ConfigurationProperties配置，作用于启动类 自动刷新@ConfigurationProperties的配置
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface AutoRefreshApollo {
    /**
     * apollo namespace的前缀，如application.yml的application部分，如果不设置则自动寻找项目的全局namespace
     * @return
     */
    String value();

    /**
     * apollo namespace的前缀，如application.yml的.yml部分，如果不设置则自动寻找项目的全局namespace
     * @return
     */
    String namespaceSuffix() default "yml";
}
```
注解2：开关注解
```java
/**
 * @author xujian03
 * @date 2020/4/16 11:05
 * @description 开启自动添加listener，用于自动刷新@ConfigurationProperties配置，作用于启动类
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
//这里导入注解处理类的注册类
@Import(AutoRefreshApolloRegistrar.class)
public @interface EnableAutoRefreshApollo {
}
```
## 2. 注解处理类
处理@AutoRefreshApollo注解
```java
/**
 * @author xujian03
 * @date 2020/4/9 15:28
 * @description 基于AutoRefreshApolloProperties和ConfigurationProperties注解的自动配置刷新
 */
@Slf4j
public class ConfigurationPropertiesListenerRegistry implements ApplicationEventPublisherAware, ApplicationContextAware {
    private ApplicationEventPublisher publisher;
    private ApplicationContext applicationContext;
    //指定的环境
    @Value("${spring.profiles.active:}")
    private String env;

    @PostConstruct
    public void init() {
        //获取带有@AutoRefreshApollo注解的bean
        Map<String,Object> beanMap = applicationContext.getBeansWithAnnotation(AutoRefreshApollo.class);
        for (Object bean : beanMap.values()) {
            Set<String> prefixSet = new HashSet<>();
            Class clazz = bean.getClass();
            //获取该类上的@ConfigurationProperties注解
            ConfigurationProperties configurationProperties = (ConfigurationProperties) clazz.getAnnotation(ConfigurationProperties.class);
            if (configurationProperties == null) continue;
            String prefix = configurationProperties.value();
            if (StringUtils.isEmpty(prefix)) prefix = configurationProperties.prefix();
            if (!StringUtils.isEmpty(prefix)) prefix+=".";
            prefixSet.add(prefix);
            //获取该类上的@AutoRefreshApollo注解
            AutoRefreshApollo clazzAnnotation = (AutoRefreshApollo) clazz.getAnnotation(AutoRefreshApollo.class);
            String namespacePrefix = clazzAnnotation.value();
            if (!StringUtils.isEmpty(env)) namespacePrefix+="-";

            String namespaceSuffix = clazzAnnotation.namespaceSuffix();
            Config config = ConfigService.getConfig(namespacePrefix+env+"."+namespaceSuffix);
            //该方法有三个参数，第一个参数是具体的listener，第二个参数是感兴趣的key，第三个参数是感兴趣的key的前缀
            log.info("添加Apollo配置监听器-----------START");
            config.addChangeListener(changeEvent -> {
                log.info("刷新配置-----------START");
                //重新绑定配置
                publisher.publishEvent(new EnvironmentChangeEvent(changeEvent.changedKeys()));
                //刷新配置bean
                publisher.publishEvent(new RefreshRoutesEvent(bean));
                log.info("刷新配置-----------END");
            },null,prefixSet);
            log.info("添加Apollo配置监听器-----------END");
        }
    }

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        this.publisher = applicationEventPublisher;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
```
注册@AutoRefreshApollo注解处理器bean
```java
/**
 * @author xujian03
 * @date 2020/4/16 10:36
 * @description 注册注解处理器bean
 */
public class AutoRefreshApolloRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        registerBeanDefinitionIfNotExists(registry, ConfigurationPropertiesListenerRegistry.class.getName(),ConfigurationPropertiesListenerRegistry.class);
    }
    private boolean registerBeanDefinitionIfNotExists(BeanDefinitionRegistry registry, String beanName,
                                                      Class<?> beanClass) {
        if (registry.containsBeanDefinition(beanName)) return false;

        String[] candidates = registry.getBeanDefinitionNames();
        for (String candidate : candidates) {
            BeanDefinition beanDefinition = registry.getBeanDefinition(candidate);
            if (Objects.equals(beanDefinition.getBeanClassName(), beanClass.getName())) {
                return false;
            }
        }

        BeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition(beanClass).getBeanDefinition();
        registry.registerBeanDefinition(beanName, beanDefinition);

        return true;
    }
}
```
## 3. 使用示例
1. 定义一个基于@ConfigurationProperties的配置类：
```java
/**
 * @author xujian03
 * @date 2020/4/16 14:42
 * @description
 */
@Component
@ConfigurationProperties("myuser")
@AutoRefreshApollo("tiku-gateway")
@Data
public class UserInfo {
    private String name;
    private String nick;
}
```
2. 在启动类上增加@EnableAutoRefreshApollo注解：
```java
@SpringBootApplication
//启用基于注解自动刷新apollo配置，使@AutoRefreshApollo注解生效
@EnableAutoRefreshGatewayApollo
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}

```
3. 在apollo配置中心修改
```yaml
myuser:
    name: xujian
    nick: jianjian
```
可以在代码里调试到UserInfo对应的bean的属性值已经改变。