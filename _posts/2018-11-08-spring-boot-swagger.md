---
layout: post
published: false
title: Spring Boot Swagger
---
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

## 2. Add dependencies required to expose Swagger

