# How i Dockerized alfa-sorb app:

## Dockerfile:

```dockerfile
FROM openjdk:21
COPY target/alfa-sorb.jar alfa-sorb.jar
ENTRYPOINT ["java", "-jar", "alfa-sorb.jar"]
RUN mvn -f /alfa-sorb/pom.xml clean package -DskipTests=true
```

## docker-compose.yml

```yaml
version: '3.9'

services:
  alfa_sorb_app:
    container_name: alfa_sorb_app
    image: alfa_sorb
    build: .
    ports:
      - "8081:8081"
    environment:
      - DATABASE_URL=jdbc:postgresql://alfa_sorb_db:5432/alfa-sorb
      - DATABASE_USERNAME=postgres
      - DATABASE_PASSWORD=password
    depends_on:
      - alfa_sorb_db


  alfa_sorb_db:
    container_name: alfa_sorb_db
    image: postgres:15.3
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata: {}
```

## application.yml

```yaml
server:
  port: 8081
spring:
  datasource:
    driver-class-name: org.postgresql.Driver
    password: ${DATABASE_PASSWORD}
    url: ${DATABASE_URL}
    username: ${DATABASE_USERNAME}
  jpa:
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
  servlet:
    multipart:
      max-file-size: 10MB
  mail:
    host: smtp.gmail.com
    port: 587
    username: alfasorb.doo@gmail.com
    password: "iltz kdzt zzug yudc"
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true
constants:
  email: "alfasorb.doo@gmail.com"
```