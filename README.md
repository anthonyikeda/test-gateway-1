# Test Gateway

Goal is to test out the Spring Cloud Gateway wit how Cors and Security work.

## Hypothesis
There is no CorsWebFilter autowired with the default, webflux Cloud Gateway when the `globalcors` configuration is included.

The expectation is that when you add the `spring.cloud.gateway.globalcors` values, an `@OnConditionalProperty` bean is created to suport the CorsWebFilter at thie highest priority.

Existing tests with Spring Security oAuth is that either the `CorsWebFilter` is not instantiated or the security filters have a higher precendence.


## Test 1 - CORS Declaration Only
Set up vanilla Cloud gateway with the following dependencies:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

and the following configuration:

```properties
spring.cloud.gateway.globalcors.cors-configurations.[/**].allowed-origins[0]=http://locahost:4200
spring.cloud.gateway.globalcors.cors-configurations.[/**].allowed-methods[0]=GET
spring.cloud.gateway.globalcors.cors-configurations.[/**].allowed-methods[1]=PUT
spring.cloud.gateway.globalcors.cors-configurations.[/**].allowed-headers[0]=Authorization
spring.cloud.gateway.globalcors.cors-configurations.[/**].allow-credentials=true

spring.cloud.gateway.routes[0].id=product-api
spring.cloud.gateway.routes[0].uri=http://localhost:8090
spring.cloud.gateway.routes[0].predicates[0]=Path=/product-api/{segment}
spring.cloud.gateway.routes[0].filters[0]=RewritePath=/product-api/?(?<segment>.*), /product/v1/$\{segment}

server.port=8020
```

### Expected Results

A CORS request for Origin of `http://localhost:4200` and Request Method of `GET` should be validated and return with:
* status of `200 OK``
* `Access-Control-Allow-Methods` of `GET` and `PUT`
* `Access-Control-Allow-Origin` of `http://localhost:4200`

### Results
Executable:

```bash
❯ http --print=BbHh OPTIONS http://localhost:8020/product-api/67909821 "Origin: http://localhost:4200" Access-Control-Request-Method:GET
OPTIONS /product-api/67909821 HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate
Access-Control-Request-Method: GET
Connection: keep-alive
Host: localhost:8020
Origin: http://localhost:4200
User-Agent: HTTPie/3.2.2



HTTP/1.1 200 OK
Access-Control-Allow-Methods: GET,PUT,OPTIONS
Access-Control-Allow-Origin: http://localhost:4200
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
content-length: 0
```

NEVER put quotes in the origins.


## Test 2 - Incorporate the Standard Spring Web dependency

We will extend Test 1 and add in the following dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

### Expected Results

A CORS request for Origin of `http://localhost:4200` and Request Method of `GET` should be validated and return with:
* status of `200 OK``
* `Access-Control-Allow-Methods` of `GET` and `PUT`
* `Access-Control-Allow-Origin` of `http://localhost:4200`

### Results

Starting the applcation fails.

```
Error starting ApplicationContext. To display the condition evaluation report re-run your application with 'debug' enabled.
2023-09-22T09:53:39.012-07:00 ERROR 91819 --- [           main] o.s.b.d.LoggingFailureAnalysisReporter   : 

***************************
APPLICATION FAILED TO START
***************************

Description:

Spring MVC found on classpath, which is incompatible with Spring Cloud Gateway.

Action:

Please set spring.main.web-application-type=reactive or remove spring-boot-starter-web dependency.

[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  5.492 s
[INFO] Finished at: 2023-09-22T09:53:39-07:00
[INFO] ------------------------------------------------------------------------
[ERROR] Failed to execute goal org.springframework.boot:spring-boot-maven-plugin:3.1.4:run (default-cli) on project test-gateway-1: Process terminated with exit code: 1 -> [Help 1]
[ERROR] 
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR] 
[ERROR] For more information about the errors and possible solutions, please read the following articles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoExecutionException
```

Both `springboot-start-web` and `springboot-starter-webflux` cannot coexist.

## Test 3 - Cors Declaration and Property Passthrough to a CorsWebFilter

Extend Test 1 configuration with adding a CorsWebFilter Bean passing through the GlobalCorsProperties

### Changes

In a new class `WebConfiguration` class we are going to add the following bean:

```java
package com.example.testgateway1;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cloud.gateway.config.GlobalCorsProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.reactive.CorsWebFilter;
import org.springframework.web.cors.reactive.UrlBasedCorsConfigurationSource;

@Configuration
public class WebConfiguration {
    private final Logger log = LoggerFactory.getLogger(getClass().getCanonicalName());

    @Bean
    CorsWebFilter corsWebFilter(GlobalCorsProperties properties) {
        log.info("Registering CorsWebFilter...");
        properties.getCorsConfigurations().keySet().stream().forEach(key -> {
            log.info(key);
        });

        CorsConfiguration corsConfig = properties.getCorsConfigurations().get("/**");

        UrlBasedCorsConfigurationSource source =
                new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", corsConfig);

        return new CorsWebFilter(source);
    }
}
```

### Expected Results

A CORS request for Origin of `http://localhost:4200` and Request Method of `GET` should be validated and return with:
* status of `200 OK``
* `Access-Control-Allow-Methods` of `GET` and `PUT`
* `Access-Control-Allow-Origin` of `http://localhost:4200`

### Results

The request passes straight through to the proxied service without activating the CORS controls.

```bash
http --print=HhBb OPTIONS http://localhost:8020/product-api/54321 "Origin: http://localhost:4200" Access-Control-Request-Method:GET Access-Control-Request-Headers:Authorization
OPTIONS /product-api/54321 HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate
Access-Control-Request-Headers: Authorization
Access-Control-Request-Method: GET
Connection: keep-alive
Host: localhost:8020
Origin: http://localhost:4200
User-Agent: HTTPie/3.2.2



HTTP/1.1 403 Forbidden
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
content-length: 0
```

# References

https://www.baeldung.com/spring-show-all-beans
