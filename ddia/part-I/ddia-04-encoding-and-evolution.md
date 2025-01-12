# Encoding and Evolution
Applications inevitably change over time. Features are added or modified as new products are launched, user requirements become more understood, or business requirements change.

*Evolvability* is buiding systems that make it easy to adapt to change. 

In most cases, a change to an application's features also requires a change to the data it stores. When a data format or schema changes, a corresponding change to application code often needs to happen. However in large applications, code changes often cannot happen instantaneously.
- For server side applications, we may perform a *rolling upgrade*(aka *staged rollout*) to avoid service downtime, thus encouraging more frequent releases and better evolvability.
- For client side applications, users may not install the update for some time.

This means that both new and old verions of code, and old and new data formats, may potntially all coexist in the system at the same time. In order for the system to continue running smoothly, we need to maintain compatibility in both directions:
- Backward Compatibility: Newer code can read data written by older code
    - Backward compatibility is normally not hard to achieve: As author of the newer code, we know the format of the older code, and so we can explicitly handle it. 
- Forward Compatibility: Older code can read data written by newer code
    - Forward compatibility can be trickier, because it requires older code  to ignore additions made by the newer code

## Formats for Encoding Data
Programs usually work with data in (at least) two different representations.
1. In memory, data is kept in objects, structs, lists, arrays, hash tables, trees and so on. These data structures are optimized for efficient access and manipulation by the CPU(typically using pointers).
2. When you want to write data to a file or send it over the network, you have to encode it as some self-contained sequence of bytes(for example a JSON Document).Since a pointer wouldn't make sense in other process, this sequence-of-bytes representation looks quite different from the data structures that are normally used in memory.

Thus we need some kind of translation between the two formats.
- *Encoding(aka serialization or marshalling):* Translation from in-memory representation to a byte sequence is called encoding
- *Decoding(aka deserialization or unmarshalling):* Translation from a byte sequence to an in-memory representaion is called 
### Language-Specific Encoding Formats
Many programming languages come with built-in support for encoding in-memory objects into byte sequences. This is very convenient because they allow in-memory objects to be saved and restored with minimal additional code.

Drawbacks of language specific formats:
- The encoding is often tied to a particular programming language, and reading the data in another language is very difficult. This precludes integration of our systems with other systems which might be using a different programming languages.
- In order to restore data in same object types, the decoding process needs to be able to instantiate arbitrary classes. This is frequently a source of security problems.
- As they are intended for quick and easy encoding of data, they often neglect the inconvenient problems of forward and backward compatibility.
- Efficiency(CPU time taken to encode and decode, and the size of the encoded structure) is also an afterthought

For these reasons it is generally a bad idea to use your language's built-in encoding for anything other than very transient purposes.
### JSON, XML and Binary Variants
Standardized encodings like JSON, XML and CSV can be written and read by many programming languages. They are widely known and widely supported. They are textual formats and somewhat human readable.

They have some subtle problems
- *Ambiguity around encoding of numbers:*
    - In XML and CSV we cannot distinguish between a number and string that happens to consist of digits
    - JSON distinguishes strings and numbers, but it doesn't distinguish integers and floating point numbers and it doesn't specify precision.
- *Lack of support for binary strings:*
    - Binary strings are sequences of bytes without character encoding
    - JSON and XML have good support for Unicode character strings(i.e.human readable text), but they don't support binary strings.
    - People get around this limitation by encoding binary data as text using Base64. But this is hacky and increases the data size by 33%.
- *Schema support:* 
    - XML and JSON have optional schema support
    - These schemas are powerful and thus quite complicated to learn and implement

*Data interchange formats:*
- JSON, XML and CSV will remain relevant as data interchange formats(i.e. for sending data from one organization to another).
- In these situations, as long as the people agree on what the format is, it often doesn't matter how pretty or efficient the format is. The difficulty of getting different organizations to agree on anything outweighs most other concerns.
#### Binary Encoding of textual formats
For data that is used only internally within the organization, there is less pressure to use a lowest comoon denominator encoding format. Instead we could choose a format that is more compact or faster to parse. The choice of format can have a big impact when you get into terabytes of data.

Binary encodings for JSON and XML have been developed, but the small space reduction they offer may not be worth the loss of human readability.
- *JSON:* MessagePack, BSON, BISON etc.
- *XML:* WBXML, Fast Infoset
### Thrift and Protocol Buffers
Apache Thrift and Protocol Buffers are binary encoding libraries. 

They both require a schema for any data that is encoded.
- *Thrift Interface Definition Language:*
        
        struct Person{
            1:required string username,
            2:optional i64 favoriteNumber
            3:optional list<string> interests
        }
- Protocol Buffer Schema Definition

        message Person{
            require string user_name = 1,
            optional int64 favorite_number = 2,
            repreated string interests = 3
        }

Thrift and Protocol buffers both use *field tags* in encoded data, instead of field names.Field tags are like aliases for fields--they are a compact way of saying what field we are talking about without having to spell out the field name.

Thrift has two encoding protocola: BinaryProtocol and CompactProtocol. CompactProtocol produces more compact encoding compared to BinaryProtocol. It does this: 
- By packing field type and tag number into a single byte as opposed to three bytes in BinaryProtocol and 
- By using variable-length integers in minimum necessary bytes, instead of fixed eight byte integers. Top bit of each byte is used to indicate if there are still more bytes to come.

Protocol Buffer has a single binary encoding format. It does bit packing slightly differently, but is otherwise very similar to Thrift's CompactProtocol.
#### Field Tags and Schema Evolution
**Field tags:**
Field tags are critical to the meaning of encoded data. 
- We can change name of encoded field in schema, but we cannot change the field tag, since that would make all existing encoded data invalid.

**Adding new fields:**
- *Forward compatibility:* We can add new field tags to the schema, provided that we give each field a new tag number. If old code tries to read data written by new code, including a new field with tag number it doesn't recognize, it can simply ignore that field. This maintains forward compatibility.
- *Backward compatibility:* As long as each field has a unique tag number, new code can always read old data, because the tag numbers still have the same meaning. Every field we add after the initial deployment of the schema must be optional or have a default value.

**Removing Fields:**
Removing a field is just like adding a field, with backward and forward compatibility concerns reversed.
- *Forward compatibility:* We can only remove a field that is optional(Required fields can never be removed), because then old code will not work for data added by new code.
- *Backward compatibility:* You can never use the same tag number again(because you may still have data written somewhere that may be using the old tag number and that field must be ignored by the new code)
#### Datatypes and Schema Evolution
Changing datatype may be possible, but there is a risk that value might lose precision or get truncated.

Protocol Buffers does not have list or array datatype, but instead has a repeated marker for fields. The same field tag simply appears multiple times in the record. This has a nice effect that it's okay to change an optional(single-vaued) field into a repeated(multi-valued) field. New code reading old data sees a list with zero or one elements(depending on whether the field was present); old code reading new data sees only the last element of the list.

Thrift has a list datatype, which is parameterized with the datatype of the list elements. This does not allow same evolution from single-valued to multi-valued fields as in Protocol Buffers does, but it has the advantage of supporting nested schema
### Avro
Apache Avro has two schema languages
- Avro IDL(Intended for human editing)

        record Person{
            string userName;
            union {null, long} favouriteNumber = null;
            array<string> interests;
        }
- JSON based

        {
            "type": "record",
            "name": "Person",
            "fields": [
                {"name":"userName", "type":"string"},
                {"name":"favoriteNumber", "type":["null", "long"], "default": null},
                {"name":"interests", "type":{"type", "array", "item":"string"}}
            ]
        }
 
There are no tag numbers in the Avro schema. In Avro encoded byte sequence there is nothing to identify fields or datatypes. The encoding simply consists of values concatenated together.

To parse the binary data, we go through the fields ion order that they appear in schema and use the schema to tell us the datatype of the field. Thus binary data can oinly be decoded correctly if the code reading the data is using the exact same schema as the code that wrote the data.
#### Schema Evolution with Avro
##### The writer's schema and the reader's schema
With Avro, the schema that an application uses to encode data is known as *writer's schema*. The schema that an application uses to read encoded data is known as *reader's schema*.

The key idea with Avro is that the writer's schema and reader's schema don't have to be same--they only need to be compatible. When the data is decoded(read), the Avro library resolves the differences by looking at the writer's schema and reader's schema side by side and translating the data from writer's schema into the reader's schema.
##### Schema Evolution Rules
**Forward Compatibility:** We can have a new version of schema as a writer and old version of schema as reader.
**Backward Compatibility:** We can have a new version of schema as reader and old version of schema as writer.

To maintain compatibility, you may only add or remove field that has a default value.

If we want to allow a field to be null, we have to use a *union type*. Consequently, Avro doesn't have optional and required markers.
##### How does a reader know about writer's schema?
It depends on the context in which avro is being used
- *Large file with lots of record*: Millions of records, all encoded with the same schema. Include writer's schema once at the begining of the file. Avro specifies a file format(object container files) to do this
- *Database with individually written records:* In a database, different record may be written at different pints in time using different writer's schemas. Keep a list of schema versions in the database, and include a version number at the begining of every encoded record. 
- *Sending records over a network connection:* Two processes communicating over a bidirectional network connection can negotiate the schema version on connection setup and then use that schema for the lifetime of the connection. The Avro RPC protocol works like this.
##### Dynamically generated schemas
As Avro schema doesn't contain field tags, it is friendlier to *dynamically generated schemas*(e.g. schemas generated from database tables) compared to Thrift or ProtoBuf.
##### Code generation and dynamically typed languages
Thrift and ProtoBuf rely on code generation. Avro provides optional code generation for statically typed languages, but it can be used just as well without any code generation. An *object container file* is self-describing since it includes all the necessary metadata. We can simply open it using an Avro library and look at the data in the same way as we could look at JSON file.
### The Merits of Schemas
Protocol Buffers, Thrift and Avro all use schemas to describe a binary encoding format. There schema languages are much simpler than XML or JSON schemas which support much more detailed validation rules. They have a number of nice properties:
- They can be much more compact than the various "binary JSON" variants, since they can omit field names from the encoded data.
- Schema is a valuable form of living documentation
- Keeping a database of schemas allows us to check forward and backward compatibility of schema changes, before anything is deployed.
- For users of statically typed languages, the ability to generate code from the schema is useful, since it enables type checking at compile time.

In summary, schema evolution allows same kind of flexibility as schemaless/schema-on-read JSON databases provide, while alos providing better guarantees about our data and better tooling.

## Modes of Dataflow
Compatibility is a relationship between one process that encodes the data, and another process that decodes it. There are many ways data can flow from one process to another. Some of the most common ways data flows between processes are:
- Via databases
- Via service calls
- Via asynchronous message passing
### Dataflow Through Databases
In a database, the process that writes to the database encodes the data, and the process that reads from the database decodes it. 

There may just be a single process accessing the database, in which case the reader is simply a later version of the same process--in that case we can think of storing something in database as *sending a message to our future self*. This requires backward compatibility.

It is common for several different processes to access the database at the same time. Those processes might be several different applications or services, or they may simply be different instances of the same service(running in parallel for scalability or fault tolerance). It is likely that some processes accessing the database will be running newer code and some will be running older code--for example becausea new version is currently being deployed in a rolling upgrade, so some instances have been upgraded while others haven't yet. This means that a value in database might be written by a newer version of code and read by an older version of code.  Thus forwaerd compatibility is also often required for databases. 

We also need to handle preservation of unknown fields(e.g. new fields added by newer code)
#### Different values written at different times
Within a single database, we may have values that were written 5 milliseconds ago and some that were written 5 years ago.

*Data outlives code:* We may entirely replace the old version with a new version of application within minutes. The same is not true for database contents: the five year old data will still be there in it's original encoding, unless we have explicitly rewritten it since then.

*Rewriting(migrating) data* into a new schema for a large dataset is expensive, so most databases avoid it if possible. 

*Schema evolution* allows entire database to appear as if it was encoded with a single schema, even though the underlying storage may contain records encoded with various historical versions of the schema.
#### Archival Storage
Data dumps (for backup or DWH), will typically be encoded using the latest schema.

As data dump is written in one go and is thereafter immutable, formats like Avro object container files are a good fit. This is also a good opportunity to encode the data in an analytics-friendly column-oriented format such as Parquet.
### Dataflow through Services: REST and RPC
For processes that need o communicate over a network, the most common arrangement is to have two roles: *clients and servers*.
The servers expose an API over the network, and the clients can connect to the servers to make requests to that API. The API exposed by a server is known as a *service*. Although HTTP may be used as the transport protocol, the API implemented on the top is application-specific, and the client and server need to agree on the details of that API. 

Services are similar to databases in some ways: they allow clients to submit and query data. However, while database support arbitrary queries, services expose an application specific API, only allowing inputs and outputs predetermined by the business logic of the service. This provides a degree of encapsulation: services can impose fine-grained restrictions on what clients can and cannot do.

A server can itself be a client to another *service* (e.g. SOA or microservices). A key design goal of service-oriented/microservices architecture is to is to make applications easier to change and maintain by making services independently deployable and evolvable. Thus, we should expect old and new versions of clients and servers to be running at the same time, and so the data encoding used by the servers and clients must be compatible across versions of the service API.
#### Web services
When HTTP is used as the underlying protocol for talking to the service, it is called a *web service*. Web services are used in several different contexts and not just on web.
1. A client application running on a user's device(e.g. a native mobile app or JS web app using Ajax) making requests  to a service over HTTP, usualy over the internet.
2. One service making requests to another service owned by the same organization, often located within the same datacenter, as part of a service-oriented/microservices architecture(Software that supports this kind of use case is sometimes called middleware)
3. One service is making requests to a service owned by a different organization, usually via the internet. this is used for data exchange between different organization's backend systems.

Two poular approaches to web services are *REST* and *SOAP*.

REST is not a protocol, but rather a design philosophy that builds upon the principles of HTTP. It emphasizes simple data formats, using URLs for identifying resources and using HTTP features for cache control, authentication, and content type negotiation

SOAP by contrast, is an XML-based protocol for making network API requests. Although it is most commonly used with HTTP, it aims to be independent from HTTP and avoids using most HTTP features. API of SOAP-based services is defined using an XML-based language called WSDL. 
#### The problems with remote procedure calls(RPC)
Web services are just one type of RPC. Other examples include EJB, Java RMI, DCOM, CORBA etc.

The RPC model tries to make a *request to a remote network service* look the same as *calling a function or method in your programming language within the same process*. This is called *location transparency*. This approach is fundamentally flawed as the network request is very different than a local function call.
- *A local function call is predictable:* it either succeds or fails, depending on parameters that are under your control. *A network request is unpredictable:* the request or response may be lost due to a network problem,  or the remote machine might be slow or unavailable, and such problems are entirely outside of our control. Network problems are common, so you have to anticipate them, for example by retrying a failed request.
- A local function call either returns a result, or throws an exception, or never returns(because it goes into an infinite loop or it crashes). A network request has another possible outcome: it may return without a result, in case of a timeout. In that case, we simply don't know what happened: if we don't get a response from remote service, there is no way of knowing whether the request got through or not.
- If we retry a failed request, it could happen that the previous request did go through and only the response was lost. In that case, retrying will cause the action to be performed multiple times, unless you build a mechanism for deduplication(*idempotence*) into the protocol.
- Every time we call a local function, it takes about the same time to execute. A network request is much slower than the function call, and its latency is also wildly variable.
-  When you call a local function, you can efficiently pass it reference(pointers) to objects in local memory. In a network request all those parameters need to be encoded into a sequence of bytes that can be sent over the network.That's okay if the parameters are primitives like numbers or strings but quickly become problematic with larger objects.
- The client and service may be implemented in different programming languages, so the RPC framework must translate datatypes from one language into another.

All of these factors mean that there's no point trying to make a remote service look too much like a local object in our programming language. Part of the appeal of REST is that it doeesn't try to hide the fact that it's a network protocol
#### Current directions for RPC
Various RPC frameworks have been built on top of encodings discussed 
- Thrift and Avro come with RPC support included
- gRPC uses Protocol Buffers
- Finagle uses Thrift
- Rest.li uses JSON over HTTP

The new generation of RPC frameworks are more explicit about remote request being different from a local function call through following features:
- *Futures(promises)*: Futures encapsulate asynchronous actions that may fail. Futures also simplify situations where we need to make multiple requests in parallel, and combine their results.
- *Streams*: With streams, a call consists of not just one request and one response, biut a series of requests and responses over time.
- *Service Discovery:* Service discovery allows clients to find out at which IP address and port number it can find a particular service.

*Advantage of RPC:*
Custom RPC protocols with a binary encoding format can achieve better performance than something generic like JSON or REST.  
*Advantages of a RESTful APIs*:
- It is good for experimentation and debugging(we can simply make requests to it using a web browser or the command-line tool curl, without any code generation or software installation)
- It is supported by all mainstream programming languages and platforms
- There is a vast ecosystem of tools available(servers, caches, proxies, load balancers, firewalls, monitoring, debugging tools, testing tools, etc.)

For these reasons, REST seems to be the predominant style for public APIs. The main focus of RPC frameworks is on requests between services owned by the same organization, typically within the same datacenter.
#### Data encoding and evolution for RPC
For evolvability, it is important that RPC clients and servers can be changed and deployed independently. 

In case of dataflow through services, it is reasonable to assume that all the servers will be updated first, and all the clients second. Thus, you only need backward compatibility on requests, and forward compatibility on responses.

The backward and forward compatibility properties of an RPC scheme are inherited from whatever encoding it uses.

Servicecompatibility is often made harder by the fact that RPC is often used for communication across organizational boundaries, so the provider of a service often has no control over the clients, and cannot force them to upgrade. Thus compatibility needs to be maintained for a long time, perhaps indefinitely. 

If a compatibility-breaking change is required, the service provider often ends up maintaining multiple versions of the service API side by side. There is no agreement on how API versioning should work.
For RESTFul APIs common approaches are:
- Use version number in the URL or HTTP header
- Store clients requested API version on the servers(when API keys are used to identify a client.)
### Message Passing Dataflow
*Asynchronous message pasing systems* are somewhere in between RPC and databases.
- Similar to RPC, the client's request(message) is delivered to another process with low latency.
- Similar to databases, the message is not sent via a direct network connection, but goes via an intermediary called a message broker(also called a *message queue* or *message-oriented middleware*).
#### Message Brokers
In general, message brokers are used as follows. One process sends a message to a named *queue* or *topic*, and the broker ensures that the message is delivered to one or more *consumers* of or *subscribers* to that queue or topic. There can be many producers and many consumers on the same topic.

Message brokers typically don't enforce any particular data model--a message is just a sequence of bytes with some metadata, so you can use any encoding format. If the encoding is backward and forward compatible, we have the greatest flexibility to change publishers and consumers independently and deploy them in any order.
##### Advantages of Message Broker over RPC
- It can act as a buffer if the recipient is unavailable or overloaded, and thus improve system reliability.
- It can automatically redeilver messages to a process that has crashed, and thus prevent messages from being lost.
- It avoids the sender needing to know the IP address and port number of the recipient(which is particularly useful in cloud deployment where virtual machines come and go).
- It allows one message to be sent to several recipients.
- It logically decouples the sender from the recipient(the sender just publishes the messages and doesn't care who consumes them)
- The communication pattern is asynchronous(the sender doesn't wait for the message to be delivered, but simply sends it and then forgets it)
#### Distributed Actor Frameworks
The actor model is a programming model for concurrency in a single process. Rather than dealing directly with threads(and the associated problems of race conditions, locking, and deadlock), logic is encapsulated in actors. Each actor typically represents one client or entity, it may have some local state(which is not shared with any other actor), and it communicates with other actors by sending and receiving asynchronous messages. Message delivery is not guaranteed: in certain scenarios, messages will be lost. Since each actor processes only one message at a time, it doesn't need to worry about threads, and each ator can be scheduled independently by the framework.

In *distributed actor frameworks*, this programming model is used to scale an application across many nodes. The same message-passing mechanism is used, whether the sender and recipient are on the same node or different nodes. If they are on different nodes, message is transparently encoded into a sequence of bytes, sent over the network, and decoded on the other side.

Location transparency works better in the actor model than in RPC, because the actor model already assumes that the messages may be lost, even within a single process.

A distributed actor framework essentially integrates a message broker and the actor programming model into a single framework.