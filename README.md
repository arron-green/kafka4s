# kafka4s - Functional programming with Kafka and Scala [![Build Status](https://travis-ci.com/banno/kafka4s.svg?branch=master)](https://travis-ci.com/banno/kafka4s) [![Maven Central](https://maven-badges.herokuapp.com/maven-central/com.banno/kafka4s_2.12/badge.svg)](https://maven-badges.herokuapp.com/maven-central/com.banno/kafka4s_2.12) ![Code of Conduct](https://img.shields.io/badge/Code%20of%20Conduct-Scala-blue.svg)

kafka4s provides pure, referentially transparent functions for working with Kafka, and integrates with FP libraries such as [cats-effect](https://typelevel.org/cats-effect) and [fs2](https://fs2.io).

## [Head on over to the microsite](https://banno.github.io/kafka4s)

## Quick Start

To use kafka4s in an existing SBT project with Scala 2.12 or a later version, add the following dependencies to your
`build.sbt` depending on your needs:

```scala
libraryDependencies ++= Seq(
  "com.banno" %% "kafka4s" % "<version>"
)
```

Sending records to Kafka is an effect. If we wanted to periodically write random integers to a Kafka topic, we could do:

```scala
Stream
  .awakeDelay[F](1 second)
  .evalMap(
    _ =>
      Sync[F]
        .delay(Random.nextInt())
        .flatMap(i => producer.sendAndForget(new ProducerRecord(topic, i, i)))
  )
```

Polling Kafka for records is also an effect, and we can obtain a stream of records from a topic. We can print the even random integers from the above topic using:

```scala
consumer.recordStream(
    initialize = consumer.subscribe(topic),
    pollTimeout = 1.second
  )
  .map(_.value)
  .filter(_ % 2 == 0)
  .evalMap(i => Sync[F].delay(println(i)))
```

## Learning more

To learn more about kafka4s, start with our [Getting Started Guide](/kafka4s/docs/), play with some [example apps](https://github.com/Banno/kafka4s/tree/master/examples/src/main/scala), and check out the [kafka4s Scaladoc](https://www.javadoc.io/doc/com.banno/kafka4s_2.12) for more info.
