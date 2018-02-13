---
layout: post
title: 'Event driven with Spring REST, Kafka and MongoDB'
published: false
---
The aim of this post is to illustrate how to build a simple Event Driven system using
* Spring REST
* MongoDB
* Kafka

In this post we'll pretend we are creating a Story editing web application, where authors can create their account and then create stories.

Well, it may take a while until we get to create stories, as the focus really is to get hands on experience in building an Event Driven system using the building blocks described above, using Spring Boot, Docker, and later on explore how we can build observabiliy into the system

I'll throw in a few tooling goodies along the way, which should make life easier. 

Enough chat, time for sleeve rolling

## Going REST...full....ish
Let's start by creating a Spring Boot project

Let's start by building the `Author` service. We'll depend on web for REST capabilities and `lombok` to auto-generate getters and setters for the REST DTOs. We're also going to throw in `data-mongodb` and `kafka` dependencies as we are going to use them later on.

[Spring Boot CLI](https://docs.spring.io/spring-boot/docs/current/reference/html/getting-started-installing-spring-boot.html#getting-started-installing-the-cli) provides a very easy way to create Spring Boot applications. After you have installed it, type this command:

```bash
$ spring init \
    -d=web,lombok,data-mongodb,kafka \
    -groupId=io.github.joaovicente \
    -artifactId=stories \
    -name=stories \
    stories
```

Let's build it  

```bash
$ cd stories
$ mvn clean package
```

and run it
```bash
$ mvn spring-boot:run
```

So the application runs but is does not do anything useful, so lets stop the app now and let's create a`CreatAuthorController`by editing`./src/main/java/com/joaovicente/CreateAuthorController.java`and add  POST capabilities

The handler method as shown below, to expose`POST /authors`

```java
package com.joaovicente.stories;

import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMethod;

@RestController
public class CreateAuthorController {
    @RequestMapping(value = "/authors", method = RequestMethod.POST)

    public CreateAuthorDto createAuthor(@RequestBody CreateAuthorDto createAuthorDto) {
        return createAuthorDto;
    }
}
```

which will use the`/src/main/java/com/joaovicente/CreateAuthorDto.java` shown below

```java
package com.joaovicente.stories;

import lombok.Data;

@Data
public class CreateAuthorDto {
    private String name;
    private String email;
}
```

Let's run the app

```bash
mvn spring-boot:run
```

and try out the `GET` endpoint

```bash
curl http://localhost:8080/authors/123
```

I am going to use [httpie](https://httpie.org/) instead of curl to interact with the REST interface

So here goes a POST /authors

```bash
http POST localhost:8080/authors name=joao email=joao.diogo.vicente@gmail.com
```

and the output shows the DTO returned as expected

```bash
HTTP/1.1 200 
Content-Type: application/json;charset=UTF-8
Date: Sat, 13 Jan 2018 23:00:26 GMT
Transfer-Encoding: chunked

{
    "email": "joao.diogo.vicente@gmail.com", 
    "name": "joao"
}
```

## MongoDB enters

