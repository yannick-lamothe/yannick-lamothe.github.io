---
layout: single
title:  "Generate an API implementation in Java using Spring-Boot and Swagger-Codegen"
date:   2020-07-07 00:00:01 +0200
categories: tutorial java spring-boot swagger openapi api
toc: true
---

This page is describing how to generate an api in Java based on an OpenAPI specification.

## Pre-Requisites

This tutorial explainsfully relies on Maven. You'll have to make sure mavven is available on your computer. On MacOS, Maven can be installed using the following command:

```bash
$ brew install maven
````

As recommended, please update your `.bash_profile` to include Java as part of your PATH.

```bash
export PATH="/usr/local/opt/openjdk/bin:$PATH"
```

## Initial Project setup

### Generate the API code

This document does not cover OpenAPI specification. For more details, please refer to [OpenAPI Specification](https://swagger.io/specification/v2/). In this tutorial, we'll use [the following specification](/assets/files/api.yaml).

In order to generate API code, please make sure your use the latest [swagger-codegen](https://repo1.maven.org/maven2/io/swagger/swagger-codegen-cli/2.4.14/). You can generate the code by running the following command:

```bash
$ java -jar swagger-codegen-cli-2.4.14.jar generate \
  -i api.yaml \
  --api-package com.cisco.datasvc.api \
  --model-package com.cisco.datasvc.model \
  --group-id com.cisco.datasvc \
  --artifact-id spring-swagger-codegen-datasvc \
  --artifact-version 0.0.1-SNAPSHOT \
  -l spring \
  -o spring-swagger-codegen-datasvc
```

By default, code will not compile. Some binding libraries have to be added to `pom.xml` (maven dependencies file).

```xml
<!-- binding libraries -->
<dependency>
  <groupId>javax.xml.bind</groupId>
  <artifactId>jaxb-api</artifactId>
  <version>2.3.0</version>
</dependency>
<dependency>
  <groupId>com.sun.xml.bind</groupId>
  <artifactId>jaxb-core</artifactId>
  <version>2.3.0</version>
</dependency>
<dependency>
  <groupId>com.sun.xml.bind</groupId>
  <artifactId>jaxb-impl</artifactId>
  <version>2.3.0</version>
</dependency>
```

While navigating to `main/resources`, you will observe application properties are stored in a properties file. If you think this format is outdated and you prefer YAML format, run the following command. This step is optional and will not change this tutorial.

```bash
$ mvn io.codearte.props2yaml:props2yaml-maven-plugin:convert -Dproperties=application.properties
```

### Compile and run

Code can then be directly compiled using Maven. Dependencies will be automatically downloaded. 

```bash
$ mvn -e package -Dmaven.test.skip=true -DskipTests
```

Following target is produced ``target/spring-swagger-codegen-datasvc-0.0.1-SNAPSHOT.jar``, and can directly be executed. It will launch a tomcan server, listening on port 8080.

```bash
$ java -jar target/spring-swagger-codegen-datasvc-0.0.1-SNAPSHOT.jar
```

To verify the server is running, you can access ``http://localhost:8080/swagger-ui.html``from your favorite web browser. If OK, you should then be able to access your service via curl.

```bash
# This command should return an error related to missing header
$ curl -X GET --header 'Accept: application/json' 'http://localhost:8080/data?serviceName=abc'

# This one should return a 501 / Not implemented error, which is expected
$ curl -I -X GET --header 'Accept: application/json' --header 'Content-type: application/json' 'http://localhost:8080/data?serviceName=abc'
```

### Returning Data

As stated before, service currently returns a "501 - Not implemented" error. In order to return a Data object, ``DataApiController`` can be modified as follows:

```java
Data data = new Data();
data.setServiceName(serviceName);
data.setLastPull(new Date().toString());
data.setId("abc");
return new ResponseEntity<Object>(objectMapper.readValue(objectMapper.writeValueAsString(data), Object.class), HttpStatus.OK);
```

Note the error code will have to modified to return an error object.

## Authentication

This tutorial is covering the following:

* Basic Authentication
* Custom Oauth

### Basic Authentication

#### Generate code

[Swagger specification](https://swagger.io/docs/specification/2-0/authentication/basic-authentication/) describes how to add basic authentication in your API. Regenerate your code and modify ``pom.xml`` as described in the previous section.

#### Adding Authentication

It appears the generated code does not work well with older versions of Spring. Modify ``pom.xml`` to use latest SpringBoot version by specifying parent element version.

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.3.1.RELEASE</version>
</parent>
```

In order to protect uour API, add the following security library to ``pom.xml``. 

```xml
<!-- security libraries -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

That's it! Your API is now protected.

#### Customizing the Security Adapter

Being able to control which credentials can be used to access the API is key. This can be done by customizing the WebSecurityConfigurerAdapter, as follows. In this example, admin/password will be used to connect to the service.

```java
package io.swagger.configuration;
 
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import springfox.documentation.swagger2.annotations.EnableSwagger2;
 
@Configuration
@EnableSwagger2
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter
{
    @Override
    protected void configure(HttpSecurity http) throws Exception 
    {
        http
         .csrf().disable()
         .authorizeRequests().anyRequest().authenticated()
         .and()
         .httpBasic();
    }
  
    @Autowired
    public void configureGlobal(AuthenticationManagerBuilder auth) 
            throws Exception 
    {
        auth.inMemoryAuthentication()
            .withUser("admin")
            .password("{noop}password")
            .roles("USER");
    }
}
```

#### Testing it out

```bash
$ curl -I -X GET --user admin:password --header 'Accept: application/json' --header 'Content-type: application/json' 'http://localhost:8080/data?serviceName=abc'
```

### Custom Oauth 

In this section, we are going to focus on how to put in place resource validation for oauth. We are making the following asumptions:
* This code assumes a Bearer token is retrieved from an external Authorization Server.
* This Bearer token will be passed when invoking the API
* The token validation is done against an external Resource Server, using POST method.

#### Add libraries

Following security libraries have to be added in ``pom.xml``.

```xml
<!-- security libraries oauth -->
<dependency>
    <groupId>org.springframework.security.oauth</groupId>
    <artifactId>spring-security-oauth2</artifactId>
    <version>2.5.0.RELEASE</version>
</dependency>        
<dependency>
    <groupId>org.springframework.security.oauth.boot</groupId>
    <artifactId>spring-security-oauth2-autoconfigure</artifactId>
    <version>2.3.1.RELEASE</version>
</dependency> 
```

#### Enable resource server

Enabling Oauth by specifying ``security.oauth2.resource.token-info-uri`` in ``application.yml`` is feasible. However, it assumes the token is checked using an HTTP GET, which is not our case. Therefore, we have to customize resource verification is handled.

Instead of duplicating code, [this article](https://medium.com/@supunbhagya/spring-oauth2-resourceserver-oauth2-security-authorization-code-grant-flow-9eb72fd5d27d) by Supun Bhagya is giving all the details on how to enable a custom resource service.

Please note ``CustomRemoteTokenService.java`` has to be modified for our needs:

```java
@Override
public OAuth2Authentication loadAuthentication(String accessToken) throws AuthenticationException, InvalidTokenException {
    String tokenValidationUrl = "https://<my_resource_sercer_url>&token=";
    HttpHeaders headers = new HttpHeaders();
    Map<String, Object> map = executePost(tokenValidationUrl + accessToken, headers);
    if (map == null || map.isEmpty() || map.get("error") != null) {
        throw new InvalidTokenException("Token not allowed");
    }
    return tokenConverter.extractAuthentication(map);
}
```

#### Testing Oauth

Get a token by placing a request to your authorization server. Then invoke your service as follows:

```bash
$ curl -i -X GET --header 'Authorization: Bearer <token>' --header 'Accept: application/json' --header 'Content-type: application/json' 'http://localhost:8080/data?serviceName=abc'
```
