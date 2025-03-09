+++
title = 'Protobuf - Mutability'
summary = 'A brief examination of Protobuf immutability and ways to work around it'
tags = ["protobuf"]
date = 2025-03-09
showToc = true
draft = false
+++

## Introduction

Protocol buffers (aka protobuf) is an efficient binary protocol for defining and exchanging structured data across languages and platforms. However, Protobuf is immutable. In this article, we discuss ways to support mutability when working with Protobuf data.

## Immutability

Google, the original developers of Protobuf, made a design decision to enforce immutability of Protobuf messages. Google encountered a lot of subtle and not-so-subtle errors when they allowed Protobuf data to be modified as it transited through various systems. In fact, the earliest internal implementations of the Protocol specification in Java allowed the data to be mutated [1]. They felt the safety guarantees from making the data immutable outweighed the convenience of easily changing field values. 

Let's briefly examine some of the problems that can arise when the data is mutable [2].

### Aliasing bugs

Martin Fowler describes a common bug that can arise when multiple references point to the same object in memory [3]. A change to the object's field by any reference will be reflected in all references since they point to the same object. Sometimes this is intentional and desired. Other times, this is unintentional since the programmer may have actually intended to create a **deep copy** of the object so that they can mutate the copy without changing the original.

To avoid the **aliasing bug**, programmers should take care to deep copy the data when they're interested in the value(s) of the object and not the object itself. The obvious trade-off here is the extra CPU and memory required to assemble the copy.

### Uniqueness in collections

Collections such as Java's HashMap or HashSet rely on immutable keys to ensure that inserted values can later be found. In the case of the HashSet, a hash code is calculated from the object. The hash code serves as the key to access the value in the underlying array. If we change the object and the change causes the hashcode to differ, then we will not be able to find the object in the hashset. The same applies for the hashmap.

This does not mean the values have to be strictly immutable. It depends on the semantics of the domain object. It's possible the hash code is calculated on a subset of object properties that are immutable. However, it is generally safer to compute a hash code that takes into account all of the attributes of the object. 

## Mutating Protobuf in code

We discussed Protobuf immutability and some of the issues it tries to avoid. The natural question is - why would we want mutability? The simple answer is that data rarely remains static as it flows through a data pipeline. We may want to enrich the data with additional metadata, standardize certain fields, or anonymize/scrub personally identifiable information (PII) to remain compliant with regulations. Whatever the use case is, it's clear that mutability can be desirable if done carefully.

Here's a couple ways we can achieve mutability.

### Use Proto builders

While the Protobuf message itself may be immutable, Protobuf provides a `toBuilder()` method in the Message interface [4]. All generated code for your custom Protobuf message will implement the Message interface and therefore have access to the `toBuilder()` method. This method provides a Builder that is pre-loaded with the (nested) values of the given message. The Builder provides methods for getting, setting, and clearing values from any field or nested struct. 

Let's examine the following Proto schema:

```
syntax = "proto2";

message Person {
    optional string first_name = 1;
    optional string last_name = 2;

    optional Address address = 3;
}

message Address {
    optional int32 house_number = 1;
    optional string street = 2;
    optional string city = 3;
}
```

Here's pseudocode for mutating an incoming Proto message.

```Kotlin
val bytes: ByteArray = FileUtils.readFileToByteArray(myFile)
val person: Person = Person.parseFrom(bytes) // immutable

val personBuilder = person.toBuilder()
personBuilder.setFirstName(person.firstName.uppercase())
personalBuilder.getAddressBuilder().setStreet(person.address.street.uppercase())

val uppercasedPerson = personBuilder.build()
```

The example is trivial, but it illustrates how one can mutate a Proto message by creating a builder. It's clear from the code that we're modifying a builder and not the original object. Once we've completed processing, we can generate the final Proto message by invoking the builder's `build()` method.

The downside is that we've created up to 2 additional objects that we have to store in memory - the builder and the modified message. If our application processes the incoming messages as a stream, this may not be too bad. If we process a big batch of messages, then our application can run out of memory if we're not careful.

Additionally, the builder does not offer an easy way to serialize itself. This doesn't matter if the application is contained within a single JVM, but if we're using distributed frameworks like Flink, then the object needs to be serializable in order for the framework to transfer it from node to node. If this applies, refer to the next section for a suggested approach.

### Use a POJO

The MessageBuilder interface is not serializable by default. It does not provide a convenient method for converting the builder to bytes and vice-versa. You could write a custom serializer that converts it to a Proto message and then to bytes. However, if the program performs multiple processing steps on different nodes, that conversion cost starts to add up for each additional serialization and deserialization that we have to do.

A simpler approach is to convert the Message into a native data structure that the framework knows how to serialize. In the case of Flink, it supports various built-in types like Flink rows or POJOs [5]. Thus, we can deserialize a message into a POJO in our source function, carry the POJO from stage to stage, and then perform a final step to convert it back to a complete message in the sink function.

We have not avoided the cost of converting the Proto message to a POJO (or equivalent in other, non-JVM languages), but we've simplified the intra-service business logic. Now it's trivial to mutate the POJO. Furthermore, we can carefully control access to which fields can or can't be mutated in our POJO unlike Proto or its builder.

```Kotlin
data class PersonPojo(
    companion object (
        fun fromProto(person: Person): { ... }
        fun toProto(personPojo: PersonPojo) { ... }
    )

    var firstName: String? = null, // mutable
    val lastName: String? = null, // immutable
    val address: Address? = null, // immutable
)

data class Address(
    val houseNumber: Int? = null, // immutable
    var street: String? = null, // mutable
    val city: String? = null // immutable
)
```

And the corresponding pseudo-code for a hypothetical streaming framework:

```
val byteStream: Stream<ByteArray> = HypotheticalFramework.readByteStream(myInputStream)
val personStream: Stream<Person> = byteStream.apply(PersonPojoDeserializer())

val personPojoStream: Stream<PersonPojo> = personStream.apply(PersonPojo::toPojo)
val intermediateResult: Stream<PersonPojo> = personPojoStream.apply(SomeProcessing())
val finalResult: Stream<PersonPojo> = intermediateResult.apply(SomeMoreProcessing())

finalResult.sinkTo(MySinkFunction())
```

In this example, the processing steps can happen on different nodes (and therefore different JVMs) so the stream elements need to be serializable by the framework.

## Conclusion

We learned that Proto is immutable by default to avoid common bugs. While Proto does offer a builder to mutate properties, it may be inefficient and require additional code for distributed frameworks. 

Alternatively, we can convert the Proto to a POJO from the wire. Within the business logic of the app, we work with the POJO. Once we've finished processing the message, we simply convert it back to a Proto message before writing it back to the wire.

## References

[1] https://groups.google.com/g/protobuf/c/Fpl5EHHpLIM

[2] Yevsyukov, A., & Dashenkov, D. (2025, March 9th). _Protobuf — Serialization and Beyond. Part 2: Immutability_. Medium. https://blog.teamdev.com/protobuf-immutability-3d7b4995712c

[3] Fowler, M. (2025, March 9th). _Aliasing Bug_. Martin Fowler Blog. https://martinfowler.com/bliki/AliasingBug.html

[4] https://protobuf.dev/reference/java/java-generated/#builders

[5] https://nightlies.apache.org/flink/flink-docs-release-1.20/docs/dev/datastream/fault-tolerance/serialization/types_serialization/#supported-data-types
