---
layout: post
title: 'Event driven with Spring REST, Kafka and MongoDB'
published: true
---
The aim of this post is to do a study on building a very simple Event Driven application using Spring REST, Kafka and MongoDB

In this post we'll pretend we are creating a Story editing web application, where authors can create their account and then create stories.

Well, it may take a while until we get to create stories, as the focus really is to get hands on experience in building an Event Driven system using the building blocks described above, using Spring Boot, Docker, and on a later post we will build observabiliy into the application.

I'll throw in a few tooling goodies along the way (e.g. `Lombok`, `Httpie`, `Docker`, `Kafkacat`) which should make life easier. 

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

> You'll see MongoDB connection error at this point, because we don't have MongoDB running yet, so we'll ignore this error for now.

So the application runs but is does not do anything useful, so lets stop the app now (using `ctrl+C`) and let's create a `CreateAuthorController` by editing `./src/main/java/io/github/joaovicente/stories/CreateAuthorController.java` and add  POST capabilities

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

which for the time being will simply take in a `CreateAuthorDto`, in `./src/main/java/io/github/joaovicente/stories/CreateAuthorDto.java` and return it back to the caller

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

In an Event Driven system, a command handler would handle a command (e.g. `CreateAuthor`) and if validated it would then issue a `author-created` event.
The `AuthorCreated` event would then be published so that subscribers can consume it and do useful things with it.

So, in the next step, after we get Kafka running, we are going to:
1. Define the `AuthorCreated` event (without validating the command because we're living dangerously) 
2. Publish the event to the `author-created` Kafka topic 
3. Consume `author-created` it back in the service (so we can then persist it in MongoDB)

### Setting up Kafka
The easiest way to get Kafka up-and-running is by using the Confluent Kafka Docker OSS images. For the purpose of this article, we're going to create the simplest Kafka deployment, which requires the [Kafka](https://hub.docker.com/r/confluentinc/cp-kafka/) and the [Zookeepeer](https://hub.docker.com/r/confluentinc/cp-zookeeper/) Docker images.

> You will need to have Docker and Docker Compose installed on your host to continue. If you don't have it already, have a look in [Install Docker Compose documentation](https://docs.docker.com/compose/install)

Now let's create a `../docker-compose-kafka.yml` Docker Compose file which we are going to use to bring-up Kafka.

```yaml
---
version: '2'
services:
  zookeeper:
    container_name: zookeeper
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
    container_name: kafka
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

There is a useful command line utility named [kafkacat](https://github.com/edenhill/kafkacat) which is handy to interact with Kafka, for example, to publish and consume messages from topics

Under linux Kafkacat can be installed using

```bash
$ sudo apt-get install kafkacat
```
 
### Start Kafka

```bash
$ docker-compose -f ../docker-compose-kafka.yml up
```

> When you want to stop the containers define in the compose file `Ctrl+C` is not enough. To bring them down fully, you will need to run the inverse `down` command: `docker-compose -f ../docker-compose-kafka.yml down`

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
exit using `Ctrl+C` followed by `Return`

now consume the messages from `greeting` topic

```bash
$ kafkacat -C -b localhost -t greeting
```

if you see the following in the console your Kafka is ready to go!
~~~
hello1
hello2
~~~


## Defining the AuthorCreated event
The AuthorCreated event is the POJO which will be serialised and sent to the the `author-created` Kafka topic

`./src/main/java/io/github/joaovicente/stories/AuthorCreated.java`

```java
package io.github.joaovicente.stories;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.UUID;

@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class AuthorCreated {
    private final UUID id = UUID.randomUUID();
    private String name;
    private String email;
}
```

Using `lombok` `@Data` and `@Builder` allows us to create the POJO without having to write any boiler plate code such as getters, setters and builder.
We are also auto-generating an `id` of the `Author`.

## Publishing the `author-created` event to Kafka 

Before we can publish the event we have to create define configuration

The following `./src/main/resources/application.yaml` defines where to find the Kafka broker and the Kafka topic the application will be using

```yaml
kafka:
  bootstrap-servers: localhost:9092
```

`./src/main/java/io/github/joaovicente/stories/KafkaProducerConfig.java` creates all additionally configuration required for the Kafka Producer.

```java
package io.github.joaovicente.stories;

import java.util.HashMap;
import java.util.Map;

import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.core.DefaultKafkaProducerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.core.ProducerFactory;
import org.springframework.kafka.support.converter.StringJsonMessageConverter;

@EnableKafka
@Configuration
public class KafkaProducerConfig {

    @Value("${kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public Map<String, Object> producerConfigs() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

        return props;
    }

    @Bean
    public ProducerFactory<String, String> producerFactory() {
        return new DefaultKafkaProducerFactory<>(producerConfigs());
    }

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
        KafkaTemplate<String, String> template = new KafkaTemplate<>(producerFactory());
        template.setMessageConverter(new StringJsonMessageConverter());
        return template;
    }

    @Bean
    KafkaTopicSender sender()   {
        return new KafkaTopicSender();
    }
}
```

Next we create the `./src/main/java/io/github/joaovicente/stories/KafkaTopicSender.java` which will encapsulate the payload `MessageBuilder` mechanism and the `KafkaTemplate`  

```java
package io.github.joaovicente.stories;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.messaging.support.MessageBuilder;

public class KafkaTopicSender {
    @Autowired
    private KafkaTemplate<String, ?> kafkaTemplate;

    public void send(String topic, Object payload) {
        kafkaTemplate
                .send(MessageBuilder.withPayload(payload).setHeader(KafkaHeaders.TOPIC, topic).build());
    }
}
```

and we now update `./src/main/java/io/github/joaovicente/stories/CreateAuthorController.java` to create the `AuthorCreated` event, when receiving a `CreateAuthor` command (i.e. `POST /authors`) and send it through to the `author-created` topic

```java
package io.github.joaovicente.stories;

import lombok.extern.java.Log;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMethod;

@Log
@RestController
public class CreateAuthorController {
    @Autowired
    private KafkaTopicSender sender;
    private final String authorCreatedTopic = "author-created";

    @RequestMapping(value = "/authors", method = RequestMethod.POST)

    public AuthorCreated createAuthor(@RequestBody CreateAuthorDto dto) {
        log.info("create-author command received: " + dto.toString());
        AuthorCreated authorCreated = AuthorCreated.builder()
                .name(dto.getName())
                .email(dto.getEmail()).build();

        sender.send(authorCreatedTopic, ((Object) authorCreated));
        log.info("author-created event transmitted: " + authorCreated.toString());
        return authorCreated;
    }
}
```

At this point the application should be producing the `author-created` event every time the `create-author` command is received.

To try this it out ensure you still have Kafka running (rememeber ... `docker-compose -f ../docker-compose-kafka.yml up`) 

And build/run the application again

```bash
$ mvn spring-boot:run
```

Next issue a CreateAuthor command

```bash
$ http POST localhost:8080/authors name=joao email=joao.diogo.vicente@gmail.com
```

And you should have a message in the `author-created` Kafka topic

```bash
$ kafkacat -C -b localhost -t author-created
{"id":"67d6fd66-9f0c-47d9-9302-a19773644a85","name":"joao","email":"joao.diogo.vicente@gmail.com"}
```

## Consume `author-created`

First we are going to create the `./src/main/java/io/github/joaovicente/stories/KafkaConsumerConfig.java` class to handle consumer configuration

```java
package io.github.joaovicente.stories;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.support.converter.StringJsonMessageConverter;

import java.util.HashMap;
import java.util.Map;

@Configuration
@EnableKafka
public class KafkaConsumerConfig {

    @Value("${kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public Map<String, Object> consumerConfigs() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "group-id");

        return props;
    }

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        return new DefaultKafkaConsumerFactory<>(consumerConfigs(), new StringDeserializer(),
                new StringDeserializer());
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setMessageConverter(new StringJsonMessageConverter());

        return factory;
    }

    @Bean
    public KafkaTopicReceiver receiver() {
        return new KafkaTopicReceiver();
    }
}
```

Next we create `./src/main/java/io/github/joaovicente/stories/KafkaTopicReceiver.java`

```java
package io.github.joaovicente.stories;

import lombok.extern.java.Log;
import org.springframework.kafka.annotation.KafkaListener;

@Log
public class KafkaTopicReceiver {
    private final String authorCreatedTopic = "author-created";

    @KafkaListener(topics = authorCreatedTopic)
    public void receive(AuthorCreated authorCreated) {
        log.info("author-created event received: " + authorCreated.toString());
    }
}
```

Let's re-build and run the app 

```bash
$ mvn spring-boot:run
```

and execute the create author command

```bash
$ http POST localhost:8080/authors name=test email=test@gmail.com
```

We should now see the following in the Spring boot console
~~~
... author-created event received: AuthorCreated(id=37c2fed9-8181-4a4a-bbe6-122c16e752d1, name=test, email=test@gmail.com)
~~~

So, next we are going to bring-in MongoDB to persist the Author when it is created.

## Enter MongoDB

First let's enhance our compose file to include a MongoDB service

```bash
$ cp ../docker-compose-kafka.yml ../docker-compose-kafka-mongo.yml
```

Now add the MongoDB image to the yml

```yaml
...
  mongodb:
    container_name: mongodb
    image: mongo:3.0.4
    ports:
      - "27017:27017"
    command: mongod --smallfiles
```

So `docker-compose-kafka-mongo.yml` should now look like this 

```yaml
---
version: '2'
services:
  zookeeper:
    container_name: zookeeper
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
    container_name: kafka
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

  mongodb:
    container_name: mongodb
    image: mongo:3.0.4
    ports:
      - "27017:27017"
    command: mongod --smallfiles
```

When you start execute the docker-compose

```bash
$ docker-compose -f ../docker-compose-kafka-mongo.yml up
```

you should see log entries for `mongodb` service.

Now you should be able to access the mongodb container bash as follows

```bash
$ docker exec -it mongodb bash
```

And start the mongo CLI 

```bash
$ mongo
```

The mongo CLI is very handy to inspect DBs and collections. you can type `help` to find some useful commands

~~~
> help
	db.help()                    help on db methods
	db.mycoll.help()             help on collection methods
	sh.help()                    sharding helpers
	rs.help()                    replica set helpers
	help admin                   administrative help
	help connect                 connecting to a db help
	help keys                    key shortcuts
	help misc                    misc things to know
	help mr                      mapreduce

	show dbs                     show database names
	show collections             show collections in current database
	show users                   show users in current database
	show profile                 show most recent system.profile entries with time >= 1ms
	show logs                    show the accessible logger names
	show log [name]              prints out the last segment of log in memory, 'global' is default
	use <db_name>                set current database
	db.foo.find()                list objects in collection foo
	db.foo.find( { a : 1 } )     list objects in foo where a == 1
	it                           result of the last line evaluated; use to further iterate
	DBQuery.shellBatchSize = x   set default number of items to display on shell
	exit                         quit the mongo shell
~~~

> You should be able to inspect the `author` MongoDB collection we'll create shortly, automatically named by the `AuthorRepository` POJO using `db.author.find()` 

Now that we have MongoDB up and running, let's create the `./src/main/java/io/github/joaovicente/stories/Author.java` entity 

```java
package io.github.joaovicente.stories;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import org.springframework.data.annotation.Id;

import java.util.UUID;

@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class Author {
    @Id
    private String id;
    private String name;
    private String email;
}
```

and the MongoRepository `./src/main/java/io/github/joaovicente/stories/AuthorRepository.java` 

```java
package io.github.joaovicente.stories;

import org.springframework.data.mongodb.repository.MongoRepository;

public interface AuthorRepository extends MongoRepository<Author, String> {
}
```

nearly there ...

now we'll wire the MongoRepository to `./src/main/java/io/github/joaovicente/stories/KafkaTopicReceiver.java` 

```java
    @Autowired
    AuthorRepository authorRepository;
```

and construct and persist the Author entity when the KafkaReceiver receives the `AuthorCreated` event.

```java
        Author author = Author.builder()
                .id(authorCreated.getId().toString())
                .email(authorCreated.getEmail())
                .name(authorCreated.getName())
                .build();
        authorRepository.insert(author);
```

All together `./src/main/java/io/github/joaovicente/stories/KafkaTopicReceiver.java` will look like this

```java
package io.github.joaovicente.stories;

import lombok.extern.java.Log;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.annotation.KafkaListener;

@Log
public class KafkaTopicReceiver {
    private final String authorCreatedTopic = "author-created";
    @Autowired
    AuthorRepository authorRepository;

    @KafkaListener(topics = authorCreatedTopic)
    public void receive(AuthorCreated authorCreated) {
        log.info("author-created event received: " + authorCreated.toString());
        Author author = Author.builder()
                .id(authorCreated.getId().toString())
                .email(authorCreated.getEmail())
                .name(authorCreated.getName())
                .build();
        authorRepository.insert(author);
    }
}
```

> We should really not access the persistency layer inside the topic receiver, but hey, I've stopped counting transgressions at this point, and I am pinning it all on brevity and simplicity, because [it's only rock'n'roll, but I like it!](https://www.youtube.com/watch?v=JGaBlygm0UY)

So at this point we're persisting the Author entity into MongoDB, so it makes sense to retrieve it back via `GET /authors/{authorId}`, so let's write an `AuthorQueryController` to do so

Here's `./src/main/java/io/github/joaovicente/stories/AuthorQueryController.java`

```java
package io.github.joaovicente.stories;

import lombok.extern.java.Log;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@Log
@RestController
public class AuthorQueryController {
    @Autowired
    private AuthorRepository authorRepository;

    @RequestMapping(value = "/authors/{authorId}", method = RequestMethod.GET)
    public Author createAuthor(@PathVariable String authorId) {
        log.info("author query by id : " + authorId);
        Author author = authorRepository.findOne(authorId);
        return author;
    }
}
```

Now, let's give it a spin, by starting docker-compose (unless you still have it running, you wild thing!)

```bash
$ docker-compose -f docker-compose-kafka-mongo.yml up
```

Run the Spring Boot app

```bash
$ mvn spring-boot:run
```

Now create a new Author

```bash
$ http post localhost:8080/authors name=me email=me@gmail.com
HTTP/1.1 200 
Content-Type: application/json;charset=UTF-8
Date: Tue, 20 Feb 2018 22:43:50 GMT
Transfer-Encoding: chunked

{
    "email": "me@gmail.com", 
    "id": "b5c220e4-8a15-41aa-b667-ae3a34e32593", 
    "name": "me"
}

```

Now we should be able to `GET` by `id`

```bash
$ http get localhost:8080/authors/b5c220e4-8a15-41aa-b667-ae3a34e32593
HTTP/1.1 200 
Content-Type: application/json;charset=UTF-8
Date: Tue, 20 Feb 2018 22:45:07 GMT
Transfer-Encoding: chunked

{
    "email": "me@gmail.com", 
    "id": "b5c220e4-8a15-41aa-b667-ae3a34e32593", 
    "name": "me"
}
```

And hereby ends the illustration of the main building blocks to make up an event driven Spring Boot service.

In the next post I'll tidy up the code, organizing it into more organized packages and aim at introducing some built-in observability, as otherwise it is hard to understand what is going on, as there is quite a bit of asynchronounicity going on.

You can find all the code from this blog in <https://github.com/joaovicente/stories-event-driven-study-part-1>
