---
layout: post
title: 'Event driven with Spring REST, Kafka and MongoDB'
published: true
---
The aim of this post is to illustrate how to build a simple Event Driven application using: 
* Spring REST
* Kafka
* MongoDB

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

So the application runs but is does not do anything useful, so lets stop the app now (using `ctrl+C`) and let's create a `CreateAuthorController` by editing `./src/main/java/io/github/joaovicente/CreateAuthorController.java` and add  POST capabilities

The handler method as shown below, to expose`POST /authors`

```java
package io.github.joaovicente.stories;

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

which for the time being will simply take in a  `./src/main/java/io/github/joaovicente/CreateAuthorDto.java` (below) and return it back to the caller

```java
package io.github.joaovicente.stories;

import lombok.Data;

@Data
public class CreateAuthorDto {
    private String name;
    private String email;
}
```

Let's run the app

```bash
$ mvn spring-boot:run
```

and create a new author via the  `POST` endpoint

> We are going to use [httpie](https://httpie.org/) instead of curl to interact with the REST interface because it is so much more expressive

```bash
$ http POST localhost:8080/authors name=joao email=joao.diogo.vicente@gmail.com
```

and the output shows the DTO returned as expected

```
HTTP/1.1 200 
Content-Type: application/json;charset=UTF-8
Date: Sat, 13 Jan 2018 23:00:26 GMT
Transfer-Encoding: chunked

{
    "email": "joao.diogo.vicente@gmail.com", 
    "name": "joao"
}
```

## Enter Kafka

In an Event Driven system, a command handler would handle a command (e.g. create-author) and if validation passes it would then issue a author-created event.
The author-created event would then be published so that subscribers can consume it.

So, in the next step we are going to 
1. Create the author-created event (without validating the command because we're living dangerously) 
2. Publish the event into the author-created Kafka topic 
3. Consume it back in the service (so we can then persist it later on in MongoDB)

### Setting up Kafka
The easiest way to get Kafka up-and-running is by using the Confluent Kafka Docker OSS images. For the purpose of this article, we're going to create the simplest Kafka deployment, which requires the [Kafka](https://hub.docker.com/r/confluentinc/cp-kafka/) and the [Zookeepeer](https://hub.docker.com/r/confluentinc/cp-zookeeper/) Docker images.

Now let's create a `docker-compose-kafka.yml` Docker Compose file which we are going to use to bring-up Kafka.

```yaml
---
version: '2'
services:
  zookeeper:
    image: "confluentinc/cp-zookeeper:4.0.0"
    hostname: zookeeper
    ports:
      - '32181:32181'
    environment:
      ZOOKEEPER_CLIENT_PORT: 32181
      ZOOKEEPER_TICK_TIME: 2000
    extra_hosts:
      - "moby:127.0.0.1"

  kafka:
    image: "confluentinc/cp-kafka:4.0.0"
    hostname: kafka
    ports:
      - '9092:9092'
      - '29092:29092'
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:32181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    extra_hosts:
      - "moby:127.0.0.1"
```

There is a useful command line utility called [kafkacat](https://github.com/edenhill/kafkacat) which is handy to interact with Kafka, for example, to pulish and consume messages from topics

Under linux Kafkacat can be installed using

```bash
$ sudo apt-get install kafkacat
```
 
### Start Kafka

```bash
$ docker-compose -f docker-compose-kafka.yml up
```

### Test Kafka

check the kafka broker is listening and has no topics

```bash
$ kafkacat -L -b localhost
```

you should see this
~~~
Metadata for all topics (from broker -1: localhost:9092/bootstrap):
 1 brokers:
  broker 1 at localhost:9092
 0 topics:
~~~

now try publishing a couple of messages using stdin to a `greeting` topic using `kafkacat`

```bash
$ kafkacat -P -b localhost -t greeting
hello1
hello2
```
exit using `Ctrl+C`

now consume the messages from `greeting` topic

```bash
$ kafkacat -C -b localhost -t greeting
```

if you see
~~~
hello1
hello2
~~~
in the console your Kafka is ready to go!



