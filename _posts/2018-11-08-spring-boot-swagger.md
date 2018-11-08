---
layout: post
published: true
title: Spring Boot Swagger
tags: spring swagger
---
Expose swagger for a simple Spring Boot Web app

## 1. Create a Spring Boot Swagger with Web dependency

```
$ spring init -d=web -groupId=io.github.joaovicente -artifactId=springbootswagger spring-boot-swagger
```

Step into the app folder

```
$ cd spring-boot-swagger
```

Create a ```SampleController``` class and expose a ```GET /sample``` endpoint

`./src/main/java/io/github/joaovicente/springbootswagger/SampleController.java`

```
package io.github.joaovicente.springbootswagger;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.bind.annotation.RequestMethod;

@RestController
public class SampleController {
    @RequestMapping(value="/sample", method=RequestMethod.GET)
    public String sample()   {
        return "Sample!\n";
    }
}
```

Run the app
```
$ mvn spring-boot:run
```

```
$curl http://localhost:8080/sample
Sample!
```

## 2. Configure application to expose Swagger

Add springfox dependencies to the `pom.xml`

```
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.6.1</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.6.1</version>
</dependency>
```

Add SwaggerConfig to the app in `./src/main/java/io/github/joaovicente/springbootswagger/SwaggerConfig.java`

```
package io.github.joaovicente.springbootswagger;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;
import springfox.documentation.swagger2.annotations.EnableSwagger2;

@Configuration
@EnableSwagger2
public class SwaggerConfig {
    @Bean
    public Docket api() {
        return new Docket(DocumentationType.SWAGGER_2)
                .select()
                .apis(RequestHandlerSelectors.basePackage("io.github.joaovicente.springbootswagger"))
                .build();
    }
}
```

## 3. Profit!

After running the app again (`mvn spring-boot:run`) you should be able to get Swagger JSON describing the API

```
$ curl http://localhost:8080/v2/api-docs
{"swagger":"2.0","info":{"description":"Api Documentation","version":"1.0","title":"Api Documentation","termsOfService":"urn:tos","contact":{},"license":{"name":"Apache 2.0","url":"http://www.apache.org/licenses/LICENSE-2.0"}},"host":"localhost:8080","basePath":"/","tags":[{"name":"sample-controller","description":"Sample Controller"}],"paths":{"/sample":{"get":{"tags":["sample-controller"],"summary":"sample","operationId":"sampleUsingGET","consumes":["application/json"],"produces":["*/*"],"responses":{"200":{"description":"OK","schema":{"type":"string"}},"401":{"description":"Unauthorized"},"403":{"description":"Forbidden"},"404":{"description":"Not Found"}}}}}}
```

And be able to interact with swagger-ui at [http://localhost:8080/swagger-ui.html] as shown below

![swagger-ui.png]({{site.baseurl}}/img/swagger-ui.png)

### References
For more comprehensive guides see 
* [Setting Up Swagger 2 with a Spring REST API](https://www.baeldung.com/swagger-2-documentation-for-spring-rest-api)
* [Spring Boot RESTful API Documentation With Swagger 2](https://dzone.com/articles/spring-boot-restful-api-documentation-with-swagger)
