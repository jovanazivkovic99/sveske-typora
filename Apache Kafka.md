# Kafka Instalacija i Pokretanje ZooKeeper-a

## Preuzimanje Kafke

1. **Preuzimanje Kafka**: Preuzeto sa zvanične stranice Kafka: [Apache Kafka QuickStart](https://kafka.apache.org/quickstart).
   
   - Kliknuti na **Download** link koji je istaknut.
   
     ![image-20240128200821327](C:\Users\Jovana\AppData\Roaming\Typora\typora-user-images\image-20240128200821327.png)
   
     ![image-20240128200847107](C:\Users\Jovana\AppData\Roaming\Typora\typora-user-images\image-20240128200847107.png)

## Priprema i Konfiguracija

2. **Ekstrakcija i Preimenovanje**:
   - Ekstraktovati Kafku u željeni direktorijum, ja sam i preimenovala folder u `kafka_server` radi lakse navigacije

## Pokretanje ZooKeeper-a i Kafka Brokera

1. **Pokretanje ZooKeeper-a**:

- Vraćamo se na stranicu sa uputstvima za dalje korake.
- U Windows sistemu, komanda za pokretanje ZooKeeper-a se nalazi u `bin\windows` direktorijumu.
- Korišćena komanda za pokretanje ZooKeeper-a je:
  ```
  bin\windows\zookeeper-server-start.bat config\zookeeper.properties
  ```

1. ### Pokretanje Kafka Brokera: 9092 port

   - ```
     bin\windows\kafka-server-start.bat config\server.properties
     ```

1. ###  Kreiranje topica:

   - ```
     bin\windows\kafka-topics.bat --create --topic quickstart-events --bootstrap-server localhost:9092
     ```

1. ### Da vidimo detalje topica:

   - ```
     bin\windows\kafka-topics.bat --describe --topic quickstart-events --bootstrap-server localhost:9092
     ```

1. Da napisemo neki event u topic:

   vec postoji skripta koja to radi

   ```
   bin\windows\kafka-console-producer.bat --topic quickstart-events --bootstrap-server localhost:9092
   ```

   nakon sto kliknemo enter, mozemo da unesemo neki event u quickstart-events topic.

1. ### Citanje eventa

   - da bismo mogli da procitamo event, otvoricemo novi terminal koji ce predstaljati consumera i pejstovacemo ovo u terminal:

     ```
     bin\windows\kafka-console-consumer.bat --topic quickstart-events --from-beginning --bootstrap-server localhost:9092
     ```

     nakon sto kliknemo enter, videcemo da consumer vec slusa i da ce nam se prikazati poruka koju je producer upisao. ako opet producer posalje poruku, videcemo da se kod consumera ona automatski pojavi

## Pravljenje app u spring boot-u

Prvo u application.yml definisemo consumera:

```yaml
spring:
  kafka:
    consumer:
      bootstrap-servers: localhost:9092
      group-id: myGroup # cosumer grupa id
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
```

producer:

```yaml
    producer:
      bootstrap-servers: localhost:9092
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
```

dalje pravimo kafka topic:

```java
@Configuration
public class KafkaTopicConfig {

    @Bean
    public NewTopic jovanaTopic(){
        return TopicBuilder
                .name("jovana")
                .build();
    }
}
```

napravimo producera:

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class KafkaProducer {
    private final KafkaTemplate<String, String> kafkaTemplate;

    public void sendMessage(String msg) {
        log.info(format("Sending message to jovana Topic:: %s", msg));
        kafkaTemplate.send("jovana", msg);
    }
}
```

i ekspozujemo endpoint za slanje poruka:

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/v1/messages")
public class MessageController {
    private final KafkaProducer kafkaProducer;

    @PostMapping
    public ResponseEntity<String> sendMessage(@RequestBody String message){
        kafkaProducer.sendMessage(message);
        return ResponseEntity.ok("Message queued successfully");
    }
}
```

**Consumer iz terminala**

iz terminala u Intellij-u udjemo u kafka_server folder i tu pokrenemo komandu za citanje eventa sa odredjenog topika. komanda se nalazi iznad vec napisana, a ovako izgleda konkretno za nas slucaj, topic je 'jovana':

```
bin\windows\kafka-console-consumer.bat --topic jovana --from-beginning --bootstrap-server localhost:9092
```

Testiramo slanje poruka u postmanu:

![image-20240128212600695](C:\Users\Jovana\AppData\Roaming\Typora\typora-user-images\image-20240128212600695.png)

**Pravljenje pravog consumera u app**

```java
@Service
@Slf4j
public class KafkaConsumer {

    @KafkaListener(topics = "jovana", groupId = "myGroup")
    public void consumeMsg(String msg){
      log.info(format("Consuming the message from jovana Topic:: %s", msg));
      
    }
}
```

## Slanje JSON formata preko kafke

Kafka ima mnogo built in serializera i deserializera, ali NE i za json. Zato sto zele da izbegnu da nametnu specifican serialization format useru. Ali zato je Spring kafka uvela json serializer i deserializer koji mozemo da koristimo da konvertujemo java objekat u json i iz jsona.

prvo treba da promenimo konfiguraciju u application.yml

```yaml
spring:
  kafka:
    consumer:
      bootstrap-servers: localhost:9092
      group-id: myGroup
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      # value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer

    producer:
      bootstrap-servers: localhost:9092
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      # value-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

onda kreiramo objekat koji zelimo da saljemo kao producer i da konzumiramo kao consumer. **Paket se zove payload jer kad saljemo nesto to se zove payload**:

```java
@Getter
@Setter
@ToString
public class Student {
    private int id;
    private String firstName;
    private String lastName;
}
```

Producer se sada pravi malo drugacije posto saljemo json objekat:

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class KafkaJsonProducer {
    private  final KafkaTemplate<String, Student> kafkaTemplate;

    public void sendMessage(Student student){
        Message<Student> message = MessageBuilder
                .withPayload(student)
                .setHeader(KafkaHeaders.TOPIC, "jovana")
                .build();

        kafkaTemplate.send(message);
    }
}
```

```java
 @PostMapping("/json")
    public ResponseEntity<String> sendMessage(@RequestBody Student student){
        kafkaJsonProducer.sendMessage(student);
        return ResponseEntity.ok("Message queued successfully as JSON");
    }
```

medjutim potrebno je promeniti i consumera, koji ocekuje String, a dobija JSON, pa se zato javalja exception: `Cannot convert from [com.jovana.kafka.payload.Student] to [java.lang.String]`

```java
@KafkaListener(topics = "jovana", groupId = "myGroup")
    public void consumeJsonMsg(Student student){
        log.info(format("Consuming the message from jovana Topic:: %s", student.toString()));

    }
```

kada pozovemo u postmanu:

![image-20240128220418429](C:\Users\Jovana\AppData\Roaming\Typora\typora-user-images\image-20240128220418429.png)

videcemo u consoli da sve radi:

![image-20240128220503570](C:\Users\Jovana\AppData\Roaming\Typora\typora-user-images\image-20240128220503570.png)

# Wikimedia Kafka Streams example

U ovoj app cemo imate dve app koje ce predstavljati producera i consumera. Medjusobno ce komunicirati uz pomoc web clienta.

## Producer module:

pravimo config klasu gde pravimo topic:

```java
@Configuration
public class WikimediaTopicConfig {

    @Bean
    public NewTopic wikimediaStreamTopic(){
        return TopicBuilder
                .name("wikimedia-stream")
                .build();
    }
}
```

i WebClient config klasu gde pravimo bean web clienta uz pomoc kojeg cemo dobiti sve wikimedia streamove

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient.Builder webClientBuilder(){
        return WebClient.builder();
    }
}
```

