---
title: Swagger API Spec + Swagger Codegen + YAPI管理接口文档
date: 2020-11-15 10:33:29
tags: Swagger
---
> 背景：使用SpringCloud进行微服务开发，且后端服务调用大都使用Feign Client进行调用。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210425111232757.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)

## 1、项目接入Swagger

```xml
<dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
            <version>2.9.2</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>2.9.2</version>
        </dependency>
```

## 2、项目引入maven插件

```xml
<plugin>
                <groupId>io.swagger</groupId>
                <artifactId>swagger-codegen-maven-plugin</artifactId>
                <version>3.1.2</version>
                <configuration>
                    <!-- springcloud固定配置-start -->
                    <language>spring</language>
                    <library>spring-cloud</library>
                    <generateApis>true</generateApis>
                    <generateModels>true</generateModels>
                    <generateSupportingFiles>false</generateSupportingFiles>
                    <!-- springcloud固定配置-end -->
                    <additionalProperties>
                        <additionalPropertie>java8=true</additionalPropertie>
                        <additionalPropertie>generateForOpenFeign=true</additionalPropertie>
                        <!-- feign服务名称 -->
                        <additionalPropertie>title=myproject</additionalPropertie>
                    </additionalProperties>
                    <!-- swagger 接口定义文件位置 -->
                    <inputSpec>/Users/jarry/Downloads/swagger.yaml</inputSpec>
                    <!-- 生成的代码输出位置 -->
                    <output>${project.artifactId}</output>
                    <!-- api包路径 -->
                    <apiPackage>com.jarry.myproject.api</apiPackage>
                    <!-- model包路径 -->
                    <modelPackage>com.jarry.myproject.dto</modelPackage>
                </configuration>
            </plugin>
```
## 3、编写Swagger API Spec（接口定义文档）
参考：

```yaml
swagger: '2.0'
info:
  title: myproject
  version: last
basePath: /feign
tags:
  - name: feign接口
schemes:
  - http
paths:
  /feign/user/foo:
    get:
      tags:
        - feign接口
      operationId: foo
      summary: ''
      description:
        0：成功
      parameters:
        - name: param
          in: query
          required: true
          description: 参数
          type: string
      responses:
        '200':
          description: successful operation
  /feign/user/bar:
    post:
      tags:
        - feign接口
      operationId: bar
      summary: ''
      description: ''
      consumes:
        - application/json
      parameters:
        - name: paramDTO
          in: body
          schema:
            type: object
            properties:
              param1:
                type: string
                description: ''
              param2:
                type: string
                description: ''
            required:
              - param1
              - param2
      responses:
        '200':
          description: successful operation
          schema:
            type: string
```

**详细的编写规范参考：[https://github.com/OAI/OpenAPI-Specification/blob/main/versions/2.0.md](https://github.com/OAI/OpenAPI-Specification/blob/main/versions/2.0.md)**

## 4、代码生成
### 生成方式
方式一：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210425112140598.png)


在IDEA插件列表双击执行

方式二：

通过命令

`mvn io.swagger:swagger-codegen-maven-plugin:generate`

执行

### 生成的目录结构
`{output}/src/main/java/{apiPackage}和{output}/src/main/java/{modelPackage}`

其中output是上面maven插件配置的输出目录，apiPackage是上面maven插件配置的api包路径，modelPackage是上面maven插件配置的model包路径

## 5、项目中加入Swagger配置文件
参考：

```java
/**
 * @author xujian
 * 2021-04-21 17:41
 **/
@Configuration
@EnableSwagger2
public class SwaggerConfig {
    @Value("${swagger-ui.enable:false}")
    private Boolean enableSwaggerUI;
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .enable(enableSwaggerUI)
                .apiInfo(new ApiInfoBuilder()
                        .title("myproject")
                        .description("myproject接口文档")
                        .version("1.0.0")
                        .contact(new Contact("墨鱼","","moyu@bjarry.com"))
                        .license("The Jarry License")
                        .licenseUrl("http://www.jarry.com")
                        .build())
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.jarry.myproject.controller"))
                .paths(PathSelectors.any())
                .build();
    }
}
```

6、访问Swagger UI
启动项目，访问http://ip:port/swagger-ui.html

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210425112224744.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)



复制图中红框里的链接

7、接口导入到YAPI

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210425112029317.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)


**注意需要将localhost替换为真实ip**