---
layout: single
title:  "Use MongoDB with SpringBoot 2.3.1"
date:   2020-07-07 12:00:01 +0200
categories: tutorial java spring-boot mongodb
toc: true
---

This page is describing how to introduce mongodb as a data serialization layer in java with the latest version of Spring.

## Pre-Requisites

### Maven and Java

This tutorial explainsfully relies on Maven. You'll have to make sure maven is available on your computer. On MacOS, Maven can be installed using the following command:

```bash
$ brew install maven
````

As recommended, please update your `.bash_profile` to include Java as part of your PATH.

```bash
export PATH="/usr/local/opt/openjdk/bin:$PATH"
```

### Inital Project setup

We are going to implement mongodb on top of the example described in [this post](https://yannick-lamothe.github.io/tutorial/java/spring-boot/swagger/openapi/api/generate-api-spring-boot-swagger/). Please follow at least initial project setup before continuing here.

### Launching Mongo

Docker is used to launch a mongo database which will be accessed from our local code.

```bash
$ docker run -p 27017:27017 mongo:3.6.15
```

## MongoTemplate approach

### Mongo initialisation

In this tutorial, we're going to use MongoTemplate approach. This approach is providing a simplified approach to data persistence, leveraging standard template pattern in Spring.

In order to use Mongo, modify your list of dependencies by adding following library in your ``pom.xml``.

```xml
<!-- mongo -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

This setup is sufficient to have mongo connecting to a local mongodb at Spring application startup time.

### Application code

Application java code is quite straight forward.

```java
import static org.springframework.data.mongodb.core.query.Criteria.where;
import org.springframework.data.mongodb.core.MongoOperations;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.query.Query;
import com.mongodb.client.MongoClients;
```

```java
MongoOperations mongoOps = new MongoTemplate(MongoClients.create(), "test");
mongoOps.insert(data);
log.info(mongoOps.findOne(new Query(where("id").is(serviceName)), Data.class).toString());
mongoOps.dropCollection("data");
```

### Setting database endpoint

Database endpoint can be customized in ``application.yml``.

```yaml
spring:
    data:
        mongodb:
            uri: mongodb://root:root@localhost:27017/test
```