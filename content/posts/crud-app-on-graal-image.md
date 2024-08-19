---
author: ["Thariq Shah"]
title : 'CRUD Application with Spring boot 3.3 and GraalVM native Image'
date : 2024-08-18T18:58:31+05:30
draft : false
ShowToc: true
cover:
  image: /crud-app-on-graal-image/cover/crud-app-on-graal-image-swagger-cover.png
categories: [ 'how-to', 'GraalVM','Springboot']
tags: [ 'springboot', 'graalvm','crud','flyway','swagger']
series: [ 'Native Spring' ]
searchHidden: true
---

## Prerequisites

- Docker 
- Java 21
- Clone [inventory-service](https://github.com/thariqshah/inventory-service)

## Introduction

This article is the second part of [Low memory footprint with Spring boot 3.3 and Graalvm native image](sprinboot-3.3-on-graal-image.md).
Here, we will be building REST CRUD resources and package as a native executable within a container.
We are using flyway for database versioning and springdoc library for swagger documentation with a web UI.

## Flyway

Flyway is a database migration tool that helps you manage and apply database changes within your application
to achieve database environment parity.

### Setting up flyway for springboot project

Adding flyway dependency for postgres for maven project.

```xml
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-core</artifactId>
    </dependency>
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-database-postgresql</artifactId>
    </dependency>
```

Configurations in application.yml

```yaml
spring:
  datasource:
    url: ${SPRING_DATASOURCE_URL:jdbc:postgresql://localhost:5432/postgres}
    username: ${SPRING_DATASOURCE_USERNAME:postgres}
    password: ${SPRING_DATASOURCE_USERNAME:postgres}
    driver-class-name: org.postgresql.Driver
  jpa:
    hibernate:
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        default_schema: inventory
  flyway:
    schemas: inventory
    enabled: true

management:
  endpoint:
    health:
      show-details: always
  health:
    diskspace:
      enabled: false
```

### Managing database scripts

Database scripts are inside ```src/main/resources/db/migration``` and follows naming conventions
```V<version>__<name>.sql``` and flyway ensure that migrations are run in order of their versions.

Let's create file in  ```src/main/resources/db/migration``` with name ```V1__create_table_product.sql```

```sql
CREATE TABLE product
(
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_name VARCHAR(255) NOT NULL,
    in_stock     boolean default false
);
```
Flyway naming pattern matters [https://www.red-gate.com/blog/database-devops/flyway-naming-patterns-matter](https://www.red-gate.com/blog/database-devops/flyway-naming-patterns-matter)

## Adding swagger web UI 

Spring doc library helps to generate the API documentation for us.

```xml
   <dependency>
      <groupId>org.springdoc</groupId>
      <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
      <version>2.6.0</version>
   </dependency>
```
The swagger UI is accessible at http://server:port/context-path/swagger-ui.html

## Let's write boilerplate

- ### create entity class

@RegisterReflectionForBinding({UUID[].class}) is needed for GraalVM native Image to specify that some hints are needed for
reflection-based serialization for our entity class by hibernate.

```java
@RegisterReflectionForBinding({UUID[].class})
@Entity
@Table(name = "product")
public class Product {

    @Id
    @GeneratedValue
    @Column(name = "id", columnDefinition = "UUID")
    private UUID id;

    @Column(name = "product_name", nullable = false)
    private String productName;


    @Column(name = "in_stock")
    private boolean inStock;
    
    //generate getters and setters or use lombok's @Data
}
```

- ### extend jpa repository

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, UUID> {
}
```

- ### create rest controller

```java
@RequestMapping("/api/v1/product")
@RestController
public class ProductResource {

    private final ProductRepository productRepository;

    public ProductResource(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @PostMapping("/create")
    public ResponseEntity<Product> createProduct(@RequestBody Product product) {
        return new ResponseEntity<>(productRepository.save(product), HttpStatus.CREATED);
    }

    @GetMapping("/list")
    public ResponseEntity<Page<Product>> getAllProducts(@RequestParam(defaultValue = "0") int pageNumber, @RequestParam(defaultValue = "10") int pageSize) {
        return new ResponseEntity<>(productRepository.findAll(PageRequest.of(pageNumber, pageSize)), HttpStatus.OK);
    }

    @PutMapping("/update/{id}")
    public ResponseEntity<Product> updateProduct(@PathVariable UUID id, @RequestBody Product productDetails) {
        Optional<Product> optionalProduct = productRepository.findById(id);
        if (optionalProduct.isPresent()) {
            Product product = optionalProduct.get();
            product.setProductName(productDetails.getProductName());
            product.setInStock(productDetails.isInStock());
            return new ResponseEntity<>(productRepository.save(product), HttpStatus.OK);
        } else {
            return new ResponseEntity<>(HttpStatus.NOT_FOUND);
        }
    }

    @DeleteMapping("/delete/{id}")
    public ResponseEntity<Void> deleteProduct(@PathVariable UUID id) {
        Optional<Product> product = productRepository.findById(id);
        if (product.isPresent()) {
            productRepository.delete(product.get());
            return new ResponseEntity<>(HttpStatus.NO_CONTENT);
        } else {
            return new ResponseEntity<>(HttpStatus.NOT_FOUND);
        }
    }
}
```

## Running the project

build native OCI image

```bash
./mvn -Pnative spring-boot:build-image
```

create a docker network 

```bash
docker network create spring-native-app-network
```

spin up a Postgres database

```bash
docker run --rm --name postgres-db --network spring-native-app-network -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres -p 5432:5432 -d postgres:latest
```

spin up our inventory service container

```bash
docker run -d --rm --name inventory-service --network spring-native-app-network -e SPRING_DATASOURCE_URL=jdbc:postgresql://postgres-db:5432/postgres -e SPRING_DATASOURCE_USERNAME=postgres -e SPRING_DATASOURCE_PASSWORD=postgres -p 8080:8080 inventory-service:0.0.1-SNAPSHOT
```

goto [http://localhost:8080/swagger-ui/index.html](http://localhost:8080/swagger-ui/index.html)

![swagger](/crud-app-on-graal-image/crud-app-on-graal-image-swagger.JPG)


## Source

Repo url: [inventory-service](https://github.com/thariqshah/inventory-service/tree/fa77e702db21289de65c28a01d0c0fd3a2a1bd26)

