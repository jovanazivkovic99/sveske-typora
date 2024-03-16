### ResponseEntity header

```java
return ResponseEntity.ok().header("custom-header", "jovana")
        .body(student);
```

### @SpringBootApplication

```java
@Configuration @EnableAutoConfiguration @ComponentScan
```

### @RestController

```java
@Controller + @ResponseBody
```

## Configure MySQL connection

in MySQL Workbanch create database: 

```sql
create database user_management
```

in application.yml:

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/user_management
    username: root
    password: password
  jpa:
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQLDialect
    hibernate:
      ddl-auto: update
```

username i password su oni prilikom instalacije MySQLa.

### GenerationType.IDENTITY

koristi nacin baze za auto increment

```
@GeneratedValue(strategy = GenerationType.IDENTITY)
```

### Records

contain fields, all-args constructor, getters, \*toString,\* and \*equals/hashCode\* methods

## MapStruct + Lombok

https://github.com/mapstruct/mapstruct-examples/blob/main/mapstruct-lombok/pom.xml

```xml
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>${org.mapstruct.version}</version>
</dependency>
```

```xml
<properties>
    <java.version>17</java.version>
    <org.mapstruct.version>1.5.5.Final</org.mapstruct.version>
    <org.projectlombok.version>1.18.20</org.projectlombok.version>
    <lombok-mapstruct-binding.version>0.2.0</lombok-mapstruct-binding.version>
</properties>
```

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.8.1</version>
    <configuration>
        <source>17</source>
        <target>17</target>
        <annotationProcessorPaths>
            <path>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct-processor</artifactId>
                <version>${org.mapstruct.version}</version>
            </path>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>${org.projectlombok.version}</version>
            </path>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok-mapstruct-binding</artifactId>
                <version>${lombok-mapstruct-binding.version}</version>
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

**Primer maper interfejsa:**



## @ExceptionHandler

Hendluje custom excpetione i salje custom odgovor klijentu.

Moze da se stavi direktno u kontroler gde zelimoda hendlujemo izuzetak:

```java
@ExceptionHandler(ResourceNotFoundException.class)
public ResponseEntity<ErrorDetails> handleResourceNotFoundException (ResourceNotFoundException exception,
                                                                     WebRequest webRequest) {
    ErrorDetails errorDetails = new ErrorDetails(LocalDateTime.now(), exception.getMessage(),
                                                 webRequest.getDescription(false), "USER_NOT_FOUND");
    return new ResponseEntity<>(errorDetails, HttpStatus.NOT_FOUND);
}
```

ILI BOLJE da se napravi Global Exception Handler:

```java
@ControllerAdvice
public class GlobalExceptionHandler
```

## @Validations

U DTO staviti valiadacije i onda u kontroleru kod @RequestBody staviti anotaciju @Valid

```java
@PostMapping
@ResponseStatus(HttpStatus.CREATED)
public UserDto createUser (@Valid @RequestBody UserDto user) {
    return userService.createUser(user);
}
```

### Customize validation error response:

to radimo tako sto extendujemo global exception handler sa **ResponseEntityExceptionHandler** klasom jer ona sadrzi metodu za hendlovanje gresaka za validacije **handleMethodArgumentNotValid**

```java
@ControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler
```

```java
@Nullable
protected ResponseEntity<Object> handleMethodArgumentNotValid(MethodArgumentNotValidException ex, HttpHeaders headers, HttpStatusCode status, WebRequest request) {
    return this.handleExceptionInternal(ex, (Object)null, headers, status, request);
}
```

metodu iznad treba da customizujemo:

```java
@Override
protected ResponseEntity<Object> handleMethodArgumentNotValid (MethodArgumentNotValidException ex,
                                                               HttpHeaders headers, HttpStatusCode status,
                                                               WebRequest request) {
    Map<String, String> errors = new HashMap<>();
    List<ObjectError> allErrors = ex.getBindingResult()
                                    .getAllErrors();
    allErrors.forEach((error) -> {
        String fieldName = ((FieldError) error).getField();
        String message = error.getDefaultMessage();
        errors.put(fieldName, message);
    });
    return new ResponseEntity<>(errors, HttpStatus.BAD_REQUEST);
}
```

i izgleda ovako:

![image-20230913133638522](C:\Users\Malkier_4\AppData\Roaming\Typora\typora-user-images\image-20230913133638522.png)

Ako hocemo da customizujemo poruku koja se pojavljuje, tj da umesto 'must not be empty' pise nesto drugo to radimo ovako:

```java
@NotEmpty(message = "User email must not be null or empty.")
@Email (message = "Email address must be valid.")
String email
```

i dobijamo: 

![image-20230913134613877](C:\Users\Malkier_4\AppData\Roaming\Typora\typora-user-images\image-20230913134613877.png)

## Actuator

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

kada pokrenemo aplikaciju vidimo da po defaultu exposujemo samo jedan endpoint '/actuator'. kako bismo exposovali sve endpointe u yml file stavljamo ovo:

```yaml
management:
   endpoints:
      web:
        exposure:
          include: "*"
```

sada smo exposovali 13 endpointova.

http://localhost:8081/actuator/info - da vidimo info o aplikaciji koje smo napisali u application.yml fajlu:

```yaml
info:
  app:
    name: Spring Boot Restful Web Services
    description: Spring Boot Restful Web Services Demo
    version: 1.0.0
```

http://localhost:8081/actuator/health - info o statusu app (ovo po defaultu prikazuje), disk space, bazama... ovako prikazujemo sve detalje za health:

```yaml
management:
    endpoint:
    	health:
    		show-details: always
```

http://localhost:8081/actuator/beans - da vidimo sve beanove:

```json
"userServiceImpl": {
    "aliases": [],
    "scope": "singleton",
    "type": "com.jovana.springbootrestfulwebservices.service.UserServiceImpl",
    "resource": "file [C:\\Users\\Malkier_4\\Downloads\\springboot-restful-			webservices\\target\\classes\\com\\jovana\\springbootrestfulwebservices\\service\\UserServiceImpl.class]",
    "dependencies": [
        "userRepository",
        "userMapperImpl"
    ]
},
```

http://localhost:8081/actuator/conditions - prikazuje auto configuration report, kategorisuci u positiveMatches i negativeMatches

http://localhost:8081/actuator/mappings - prikazuje sve @RequestMapping putanje. Korisan da vidimo koja metoda ima kojju putanju.

http://localhost:8081/actuator/configprops - ukljucuje svu konfiguraciju definisanu @ConfigurationProperties beanom, kao i onu iz application.yml fajla

http://localhost:8081/actuator/metrics - koliko memorije koristi, koliko je memorije slobodno, velicina iskoriscenog heapa, broj niti koje koristi. Prikaze nam listu **meter names** i onda izberemo neki od njih npr **/jvm.memory.max**.

http://localhost:8081/actuator/env - prikazuje sve propertije iz **ConfigurableEnvironment** interfejsa, kao sto su lista aktivnih profila, application properties, sistemske env varijable

http://localhost:8081/actuator/threaddump - thread dump sa trenutnim nitima i JVM stack trace

http://localhost:8081/actuator/loggers - da vidimo i menjamo log level aplikacije

http://localhost:8081/actuator/shutdown - za lepo gasenje aplikacije. Nije po defaultu ukljuceno, dodaje se ovako:

```yaml
management:
    endpoint:
    	shutdown:
    		enabled: true
```

I salje se POST zahtev, a ne get.

## SpringDoc

**springdoc-openapi** - java biblioteka koja omogucava integraciju izmedju spring boota i swaggera

```xml
<!-- https://mvnrepository.com/artifact/org.springdoc/springdoc-openapi-starter-webmvc-ui -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.2.0</version>
</dependency>
```

i doda se u main klasi anotacija **@OpenAPIDefinition**:

```java
@SpringBootApplication
@OpenAPIDefinition(
        info = @Info(
                title = "Spring Boot REST API Documentation",
                description = "Spring Boot REST API Documentation",
                version = "v1.0",
                contact = @Contact(
                        name = "Jovana",
                        email = "jovanakipar@gmail.com",
                        url = "https://github.com/jovanazivkovic99"
                ),
                license = @License(
                        name = "Apache 2.0",
                        url = "https://github.com/jovanazivkovic99"
                )
        ),
        externalDocs = @ExternalDocumentation(
                description = "Spring Boot User Management Documentation",
                url = "https://github.com/jovanazivkovic99"
        )
)
```

kako bismo na swaggeru stavili opis za svaku metodu, prvo iznad kontrolera stavimo anotaciju **@Tag**:

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
@Tag(
        name = "User Controller",
        description = "Operations related to user"
)
public class UserController
```

a iznad CREATE metode stavljamo: status 201

```java
@Operation(
        summary = "Create user",
        description = "Save user to database"
)
@ApiResponse(
        responseCode = "201",
        description = "HTTP Status 201 CREATED"
)
@PostMapping
@ResponseStatus(HttpStatus.CREATED)
public UserDto createUser (@Valid @RequestBody UserDto user) {
    return userService.createUser(user);
}
```

za GET: status 200

```java
@Operation(
        summary = "Get user by id",
        description = "Get single user by id from database"
)
@ApiResponse(
        responseCode = "200",
        description = "HTTP Status 200 SUCCESS"
)
@GetMapping("/{id}")
public UserDto getUserById (@PathVariable Long id) {
    return userService.getUserById(id);
}
```

**Model documentation za swagger:**

kako customizova ovo:

![image-20230914123728671](C:\Users\Malkier_4\AppData\Roaming\Typora\typora-user-images\image-20230914123728671.png)

dodamo anotaciju **@Schema** iznad DTO klase i metode:

```java
@Builder
@Schema(
        description = "UserDto info"
)
public record UserDto(Long id,
                      @Schema(
                              description = "User First Name"
                      )
                      @NotEmpty(message = "User first name must not be null or empty.")
                      String firstName,
                      
                      @Schema(
                              description = "User Last Name"
                      )
                      @NotEmpty(message = "User last name must not be null or empty.")
                      String lastName,
                      @NotEmpty(message = "User email must not be null or empty.")
                      @Email(message = "Email address must be valid.")
                      String email) {
    
}
```

i ovako izgleda nakon pokretanja:

![image-20230914123942739](C:\Users\Malkier_4\AppData\Roaming\Typora\typora-user-images\image-20230914123942739.png)

## Exceptions

1. **@ControllerAdvice**:
   - This annotation allows you to handle exceptions across the whole application, not just within a single controller. It essentially lets you create a global exception handler.
2. **@ExceptionHandler**:
   - This method-level annotation is used inside a class marked with `@Controller` or `@RestController` to handle exceptions. When used within `@ControllerAdvice`, it can handle exceptions globally.
3. **@ResponseStatus**:
   - You can combine this annotation with `@ExceptionHandler` to specify the HTTP status code to be returned in the response when an exception is thrown. For example, you might return a 404 status for `ResourceNotFoundException`.
4. **ResponseEntityExceptionHandler**:
   - This is a base class provided by Spring that contains methods to handle specific exceptions thrown by Spring itself. You can extend this class and override its methods to customize the response for certain built-in exceptions.

# Microservice App

## Ports

![image-20230914161849170](C:\Users\Malkier_4\AppData\Roaming\Typora\typora-user-images\image-20230914161849170.png)

## Compatable versions of Spring Boot and Spring Cloud

Proveravamo na https://spring.io/projects/spring-cloud - skrolujemo dole i vidimo odeljak **Adding Spring Cloud To An Existing Spring Boot Application**.

## Department-service 8000

konekcija sa MySQL bazom:

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/department_db
    username: root
    password: password
  jpa:
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQLDialect
    hibernate:
      ddl-auto: update
```

## Employee-service 8081

## @RestTemplate:

sada hocemo da zajedno sa info o zaposlenom vratimo i njegov department code. U employee servisu radimo sledece:

- U Employee entity stavljamo polje `private String departmentCode;`

- Pravimo `DepartmentDto`, znaci isti dto kao sto vraca metoda `getDepartmentByCode` u department servisu koju treba da gadjamo. ***Polja se isto zovu u oba dto, kako se radi ako se ne zovu isto?***

- dodajemo `String departmentCode` i u EmployeeDto posto smo ga prethodno dodali u Employee entity da bi moglo da se mapira i prikaze

- Pravimo RestTemplate Spring Bean u glavnoj klasi:

  ```java
  @Bean
  public RestTemplate restTemplate () {
      return new RestTemplate();
  }
  ```

- injectujemo RestTemplate u EmployeeServis

- unutar metode `getEmployeeById` pozivamo `restTemplate.getForEntity`

  ![image-20230915121022723](C:\Users\Malkier_4\AppData\Roaming\Typora\typora-user-images\image-20230915121022723.png)

```java
ResponseEntity<DepartmentDto> responseEntity = restTemplate.getForEntity("http://localhost:8000/api/departments/code/" + employee.getDepartmentCode(), DepartmentDto.class);
```

- ovako za sad izgleda kod:

```java
@Override
public EmployeeDto getEmployeeById (Long id) {
    Employee employee = employeeRepository.findById(id)
                                          .orElseThrow(() -> new ResourceNotFoundException("Employee",
                                                                                           "id",
                                                                                           id.toString()));
    ResponseEntity<DepartmentDto> responseEntity = restTemplate.getForEntity(
            "http://localhost:8000/api/departments/" + employee.getDepartmentCode(),
            DepartmentDto.class);
    DepartmentDto departmentDto = responseEntity.getBody(); 
    return employeeMapper.employeeToEmployeeDto(employee); // treba vratiti oba dto depatment + employee
}
```

- s obzirom da treba oba dto vratiti, potrebno je da oba smestimo u jedan dto:

  ```java
  public record APIResponseDto(EmployeeDto employeeDto,
                               DepartmentDto departmentDto) {
      
  }
  ```

- final:

  ```java
  @Override
  public APIResponseDto getEmployeeById (Long id) {
      Employee employee = employeeRepository.findById(id)
                                            .orElseThrow(() -> new ResourceNotFoundException("Employee",
                                                                                             "id",
                                                                                             id.toString()));
      ResponseEntity<DepartmentDto> responseEntity = restTemplate.getForEntity(
              "http://localhost:8000/api/departments/code/" + employee.getDepartmentCode(),
              DepartmentDto.class);
      DepartmentDto departmentDto = responseEntity.getBody();
      
      
      EmployeeDto employeeDto = employeeMapper.employeeToEmployeeDto(employee);
      return APIResponseDto.builder()
                           .employeeDto(employeeDto)
                           .departmentDto(departmentDto)
                           .build();
  }
  ```

## WebClient

- dodajemo webflux dependency 

  ```xml
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-webflux</artifactId>
  </dependency>
  ```

- dodajemo WebClient bean

  ```java
  @Bean
  public WebClient webClient () {
      return WebClient.builder()
                  .build();
  }
  ```

- injektujemo webclienta u servis

- i na ovaj nacin gadjamo API iz drugog servisa:

  ```java
  webClient.get()
             .uri("http://localhost:8000/api/departments/code/" + employee.getDepartmentCode())
             .retrieve()
             .bodyToMono(DepartmentDto.class)
             .block();
  ```

- final:

  ```java
  @Override
  public APIResponseDto getEmployeeById (Long id) {
      Employee employee = employeeRepository.findById(id)
                                            .orElseThrow(() -> new ResourceNotFoundException("Employee",
                                                                                             "id",
                                                                                             id.toString()));
    
      DepartmentDto departmentDto = webClient.get()
                                             .uri("http://localhost:8000/api/departments/code/" + employee.getDepartmentCode())
                                             .retrieve()
                                             .bodyToMono(DepartmentDto.class)
                                             .block();
      
      EmployeeDto employeeDto = employeeMapper.employeeToEmployeeDto(employee);
      return APIResponseDto.builder()
                           .employeeDto(employeeDto)
                           .departmentDto(departmentDto)
                           .build();
  }
  ```

## OpenFeign

- Na spring initializr biramo dependency **OpenFeign**

  ```xml
  <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
      </dependency>
  ```

- kad god dodajemo Spring Cloud dependency moramo da dodamo i dependency za njega:

  ```xml
  <dependencyManagement>
      <dependencies>
        <dependency>
          <groupId>org.springframework.cloud</groupId>
          <artifactId>spring-cloud-dependencies</artifactId>
          <version>${spring-cloud.version}</version>
          <type>pom</type>
          <scope>import</scope>
        </dependency>
      </dependencies>
    </dependencyManagement>
  ```

```xml
<spring-cloud.version>2022.0.4</spring-cloud.version>
```

- nad glavnom klasom dodajemo anotaciju **@EnableFeignClients** - omogucava component scanning za interfejse koji su Feign klijenti

- A da bismo kreirali Feign klijenta, potrebno je da napravimo interfejs i da ga anotiramo sa **@FeignClient** i ova biblioteka ce automatski kreirati implementaciju za ovaj interfejs

- U ovaj interfejs **stavljamo metodu koju employee servis treba da pozove**, u ovom slucaju **getDepartmentByCode**. U @GetMapping deo stavljamo celu putanju, a kod @FeignClient stavljamo base url

  ```java
  @FeignClient(url = "http://localhost:8000", value = "DEPARTMENT-SERVICE")
  public interface APIClient {
      
      @GetMapping("/api/departments/code/{code}")
      DepartmentDto getDepartmentByCode (@PathVariable String code);
      
  }
  ```

- unutar servisa injektujemo APIClient tj feign client

- final:

  ```java
  @Override
  public APIResponseDto getEmployeeById (Long id) {
      Employee employee = employeeRepository.findById(id)
                                            .orElseThrow(() -> new ResourceNotFoundException("Employee",
                                                                                             "id",
                                                                                             id.toString()));
    
      DepartmentDto departmentDto = apiClient.getDepartmentByCode(employee.getDepartmentCode());
      
      EmployeeDto employeeDto = employeeMapper.employeeToEmployeeDto(employee);
      return APIResponseDto.builder()
                           .employeeDto(employeeDto)
                           .departmentDto(departmentDto)
                           .build();
  }
  ```

## Service Registery and Discovery

**Netflix Eureka**

dodajemo service-registry model u postojeci spring boot projekat tako sto idemo na File->Project Structure...->Modules->izaberemo +.

service-registry ima samo jedna dependency a to je Eureka Server:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>3.1.3</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.jovana</groupId>
	<artifactId>service-registry</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>service-registry</name>
	<description>Spring Boot project for service registry</description>
	<properties>
		<java.version>17</java.version>
		<spring-cloud.version>2022.0.4</spring-cloud.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>
	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

</project>

```

- dodajemo @EnableEurekaServer anotaciju nad glavnom klasom u service-registry

- Po defaultu eureka server je i eureka klijent i moramo da **diasable eureka client za ovaj eureka server**:

  ```yaml
  eureka:
    client:
      register-with-eureka: false
      fetch-registry: false
  ```

- ostale konfiguracija ime applikacije i port:

  ```yaml
  spring:
    application:
      name: SERVICE-REGISTRY
  server:
    port: 8761
  ```

- **department-service**: bismo registrovali department service kao eureka klijent tj da se registruje u eureka server, dodajemo u njegov pom.xml dependency:

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
  ```

- i u application.yml:

  ```yaml
  eureka:
    client:
      serviceUrl:
        defaultZone: http://localhost:8761/eureka/
  ```

- takodje dodajemo i application name kako bi eureka server koristio application name kao service id

- kada pokrenemo sve service i odemo na localhost:8761 vidimo sledece:

  ![image-20230920132948159](C:\Users\Malkier_4\AppData\Roaming\Typora\typora-user-images\image-20230920132948159.png)

**KAKO POKRENUTI VISE INSTANCI SERVISA**

prvo treba da kreiramo jar fajl projekta i onda cemo pokrenuti taj jar fajl u drugom portu

- odemo na depratment-service i onda na maven->package

- u target folderu vidimo jar fajl koji je generisan

- odemo u INTELLIJ terminal (moze bilo koji) i udjemo u folder department-service

  ![image-20230920133314944](C:\Users\Malkier_4\AppData\Roaming\Typora\typora-user-images\image-20230920133314944.png)

- kucamo komandu: **java -jar -Dserver.port=8082 target/ime_jar_fajla.jar** (8082 je novi port) i sad vidimo dve instance servisa

![image-20230921150845524](C:\Users\Malkier_4\AppData\Roaming\Typora\typora-user-images\image-20230921150845524.png)

## Load balancing with Eureka, Open Feign and Spring Cloud LoadBalancer

- u slucaju da se desi da jedna instanca servisa ne radi, nedostupna je itd.. potrebno je da aplikaciji kazemo dinamicki da gadja sledecu dostupnu instancu, a to treb uraditi **dinamicki**

- za sad je to hard-codovano:

  ```java
  @FeignClient (url = "http://localhost:8000", value = "DEPARTMENT-SERVICE")
  ```

- sve sto treba da uradimo jeste da sklonimo konkretan port koji gadjamo i da ostavimo samo ime servisa DEPARTMENT-SERVICE:

  ```java
  @FeignClient (value = "DEPARTMENT-SERVICE")
  ```

- unutar dependecija koje ima employee-service, vidimo da u okviru netflix eureka client dependencija ima i interni load balancer:

  ![image-20230921151516231](C:\Users\Malkier_4\AppData\Roaming\Typora\typora-user-images\image-20230921151516231.png)

- kako bismo testirali da li load balancing radi, tesiracemo tako sto cemo zaustaviti jednu instancu (8000) i da testiramo neki endpoint - 8000 nece raditi dok 8082 instanca hoce.
- sad cemo uraditi obrnuto - pokrenucemo 8000 i stopirati 8082 - to radimo tako sto u terminalu idemo Ctrl+C, a zatim pokrenemo 8000