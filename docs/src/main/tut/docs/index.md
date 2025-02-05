---
layout: docs
title: Getting Started
---

# Getting dependency

To use kafka4s in an existing SBT project with Scala 2.12 or a later version, add the following dependencies to your
`build.sbt` depending on your needs:

```scala
libraryDependencies ++= Seq(
  "com.banno" %% "kafka4s" % "<version>"
)
```

# Some quick examples

First, some initial imports:
```tut
import cats._, cats.effect._, cats.implicits._, scala.concurrent.duration._
```

### Define our data

We'll define a toy message type for data we want to store in our Kafka topic.

```tut
case class Customer(name: String, address: String)
case class CustomerId(id: String)
```

### Create our Kafka topic

Now we'll tell Kafka to create a topic that we'll write our Kafka records to.

First, let's bring some types and implicits into scope:

```tut
import com.banno.kafka._, com.banno.kafka.admin._
import org.apache.kafka.clients.admin.NewTopic
```

Now we can create a topic named `customers.v1` with 1 partition and 3 replicas:

```tut
val topic = new NewTopic("customers.v1", 1, 1)
val kafkaBootstrapServers = "localhost:9092" // Change as needed
AdminApi.createTopicsIdempotent[IO](kafkaBootstrapServers, topic :: Nil).unsafeRunSync
```

### Register our topic schema 

Let's register a schema for our topic with the schema registry!

First, we bring types and implicits into scope:

```tut
import com.banno.kafka.schemaregistry._
```
Now we initialize a schema registry client:

```tut
val schemaRegistryUri = "http://localhost:8081" // Change as needed
val cachedSchemasPerSubject = 1000
val schemaRegistry = SchemaRegistryApi[IO](schemaRegistryUri, cachedSchemasPerSubject).unsafeRunSync
```

We'll use the name of the topic we created above:

```tut
val topicName = topic.name
```

Now we can register our topic key and topic value schemas:


```tut
(for {
  _ <- schemaRegistry.registerKey[CustomerId](topicName)
  _ <- schemaRegistry.registerValue[Customer](topicName)
} yield ()).unsafeRunSync
```

### Write our records to Kafka

Now let's create a producer and send some records to our Kafka topic!

We first bring our Kafka producer utils into scope:

```tut
import com.banno.kafka.producer._
```

Now we can create our producer instance:

```tut
val producer = ProducerApi.generic[IO](
  BootstrapServers(kafkaBootstrapServers),
  SchemaRegistryUrl(schemaRegistryUri),
  ClientId("producer-example")
).unsafeRunSync
```

And we'll define some customer records to be written:

```tut
import org.apache.kafka.clients.producer.ProducerRecord
val recordsToBeWritten = (1 to 10).map(a => new ProducerRecord(topicName, CustomerId(a.toString), Customer(s"name-${a}", s"address-${a}"))).toVector
```

And now we can (attempt to) write our records to Kafka:

```tut:fail
recordsToBeWritten.traverse_(producer.sendSync)
```

The above fails to compile, however! Our producer writes generic
`ProducerRecord`s, but we'd like to send typed records, to ensure that
our `CustomerId` key and our `Customer` value are compatible with our
topic. For this, we can use Kafka4s' `avro4s` integration!

#### Writing typed records with an Avro4s producer

Turning a generic producer into a typed producer is as simple as the following:

```tut
val avro4sProducer = producer.toAvro4s[CustomerId, Customer]
```

We can now write our typed customer records successfully!

```tut
recordsToBeWritten.traverse_(r => avro4sProducer.sendSync(r).flatMap(rmd => IO(println(s"Wrote record to ${rmd}")))).unsafeRunSync
```

### Read our records from Kafka

Now that we've stored some records in Kafka, let's read them as an `fs2.Stream`!

We first import our Kafka consumer utilities:
```tut
import com.banno.kafka.consumer._
```

Now we can create our consumer instance.

We'll create a "shifting" Avro4s consumer, which will shift its blocking calls to a dedicated `ExecutionContext`, to avoid blocking the main work pool's (typically `ExecutionContext.global`) threads. After receiving records, work is then shifted back to the work pool. We'll want an implicit `ContextShift` instance in scope to manage this thread shifting for us.

Here's our `ContextShift`:

```tut
import scala.concurrent.ExecutionContext
implicit val CS = IO.contextShift(ExecutionContext.global)
```

And here's our consumer, along with the `ExecutionContext` we'll want our consumer to use:

```tut
val blockingContext = ExecutionContext.fromExecutorService(java.util.concurrent.Executors.newFixedThreadPool(1)) 
val consumer = ConsumerApi.avro4sShifting[IO, CustomerId, Customer](
  blockingContext,
  BootstrapServers(kafkaBootstrapServers), 
  SchemaRegistryUrl(schemaRegistryUri),
  ClientId("consumer-example"),
  GroupId("consumer-example-group")
).unsafeRunSync
```

With our Kafka consumer in hand, we'll assign to our consumer to our topic partition, with no offsets, so that it starts reading from the first record.
```tut
import org.apache.kafka.common.TopicPartition
val initialOffsets = Map.empty[TopicPartition, Long] // Start from beginning
consumer.assign(topicName, initialOffsets).unsafeRunSync
```

And we can now read a stream of records from our Kafka topic:

```tut
val messages = consumer.rawRecordStream(1.second).take(5).compile.toVector.unsafeRunSync
```

To clean up after ourselves, we'll close and shut down our resources:
```tut
consumer.close.unsafeRunSync
producer.close.unsafeRunSync
blockingContext.shutdown
```

Note that kafka4s provides constructors that return `cats.effect.Resource`s for the above resources, so that their shutdown steps are guaranteed to run after use. We're manually shutting down resources simply for the sake of example.

Now that we've seen a quick overview, we can take a look at more in-depth documentation of Kafka4s utilities.
