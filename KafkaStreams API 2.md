Pokrenuli smo docker-compose up, a zatim smo pokrenuli applikaciju.

zatim cemo u terminalu da instanciramo producera i consumera, a to radimo tako sto **logujemo u kontejner**:

```cmd
docker exec -it [container-name] bash
```

ovako izgleda potvrda da smo logovani u kontejner:

```cmd
[appuser@d62875c1e8b6 ~]$
```

Komanda za pravljenje poruka u kafka topik:

```cmd
kafka-console-producer --broker-list localhost:9092 --topic greetings
```

Otvaramo novi command line i opet se logujemo u kontejner i onda pravimo konsumera koji ce da konsumuje poruke sa greetings-uppercase kafka topika:

```cmd
kafka-console-consumer --bootstrap-server localhost:9092 --topic greetings_uppercase
```

znaci producer publisuje poruke na jedan topic 'greetings', a consumer konsumuje poruke sa 'greetings-uppercase' i dobija te poruke samo u upper case.

# Operacije u Kafka Streams koristeci KStream API

## filter

- Da uklonimo elemente kafka streama koji ne zadovoljavaju kriterijum.
- Ovaj operator uzima **Predikat** funkcionalni interfejs kao input i primenjuje taj predikat na input (data kafka topika)

```java
KStream<String, String> modifiedStream = greetingsStream
                .filter((key, value) -> value.length() > 5)
                .mapValues((readOnlyKey, value) -> value.toUpperCase());
```

## filterNot

```java
KStream<String, String> modifiedStream = greetingsStream
                .filterNot((key, value) -> value.length() > 5)
                .mapValues((readOnlyKey, value) -> value.toUpperCase());
```

## map

- Kada treba da transformisemo key i value u nesto drugo, neku drugu formu.
- Kada koristimo mapu, moramo da vratimo i key i values zato koristimo KeyValue.pair
- Moramo da instanciramo producera koji ce da publisuje na topik sa key i value

```cmd
kafka-console-producer --broker-list localhost:9092 --topic greetings --property "key.separator=-" --property "parse.key=true"
```

- I onda objavljujemo poruku tako sto ukucamo npr:

  ```cmd
  gm-goodmorning!
  ```

## mapValues

- Kada zelimo da transformisemo samo vrednosti u Kafka Streamu

```java
 KStream<String, String> modifiedStream = greetingsStream
                .filter((key, value) -> value.length() > 5)
                .mapValues((readOnlyKey, value) -> value.toUpperCase());
```

## flatMap

- Koristi se kada jedan even kreira vise eventova. 
- Na primer imamo string Apple (event) i hocemo da ga podelimo na zasebne karaktere i da ih posaljemo dalje kao individualne eventove.
- U realnosti to je rest poziv za svaki event koji vraca listu kao response, i onda te eventove iz liste da posaljemo kao posebne eventove u streamu. 

```java
KStream<String, String> modifiedStream = greetingsStream
                .flatMap((key, value) -> {
                    var newValues = Arrays.asList(value.split(""));
                    
                    return newValues
                            .stream()
                            .map(val -> KeyValue.pair(key.toUpperCase(), val))
                            .collect(Collectors.toList());
                });
```

- **Kako konzumovati sa kljucem**:

- ```cmd
  kafka-console-consumer --bootstrap-server localhost:9092 --topic greetings_uppercase --from-beginning -property "key.separator= - " --property "print.key=true"
  ```

- input:

  ```cmd
  t-taca
  ```

- output:

  ```cmd
  T - t
  T - a
  T - c
  T - a
  ```

  

## flatMapValues

- Slicno kao flatMap, sem sto ovde pristupamo i menjamo samo vrednost. Kljuc je ovde samo dummy, ignorise se

```java
KStream<String, String> modifiedStream = greetingsStream
                .flatMapValues((key, value) -> {
                    var newValues = Arrays.asList(value.split(""));
                    
                    return newValues
                            .stream()
                            .map(String::toUpperCase)
                            .collect(Collectors.toList());
                });
```

- input: 

  ```cmd
  e-ejej
  ```

- output:

  ```cmd
  e - E
  e - J
  e - E
  e - J
  ```

## peek

- Zgodanj e kada treba da debugujemo ceo stream processing logiku
- Omogucava da znamo koji je element koji se salje posle trenutnog. I mozemo da odstampamo vrednost elemenata koji se salju i koji se ignorisu. 
- Ako se ignorisu, nece ni biti deo peek operacije
- peek nece uticati na key i value uopste

```java
KStream<String, String> modifiedStream = greetingsStream
                .filter((key, value) -> value.length() > 5)
                .peek((key, value) -> {
                    log.info("after filter: key : {}, value {} ", key, value);
                })
                .mapValues((readOnlyKey, value) -> value.toUpperCase())
                .peek((key, value) -> {
                    log.info("after mapValues: key : {}, value {} ", key, value);
                })
                .flatMapValues((key, value) -> {
                    var newValues = Arrays.asList(value.split(""));
                    
                    return newValues
                            .stream()
                            .map(String::toUpperCase)
                            .collect(Collectors.toList());
                });
```

## merge

- Kombinuje dva nezavisna Kafka Streama u jedan stream

```java
var greetingsStream = streamsBuilder
                .stream(GREETINGS, Consumed.with(Serdes.String(), Serdes.String()));
        
var greetingsSpanishStream = streamsBuilder
    .stream(GREETINGS_SPANISH, Consumed.with(Serdes.String(), Serdes.String()));

var mergeStream = greetingsStream.merge(greetingsSpanishStream); // merge dva streama

var modifiedStream = mergeStream
                .mapValues((readOnlyKey, value) -> value.toUpperCase());
```

- Takodje smo dodali GREETINGS_SPANISH topic u createTopic metodu:

```java
 createTopics(properties, List.of(GREETINGS_SPANISH,
                                         GREETINGS,
                                         GREETINGS_UPPERCASE));
```

- Otvaramo jos jedan cmd, i kreiramo publishera koji ce da publisuje na greetings_spanish topik:

  ```cmd
  kafka-console-producer --broker-list localhost:9092 --topic greetings_spanish --property "key.separator=-" --property "parse.key=true"
  ```

- Znaci imamo 2 producera i jednog consumera, koji prihvata eventove sa oba topika GREETINGS i GREETINGS_SPANISH

# Serdes

- Serdes je factory klasa u Kafka Streams koji hendluje serijalizaciju i deserijalizaciju kljuca i vrednosti.
- Kod Consumera je to **deserijalizacija**, a kod Producera je **serijalizacija**
- Kafka Streams to sve radi u pozadini i to postize sa Consumed.with i Produced.with, u pozadini je glavni Serde interfejs

## Defaultna deserijalizacija i serijalizacija koristeci Application Configuration

- U slucaju da smo sigurni da ce cela aplikacija da koristi jedan tip, onda ovo koristimo

```java
properties.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.StringSerde.class);
properties.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.StringSerde.class);
```

```java
KStream<String, String> greetingsSpanishStream = streamsBuilder
        .stream(GREETINGS_SPANISH);
```

```java
modifiedStream.to(GREETINGS_UPPERCASE);
```

# Build custom Serdes

- Zelimo da objavimo ovaj event u JSON formatu:

  ```json
  {
      "message" : "Good Morning",
      "timestamp" : "2022-12-04T05:26:31.060293"
  }
  ```

- Sta nam treba:

  1. Serializer
  2. Deserializer
  3. **Serde** koji ce da sadrzi Serializer i Deserializer
