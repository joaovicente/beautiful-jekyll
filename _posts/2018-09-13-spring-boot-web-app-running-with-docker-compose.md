---
layout: post
published: true
title: Spring  Boot web app running with Docker Compose
tags: spring docker-compose
---
In this post I am going to get a Docker image running using Docker Compose
> Docker Compose allows for bundling of multiple images which can form a deployment package which makes it very easy to setup and teardown, and provides a self-contained network in which containers can talk to each other by name

This post requires a `joaovicente/demo:latest Docker` image which can be created by following instructions in the 
[Spring Boot web app running on Docker](https://joaovicente.github.io/2018-09-13-spring-boot-web-app-running-on-docker-using-spotify-docker-maven-plugin/) post

Docker Compose looks for a `docker-compose.yml` file in your current directory

So create one with the equivalent content to `docker run -p 8080:8080 joaovicente/demo`

```
version: '2'
services:
  demo:
    image: joaovicente/demo:latest
    ports:
      - "8080:8080"
```

When you run 

```bash
docker-compose up
```

Your application is up-and-running again and reachable on port `8080`

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
