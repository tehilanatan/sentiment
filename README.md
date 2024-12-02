# sentiment

### stanford sentiment analysis
```
		<dependency>
			<groupId>edu.stanford.nlp</groupId>
			<artifactId>stanford-corenlp</artifactId>
			<version>3.8.0</version>
		</dependency>

		<dependency>
			<groupId>edu.stanford.nlp</groupId>
			<artifactId>stanford-corenlp</artifactId>
			<version>3.8.0</version>
			<classifier>models</classifier>
		</dependency>
```
apply patch stanford
commit - with nlp
### twitter
apply patch - news-stream
<br>
controller/AppController.java
```java
    @Autowired
    AppNewsStream twitterStream;

    @RequestMapping(path = "/startTwitter", method = RequestMethod.GET)
    public  @ResponseBody Flux<String> start(String text)  {
        return twitterStream.filter(text)
                .window(Duration.ofSeconds(3))
                .flatMap(window->toArrayList(window))
                .map(messages->{
                    if (messages.size() == 0) return "size: 0 <br>";
                    return "size: " + messages.size() + "<br>";
                });
    }

    @RequestMapping(path = "/stopTwitter", method = RequestMethod.GET)
    public  @ResponseBody Mono<String> stop()  {
        twitterStream.shutdown();
        return Mono.just("shutdown");
    }

    public static <T> Mono<ArrayList<T>> toArrayList(Flux<T> source) {
        return  source.reduce(new ArrayList(), (a, b) -> { a.add(b);return a; });
    }
```
commit - with twitter

###kafka
pom.xml
```java
		<dependency>
			<groupId>io.projectreactor.kafka</groupId>
			<artifactId>reactor-kafka</artifactId>
			<version>1.3.10</version>
		</dependency>
		<dependency>
			<groupId>org.springframework.kafka</groupId>
			<artifactId>spring-kafka</artifactId>
		</dependency>
```
application.properties
```
spring.kafka.bootstrap-servers=kafka:9092
spring.kafka.producer.retries=0
spring.kafka.producer.acks=1
spring.kafka.producer.batch-size=16384
spring.kafka.producer.properties.linger.ms=0
spring.kafka.producer.buffer-memory = 33554432
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
spring.kafka.consumer.properties.group.id=searchengine
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.consumer.enable-auto-commit=true
spring.kafka.consumer.auto-commit-interval=1000
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.properties.session.timeout.ms=120000
spring.kafka.consumer.properties.request.timeout.ms=180000
spring.kafka.listener.missing-topics-fatal=false
```
apply patch -> kafka.patch
<br>
controller/AppController.java
```java
    @Autowired
    AppKafkaSender kafkaSender;

    @Autowired
    KafkaReceiver<String,String> kafkaReceiver;


    @RequestMapping(path = "/sendKafka", method = RequestMethod.GET)
    public  @ResponseBody Mono<String> sendText(String text)  {
        kafkaSender.send(text, APP_TOPIC);
        return Mono.just("OK");
    }

    @RequestMapping(path = "/getKafka", method = RequestMethod.GET)
    public  @ResponseBody  Flux<String> getKafka()  {
        return kafkaReceiver.receive().map(x-> x.value() + "<br>");
    }
```
try:
<br>
http://localhost:8080/sendKafka?text=test1
<br>
http://localhost:8080/getKafka
<br>
commit - with kafka
### all together
controller/AppController.java
```java

    @RequestMapping(path = "/grouped", method = RequestMethod.GET)
    public  @ResponseBody Flux<String> grouped(@RequestParam(defaultValue = "obama") String text,
                                                     @RequestParam(defaultValue = "3") Integer timeWindowSec) throws TwitterException {
        var flux = kafkaReceiver.receive().map(message -> message.value());
        twitterStream.filter(text).map((x)-> kafkaSender.send(x, APP_TOPIC)).subscribe();

            return flux.map(x-> new TimeAndMessage(DateTime.now(), x))
                    .window(Duration.ofSeconds(timeWindowSec))
                    .flatMap(window->toArrayList(window))
                    .map(y->{
                        if (y.size() == 0) return "size: 0 <br>";
                        return  "time:" + y.get(0).curTime +  " size: " + y.size() + "<br>";
                    });
    }

    @RequestMapping(path = "/sentiment", method = RequestMethod.GET)
    public  @ResponseBody Flux<String> sentiment(@RequestParam(defaultValue = "obama") String text,
                                               @RequestParam(defaultValue = "3") Integer timeWindowSec) throws TwitterException {
        var flux = kafkaReceiver.receive().map(message -> message.value());
        twitterStream.filter(text).map((x)-> kafkaSender.send(x, APP_TOPIC)).subscribe();

        return flux.map(x-> new TimeAndMessage(DateTime.now(), x))
                .window(Duration.ofSeconds(timeWindowSec))
                .flatMap(window->toArrayList(window))
                .map(items->{
                    if (items.size() > 10) return "size:" + items.size() + "<br>";
                    System.out.println("size:" + items.size());
                    double avg = items.stream().map(x-> sentimentAnalyzer.analyze(x.message))
                            .mapToDouble(y->y).average().orElse(0.0);
                    if (items.size() == 0) return "EMPTY<br>";
                    return   items.size() + " messages, sentiment = " + avg +  "<br>";

                });
    }
    static class TimeAndMessage {
        SimpleDateFormat formatter = new SimpleDateFormat("yyyy-MM-dd, HH:mm:ss, z");
        DateTime curTime;
        String message;

        public TimeAndMessage(DateTime curTime, String message) {
            this.curTime = curTime;
            this.message = message;
        }

        @Override
        public String toString() {
            return "TimeAndMessage{" +
                    "formatter=" + formatter +
                    ", curTime=" + curTime +
                    ", message='" + message + '\'' +
                    '}';
        }
    }

```
commit - all together
