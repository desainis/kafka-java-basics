# Kafka Basics with Java

## Starting Kafka with Java (Maven)
- Add the Kafka dependencies to your `pom.xml` as below:
  - Fetch the latest at [mvnrepository](https://mvnrepository.com/artifact/org.apache.kafka/kafka-clients)

```xml
<!-- https://mvnrepository.com/artifact/org.apache.kafka/kafka-clients -->
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>2.4.0</version>
</dependency>

```

- This project also makes use of slf4j. Fetch the latest at [mvnrepository](https://mvnrepository.com/artifact/org.slf4j/slf4j-simple)

## Kafka Producers
- Writing a basic producer in Java. See `ProducerDemo.java` for further details. 

```java
String bootstrapServers = "localhost:9092";

// create Producer properties
Properties properties = new Properties();
properties.setProperty(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
properties.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
properties.setProperty(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

// create the producer
KafkaProducer<String, String> producer = new KafkaProducer<String, String>(properties);
ProducerRecord<String, String> record = new ProducerRecord<String, String>("first_topic", "hello world");

// send the data - asynchronous
producer.send(record);

// flush data
producer.flush();

// flush and close producer
producer.close();
```

#### Kafka Producers with Callback
- Send data with a callback function. See `ProducerWithCallback.java` for further details. 
```java
 for (int i=0; i<10; i++) {
            ProducerRecord<String, String> record = new ProducerRecord<String, String>("first_topic", "hello world " + Integer.toString(i));

            // send the data - asynchronous
            producer.send(record, new Callback() {
                public void onCompletion(RecordMetadata recordMetadata, Exception e) {
                    if (e == null) {
                        // record was successfully sent
                        logger.info("Received new metadata. \n" +
                                "Topic: " + recordMetadata.topic() + "\n" +
                                "Partition: " + recordMetadata.partition() + "\n" +
                                "Offset: " + recordMetadata.offset() + "\n" +
                                "Timestamp: " + recordMetadata.timestamp());
                    } else {
                        logger.error("Error while producing", e);
                    }
                }
            });
        }
```

#### Kafka Producers with Keys
- A producer with keys value pairs. By providing a key we guarantee that the same key goes to the same partition. 
See `ProducerDemoKeys.java` for further details. 

```java
for (int i=0; i<10; i++) {
            String topic = "first_topic";
            String value = "hello world " + Integer.toString(i);
            String key = "id_" + Integer.toString(i);

            logger.info("Key: " + key); // log the key
            // By providing a key we guarantee that the same key goes to the same partition

            ProducerRecord<String, String> record = new ProducerRecord<String, String>(topic, key, value);

            // send the data - asynchronous
            producer.send(record, new Callback() {
                public void onCompletion(RecordMetadata recordMetadata, Exception e) {
                    if (e == null) {
                        // record was successfully sent
                        logger.info("Received new metadata. \n" +
                                "Topic: " + recordMetadata.topic() + "\n" +
                                "Partition: " + recordMetadata.partition() + "\n" +
                                "Offset: " + recordMetadata.offset() + "\n" +
                                "Timestamp: " + recordMetadata.timestamp());
                    } else {
                        logger.error("Error while producing", e);
                    }
                }
            }).get(); // Bad practice, we just made the call synchronous.
        }
```

## Kafka Consumers
- A simple Kafka consumer. 
```java
Logger logger = LoggerFactory.getLogger(ConsumerDemo.class.getName());

String bootStrapServers = "localhost:9092";
String groupId = "my-fourth-application";
String topic = "first_topic";

// create consumer configs
Properties properties = new Properties();
properties.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootStrapServers);
properties.setProperty(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
properties.setProperty(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
properties.setProperty(ConsumerConfig.GROUP_ID_CONFIG, groupId);
properties.setProperty(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest"); // earliest/latest/none

// create consumer
KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(properties);

// subscribe consumer to our topic(s)
consumer.subscribe(Collections.singleton(topic));
// consumer.subscribe(Arrays.asList("first_topic", "second_topic")) to subscribe to multiple topics

// poll for new data
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

    for (ConsumerRecord<String, String> record : records) {
        logger.info("Key: " + record.key() + ", Value: "  + record.value());
        logger.info("Partition: " + record.partition() + ", Offset: " + record.offset());
    }
}
```

#### Kafka Consumers with Groups
- Assign a group to a Kafka consumer. 
```java
String bootStrapServers = "localhost:9092";
String groupId = "my-fourth-application";
String topic = "first_topic";

// create consumer configs
Properties properties = new Properties();
properties.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootStrapServers);
properties.setProperty(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
properties.setProperty(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
properties.setProperty(ConsumerConfig.GROUP_ID_CONFIG, groupId);
```

#### Kafka Consumers with Thread(s)
- If you're not familiar with Threads, this may be a little off putting. In general you want a threaded solution in production.
```java
public static void main(String[] args) {
        new ConsumerDemoWithThread().run();
    }

    private ConsumerDemoWithThread() {

    }

    private void run() {
        Logger logger = LoggerFactory.getLogger(ConsumerDemoWithThread.class.getName());

        String bootStrapServers = "localhost:9092";
        String groupId = "my-sixth-application";
        String topic = "first_topic";

        // latch for dealing with concurrency
        CountDownLatch latch = new CountDownLatch(1);

        // create the consumer runnable
        logger.info("Creating the consumer thread");
        Runnable myConsumerRunnable = new ConsumerRunnable(latch,
                bootStrapServers,
                groupId,
                topic);

        // Start the thread
        Thread myThread = new Thread(myConsumerRunnable);
        myThread.start();

        // add a shutdown hook
        Runtime.getRuntime().addShutdownHook(new Thread( () -> {
            logger.info("Caught shutdown hook");
            ((ConsumerRunnable) myConsumerRunnable).shutdown();
            try {
                latch.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            logger.info("Application has exited");
        }));

        try {
            latch.await();
        } catch (InterruptedException e) {
            logger.error("Application got interrupted", e);
        } finally {
            logger.info("Application is closing");
        }

    }

    public class ConsumerRunnable implements Runnable {

        private CountDownLatch latch;
        private KafkaConsumer<String, String> consumer;
        private Logger logger = LoggerFactory.getLogger(ConsumerRunnable.class.getName());

        public ConsumerRunnable(CountDownLatch latch,
                              String bootStrapServers,
                              String groupId,
                              String topic) {
            this.latch = latch;

            // create consumer configs
            Properties properties = new Properties();
            properties.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootStrapServers);
            properties.setProperty(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
            properties.setProperty(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
            properties.setProperty(ConsumerConfig.GROUP_ID_CONFIG, groupId);
            properties.setProperty(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest"); // earliest/latest/none

            // create consumer
            KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(properties);
        }

        @Override
        public void run() {
            try {
                while (true) {
                    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

                    for (ConsumerRecord<String, String> record : records) {
                        logger.info("Key: " + record.key() + ", Value: " + record.value());
                        logger.info("Partition: " + record.partition() + ", Offset: " + record.offset());
                    }
                }
            } catch (WakeupException e) {
                logger.info("Received shutdown signal!");
            } finally {
                consumer.close();

                // tell our main code we're done with the consumer
                latch.countDown();
            }
        }

        public void shutdown() {
            consumer.wakeup(); // special method to interrupt consumer.poll()
        }
    }
```

#### Kafka Consumers using Assign and Seek
- Use assign and seek carefully. Generally used to replay messages or fetch a specific message. 
```java
// create consumer
    KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(properties);

    // assign and seek are used to replay data (or fetch a specific message)

    // assign
    TopicPartition partitionToReadFrom = new TopicPartition(topic, 0);
    long offsetToReadFrom = 15L;
    consumer.assign(Arrays.asList(partitionToReadFrom));

    // seek
    consumer.seek(partitionToReadFrom, offsetToReadFrom);

    int numberOfMessagesToRead = 5;
    boolean keepOnReading = true;
    int numberOfMessagesReadSoFar = 0;

    // poll for new data
    while (keepOnReading) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

        for (ConsumerRecord<String, String> record : records) {
            numberOfMessagesReadSoFar += 1;
            logger.info("Key: " + record.key() + ", Value: "  + record.value());
            logger.info("Partition: " + record.partition() + ", Offset: " + record.offset());
            if (numberOfMessagesReadSoFar >= numberOfMessagesToRead) {
                keepOnReading = false;
                break;
            }
        }
    }
}
```

#### Demo: Using Kafka to consume live tweets
- See `TwitterProducer.java` for further details
```java
        logger.info("Setup");

        /** Set up your blocking queues: Be sure to size these properly based on expected TPS of your stream */
        BlockingQueue<String> msgQueue = new LinkedBlockingQueue<String>(1000);

        // create a twitter client
        Client client = createTwitterClient(msgQueue);
        // Attempts to establish a connection.
        client.connect();

        // create a kafka producer
        KafkaProducer<String, String> producer = createKafkaProducer();

        // loop to send tweets to kafka
        // on a different thread, or multiple different threads....
        while (!client.isDone()) {
            String msg = null;
            try {
                msg = msgQueue.poll(5, TimeUnit.SECONDS);
            } catch (InterruptedException e) {
                e.printStackTrace();
                client.stop();
            }
            if (msg != null) {
                logger.info(msg);
                producer.send(new ProducerRecord<>("twitter_tweets", null, msg), new Callback() {
                    @Override
                    public void onCompletion(RecordMetadata recordMetadata, Exception e) {
                        if (e != null) {
                            logger.error("Something bad happened", e);
                        }
                    }
                });
            }
        }
        logger.info("End of application");
```
![Java Consumer and Producer Demo](./media/java-tweet-demo.gif)


#### Credits to Stephane. Checkout his awesome course on [Udemy](https://www.udemy.com/course/apache-kafka)!