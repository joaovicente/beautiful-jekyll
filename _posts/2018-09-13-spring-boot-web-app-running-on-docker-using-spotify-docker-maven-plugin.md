---
layout: post
published: true
title: Spring Boot web app running on Docker using Spotify Docker Maven plugin
---
In this post I am just going to get a Spring Boot web app running on docker using the [Spotify Maven plugin](https://github.com/spotify/docker-maven-plugin)

> Assumes you have [httpie](https://httpie.org/) installed

## Create Spring Boot web app
```bash
$ http http://start.spring.io/starter.zip dependencies==web javaVersion==8 -d

$ unzip demo.zip 
Archive:  demo.zip
  inflating: mvnw                    
   creating: .mvn/
   creating: .mvn/wrapper/
   creating: src/
   creating: src/main/
   creating: src/main/java/
   creating: src/main/java/com/
   creating: src/main/java/com/example/
   creating: src/main/java/com/example/demo/
   creating: src/main/resources/
   creating: src/main/resources/static/
   creating: src/main/resources/templates/
   creating: src/test/
   creating: src/test/java/
   creating: src/test/java/com/
   creating: src/test/java/com/example/
   creating: src/test/java/com/example/demo/
  inflating: .gitignore              
  inflating: .mvn/wrapper/maven-wrapper.jar  
  inflating: .mvn/wrapper/maven-wrapper.properties  
  inflating: mvnw.cmd                
  inflating: pom.xml                 
  inflating: src/main/java/com/example/demo/DemoApplication.java  
  inflating: src/main/resources/application.properties  
  inflating: src/test/java/com/example/demo/DemoApplicationTests.java 
unzip demo.zip
```

## Build the app (without docker)

```
$ mvn spring-boot:run
```

In another terminal test you can reach the app at the default `8080` port

```bash
$ http localhost:8080
HTTP/1.1 404 
Content-Type: application/json;charset=UTF-8
Date: Thu, 13 Sep 2018 11:03:26 GMT
Transfer-Encoding: chunked

{
    "error": "Not Found", 
    "message": "No message available", 
    "path": "/", 
    "status": 404, 
    "timestamp": "2018-09-13T11:03:26.805+0000"
}
```

There are no endpoints defined in the demo web app. We are just ensuring we get an http error response which proves that the app is running and reachable on `localhost:8080` 

Now stop the terminal that is running `mvn spring-boot:run`

## Add Spotify Docker Maven plugin
In `pom.xml`, add `docker.image.prefix` and `docker.baseImage` `properties` as shown below
```bash
<properties>
    ...
    <docker.image.prefix>joaovicente</docker.image.prefix>
    <docker.baseImage>openjdk:8</docker.baseImage>
</properties>
```

In `pom.xml`, add the `plugin` as shown below
```bash
<plugins>
    <plugin>
        <groupId>com.spotify</groupId>
            <artifactId>docker-maven-plugin</artifactId>
            <version>1.1.1</version>
            <configuration>
                 <imageName>${docker.image.prefix}/${project.artifactId}</imageName>
                 <imageTags>
                     <imageTag>${project.version}</imageTag>
                     <imageTag>latest</imageTag>
                 </imageTags>
                 <baseImage>frolvlad/alpine-oraclejdk8:slim</baseImage>
                 <entryPoint>["java", "-jar", "-XX:+UnlockExperimentalVMOptions", "-XX:+UseCGroupMemoryLimitForHeap", "/${project.build.finalName}.jar"]</entryPoint>
                 <!--<dockerDirectory>src/main/docker</dockerDirectory>-->
                 <resources>
                     <resource>
                         <targetPath>/</targetPath>
                         <directory>${project.build.directory}</directory>
                         <include>${project.build.finalName}.jar</include>
                     </resource>
                 </resources>
            </configuration>
    </plugin>
</plugins>
```

## Build docker image

```bash
$ mvn clean package docker:build
...
Successfully tagged joaovicente/demo:latest
[INFO] Built joaovicente/demo
[INFO] Tagging joaovicente/demo with 0.0.1-SNAPSHOT
[INFO] Tagging joaovicente/demo with latest
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 16.891 s
[INFO] Finished at: 2018-09-13T12:27:09+01:00
[INFO] Final Memory: 55M/592M
[INFO] ------------------------------------------------------------------------
```

Check if the image is available 
```
$ docker images | grep joaovicente
joaovicente/demo                                         0.0.1-SNAPSHOT      bdd080b8e19b        About a minute ago   179MB
joaovicente/demo                                         latest              bdd080b8e19b        About a minute ago   179MB
```

## Run the app in Docker
```bash
docker run -p 8080:8080 joaovicente/demo
```

In another terminal test you can reach the app at the default `8080` port published by Docker

```bash
$ http localhost:8080
HTTP/1.1 404 
Content-Type: application/json;charset=UTF-8
Date: Thu, 13 Sep 2018 11:03:26 GMT
Transfer-Encoding: chunked

{
    "error": "Not Found", 
    "message": "No message available", 
    "path": "/", 
    "status": 404, 
    "timestamp": "2018-09-13T11:03:26.805+0000"
}
```
