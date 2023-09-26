# **Chapter 4 - Encoding and Evolution** 
### **Outline** 
* ## Formats for Encoding Data 
  * ### Language-Specific Formats
  * ### JSON-XML, and Binary Variants
    * #### Binary Encoding 
  * ### Thrift and Protocol Buffers 
    * #### Field tags and schema evolution 
    * #### Datatypes and schema evolution 
  * ### Avro 
    * #### The writer’s schema and the reader’s schema 
    * **Schema evolution rules** 
    * **But what is the writer’s schema?** 
    * **Dynamically generated schemas** 
    * **Code generation and dynamically typed languages** 
  * ### The Merits of Schemas
* ## Modes of Data Flow
  * ### Dataflow Through Databases
    * #### Different values written at different times 
    * **Archival storage** 
  * ### Dataflow Through Services: REST and RPC 
    * #### Web services 
    * **The problems with remote procedure calls (RPCs)** 
    * **Current directions for RPC** 
    * **Data encoding and evolution for RPC** 
  * ### Message-Passing Dataflow
    * #### Message brokers
    * **Distributed actor frameworks**
* ## Summary 
- - -
### NOTES
* ## Intro
  * a change to an application’s features also requires a change to data that it stores 
  * relational databases generally assume that all data in the database conforms to one schema: although that schema can be changed (through schema migrations; i.e., ALTER statements), there is exactly one schema in force at any one point in time
  * schema-on-read (“schemaless”) databases don’t enforce a schema, so the database can contain a mixture of older and newer data formats written at different times 
  * w/ server-side applications you may want to perform a *rolling upgrade* aka a *staged rollout*) - deploying the new version to a few nodes at a time and checking whether the new version is running smoothly, and gradually working your way through all the nodes
  * this allows new versions to be deployed without service downtime, and encourages more frequent releases and better evolvability 
  * w/ client-side applications you’re at the mercy of the user, who may not install the update for some time 
  * this means that old and new versions of the code, and old and new data formats, may potentially all coexist in the system at the same time.
  * in order for the system to continue running smoothly, we need to maintain compatibility in both directions: 
    * Backward compatibility - newer code can read data that was written by older code, which Is simple 
    * Forward compatibility - Older code can read data that was written by newer code, which is tricker
* ## Formats for Encoding Data 
* programs usually work with data in (at least) two different representations: 
  * in memory, data is kept in objects, structs, lists, arrays, hash tables, trees, etc.
    * these data structures are optimized for efficient access and manipulation by the CPU (typically using pointers)
  * when you want to write data to a file or send it over the network, you have to encode it as some kind of self-contained sequence of bytes (for example, a JSON document)
    * a pointer wouldn’t make sense to any other process, so the sequence-of-bytes representation looks quite different from the data structures that are normally used in memory 
* the translation from the in-memory representation to a byte sequence is called *encoding* (also known as *serialization* or *marshalling*), and the reverse is called *decoding* (*parsing*, *deserialization*, *unmarshalling*) 
* ### Language-Specific Formats
  * many programming languages come with built-in support for encoding in-memory objects into byte sequences 
  * encoding libraries are very convenient, because they allow in-memory objects to be saved and restored with minimal additional code
  * Cons
    * encoding is often tied to a particular programming language, and reading the data in another language is very difficult which means you are committing yourself to your current programming language for potentially a very long time
    * in order to restore data in the same object types, the decoding process needs to be able to instantiate arbitrary classes, which becomes a source of security problems 
    * versioning data is often an afterthought in these libraries, as they are intended for quick and easy encoding of data
    * efficiency (CPU time taken to encode or decode, and the size of the encoded structure) is also often an afterthought
      * for these reasons it’s generally a bad idea to use your language’s built-in encoding for anything other than very transient purposes
* ### JSON-XML, and Binary Variants
  * JSON and XML are widely known and widely supported
  * XML is often criticized for being too verbose 
  * JSON’s popularity is due to its built-in support in web browsers simplicity relative to XML
  * CSV is another popular language-independent format but less powerful
  * JSON, XML, and CSV are textual formats
    * Downsides 
    * IN XML and CSV, you cannot distinguish between a number and a string that happens to consist of digits 
    * JSON distinguishes strings and numbers, but it doesn’t distinguish integers and floating-point numbers
    * JSON and XML have good support for Unicode character strings (i.e., human- readable text), but they don’t support binary strings (sequences of bytes without a character encoding)
    * schema support is complicated to learn and implement
    * CSV does not have any schema, is up to the application to define the mean‐ ing of each row and column
* #### Binary Encoding 
  * JSON is less verbose than XML, but both still use a lot of space compared to binary format 
  * *did not understand the message pack example*
* ### Thrift and Protocol Buffers 
  * Apache Thrift and Protocol Buffers are binary encoding libraries that are based on the same principle
  * Protocol Buffers was originally developed at Google andThrift was originally developed at Facebook
  * Thrift and Protocol Buffers require a schema for any data that is encoded
  * Thrift has two binary encoding formats, *BinaryProtocol* and *CompactProtocol*
  * not understanding *BinaryProtocol* and *CompactProtocol* examples 
  * required fields enable a runtime check that fails if the field is not set, which can be useful for catching bugs
  * #### Field tags and schema evolution 
    * *schema evolution* - schemas inevitably need to change over time
    * field tags are critical to the meaning of the encoded data
    * you can change the name of a field in the schema, since the encoded data never refers to field names, but you cannot change a field’s tag, since that would make all existing encoded data invalid 
    * forward compatibility: old code can read records that were written by new code 
    * backward compatibility: as long as each field has a unique tag number, new code can always read old data, because the tag numbers still have the same meaning 
    * to maintain backward compatibility, every field you add after the initial deployment of the schema must be optional or have a default value
  * #### Datatypes and schema evolution 
    * changing the datatype of a field creates a risk that values will lose precision or get truncated 
* ### Avro 
  * Apache Avro is a binary encoding format 
  * Avro uses a schema to specify the structure of the data being encoded 
  * It has two schema languages
    * one (Avro IDL) intended for human editing
    * one (based on JSON) that is more easily machine-readable
    * Avra supports schema evolution 
  * #### The writer’s schema and the reader’s schema 
    * *writer’s schema* - when an application wants to encode some data (to write it to a file or database, to send it over the network, etc.), and encodes the data using whatever version of the schema it knows about—for example, that schema may be compiled into the application
    * reader’s schema - when an application wants to decode some data (read it from a file or database, receive it from the network, etc.), and expects the data to be in some schema
      * This is the schema the application code is relying on —code may have been generated from that schema during the application’s build process
    * key idea - writer’s schema and the reader’s schema *don’t have to be the same*—they only need to be compatible 
    * Avro library resolves the differences by looking at the writer’s schema and the reader’s schema side by side and translating the data from the writer’s schema into the reader’s schema 
    * if the code reading the data encounters a field that appears in the writer’s schema but not in the reader’s schema, it is ignored
    * if the code reading the data expects some field, but the writer’s schema does not contain a field of that name, it is filled in with a default value declared in the reader’s schema
  * **Schema evolution rules** 
    * with Avro, forward compatibility means you can have a new version of the schema as writer and an old version of the schema as reader
    * backward compatibility means that you can have a new version of the schema as reader and an old version as writer
    * to maintain compatibility, you may only add or remove a field that has a default value 
  * **But what is the writer’s schema?** 
    * there are different ways a reader can know when the writer’s schema encoded such as versioning 
  * **Dynamically generated schemas** 
    * Avro is friendlier to *dynamically generated* schemas
  * **Code generation and dynamically typed languages** 
    * Thrift and Protocol Buffers rely on code generation: after a schema has been defined, you can generate code that implements this schema in a programming language of your choice
    * in dynamically typed programming languages such as JavaScript, Ruby, or Python, there is not much point in generating code, since there is no compile-time type checker to satisfy 
    * *self-describing* - avro file includes all the necessary metadata 
* ### The Merits of Schemas
  * Protocol Buffers, Thrift, and Avro all use a schema to describe a binary encoding format
  * their schema languages are much simpler than XML Schema or JSON Schema, which support much more detailed validation rules 
  * Protocol Buffers, Thrift, and Avro are simpler to implement and simpler to use and have grown to support a fairly wide range of programming languages
  * many data systems also implement some kind of proprietary binary encoding for their data
  * example - most relational databases have a network protocol over which you can send queries to the database and get back responses, which can generally specific to a particular database, and the database vendor provides a driver (e.g., using the ODBC or JDBC APIs) that decodes responses from the data‐ base’s network protocol into in-memory data structures. 
* binary encodings based on schemas can be much more compact than the various “binary JSON” variants, since they can omit field names from the encoded data. 
* schema is a valuable form of documentation, and because the schema is required for decoding, you can be sure that it is up to date (whereas manually maintained documentation may easily diverge from reality)
* keeping a database of schemas allows you to check forward and backward compatibility of schema changes, before anything is deployed
* for statically typed programming languages, the ability to generate code from the schema is useful, since it enables type checking at compile time
* schema evolution allows the same kind of flexibility as schemaless/ schema-on-read JSON databases provide while also providing better guarantees about your data and better tooling. 
* ## Modes of Data Flow
  * whenever you want to send some data to another process with which you don’t share memory, you need to encode it as a sequence of bytes
    * forward and backward compatibility are important for evolvability (making change easy by allowing you to upgrade different parts of your system independently, and not having to change everything at once)
    * compatibility is a relationship between one process that encodes the data, and another process that decodes it
    * some of the most common ways how data flows between processes: 
      * Via databases 
      * Via service calls 
      * Via asynchronous message passing 
  * ### Dataflow Through Databases
    * the process that writes to the database encodes the data
    * the process that reads from the database decodes it
    * it’s common for several different processes to be accessing a database at the same time
    * processes might be several different applications or services, or they may be several instances of the same service (running in parallel for scalability or fault tolerance)
    * in an environment where the application is changing, it is likely that some processes accessing the database will be running newer code and some will be running older code
    * backward and forward compatibility is crucial 
    * you need to make sure any new data is preserved by older code at an application level 
  * #### Different values written at different times 
    * a database generally allows any value to be updated at any time
    * *data outlives code* - idea that code gets updated and replaced routinely but not database contents + encoding
    * rewriting (*migrating*) data into a new schema is certainly possible, but is expensive thing to do on a large dataset, so most databases avoid it if possible
    * most relational databases allow simple schema changes, such as adding a new column with a null default value, without rewriting existing data
    * when an old row is read, the database fills in nulls for any columns that are missing from the encoded data on disk
    * schema evolution allows the entire database to appear as if it was encoded with a single schema, even though the underlying storage may contain records encoded with various historical versions of the schema 
* **Archival storage** 
  * take a snapshot of your database from time to time, for backup purposes or for loading into a data warehouse 
  * the data dump will typically be encoded using the latest schema, even if the original encoding in the source database contained a mixture of schema versions from different eras.
* ### Dataflow Through Services: REST and RPC 
  * the most common arrangement to have processes that need to communicate over a network is to have two roles: 
    * *clients* and *servers*
    * servers expose an API over the network
    * clients can connect to the servers to make requests to that API
    * The API exposed by the server is known as a *service*
  * the web works this way: clients (web browsers) make requests to web servers, making GET requests to download HTML, CSS, JavaScript, images, etc., and making POST requests to submit data to the server.
  * the API consists of a standardized set of protocols and data formats (HTTP, URLs, SSL/TLS, HTML, etc.) 
  * web browsers are not the only type of client.
    * a native app running on a mobile device or a desktop computer can also make network requests to a server
    * a client-side JavaScript application running inside a web browser can use XMLHttpRequest to become an HTTP client (this technique is known as *Ajax*)
    * in this case, the server’s response is typically not HTML for displaying to a human, but rather data in an encoding that is convenient for further processing by the client- side application code (such as JSON)
    * although HTTP may be used as the transport protocol, the API implemented on top is application-specific, and the client and server need to agree on the details of that API
    * a server can itself be a client to another service (for example, a typical web app server acts as client to a database). 
      * this approach is often used to decompose a large application into smaller services by area of functionality, such that one service makes a request to another when it requires some functionality or data from that other service
      * this way of building applications has traditionally been called a *service-oriented architecture* (SOA), more recently refined and rebranded as *microservices architecture*
      * services are similar to databases: 
        * they typically allow clients to submit and query data
        * while databases allow arbitrary queries using the query languages, services expose an application-specific API that only allows inputs and outputs that are predetermined by the business logic (application code) of the service 
        * this restriction provides a degree of encapsulation: services can impose fine-grained restrictions on what clients can and cannot do
        * key design goal of a service-oriented/microservices architecture is to make the application easier to change and maintain by making services independently deployable and evolvable
        * example - each service should be owned by one team, and that team should be able to release new versions of the service frequently, without having to coordinate with other teams. 
        * goal - we should expect old and new versions of servers and clients to be running at the same time, and so the data encoding used by servers and clients must be compatible across versions of the service API
* #### Web services 
  * web service - when HTTP is used as the underlying protocol for talking to the service
  * web services are used when:
    * A client application running on a user’s device (e.g., a native app on a mobile device, or JavaScript web app using Ajax) making requests to a service over HTTP —> these requests typically go over the public internet
    * one service making requests to another service owned by the same organization, often located within the same data-center, as part of a service-oriented/microser‐ vices architecture —> (Software that supports this kind of use case is sometimes called *middleware*) 
    * one service making requests to a service owned by a different organization, usu‐ ally via the internet —> this is used for data exchange between different organizations’ backend systems
      * this category includes public APIs provided by online services, such as credit card processing systems, or OAuth for shared access to user data
  * there are two popular approaches to web services: *REST* and *SOAP* 
  * REST 
    * not a protocol, but rather a design philosophy that builds upon the principles of HTTP 
    * it emphasizes simple data formats, using URLs for identifying resources and using HTTP features for cache control, authentication, and content type negotiation
    * REST  is often associated with microservices 
    * an API designed according to the principles of REST is called *RESTful*
  * SOAP
    * SOAP is an XML-based protocol for making network API requests
    * it is most commonly used over HTTP, but aims to be independent from HTTP and avoids using most HTTP features
    * API of a SOAP web service is described using an XML-based language called the Web Services Description Language, or WSDL
    * WSDL enables code generation so that a client can access a remote service using local classes and method calls (which are encoded to XML messages and decoded again by the framework)
    * WSDL is not designed to be human-readable, and as SOAP messages are often too complex to construct manually, users of SOAP rely heavily on tool support, code generation, and IDEs 
  * REST APIs are favored in large orgs because they are simpler and reply less on tool support 
* **The problems with remote procedure calls (RPCs)** 
  * The RPC model tries to make a request to a remote network service look the same as calling a function or method in your programming language, within the same process (this abstraction is called *location transparency*)
  * although RPC seems convenient at first, the approach is fundamentally flawed because a network request is very different from a local function call: 
    * a local function call is predictable and either succeeds or fails, depending only on parameters that are under your control vs a network request is unpredictable: the request or response may be lost due to a network problem, or the remote machine may be slow or unavailable, and such problems are entirely outside of your control 
    * a local function call either returns a result, or throws an exception, or never returns (because it goes into an infinite loop or the process crashes) vs a network request has another possible outcome: it may return without a result, due to a *timeout*, and you have no way of knowing whether the request got through or not. 
    * if you retry a failed network request, it could happen that the requests are actually getting through, and only the responses are getting lost, retrying will cause the action to be performed multiple times, unless you build a mechanism for deduplication (*idempotence*) into the protocol 
    * every time you call a local function, it normally takes about the same time to exe‐ cute vs network request is much slower than a function call, and its latency is also wildly variable: at good times it may complete in less than a millisecond, but when the network is congested or the remote service is overloaded it may take many seconds to do exactly the same thing 
    * when you call a local function, you can efficiently pass it references (pointers) to objects in local memory but when you make a network request, all those parameters need to be encoded into a sequence of bytes that can be sent over the network 
    * client and the service may be implemented in different programming languages, so the RPC framework must translate datatypes from one language into another and not all languages have the same types
  * there’s no point trying to make a remote service look too much like a local object in your programming language, because it’s a fundamentally different thing
  * part of the appeal of REST is that it doesn’t try to hide the fact that it’s a network protocol (although this doesn’t seem to stop people from building RPC libraries on top of REST)
* **Current directions for RPC** 
  * new generation of RPC frameworks are more explicit about the fact that a remote request is different from a local function call 
  * ie using *futures* (*promises*) to encapsulate asynchronous actions that may fail
  * futures also simplify situations where you need to make requests to multiple services in parallel, and combine their results*
  * *streams*, where a call consists of not just one request and one response, but a series of requests and responses over time 
  * *service discovery*—allowing a client to find out at which IP address and port number it can find a particular service
  * a RESTful API has other significant advantages:
    * it is good for experimentation and debugging (you can simply make requests to it using a web browser or the command-line tool curl, without any code generation or software installation)
    * it is supported by all main‐ stream programming languages and platforms, and there is a vast ecosystem of tools available (servers, caches, load balancers, proxies, firewalls, monitoring, debugging tools, testing tools, etc.). 
  * REST seems to be the predominant style for public APIs
  * main focus of RPC frameworks is on requests between services owned by the same organization, typically within the same data-center. 
* **Data encoding and evolution for RPC** 
  * for evolvability, it is important that RPC clients and servers can be changed and deployed independently
  * compared to data flowing through databases (as described in the last section), we can make a simplifying assumption in the case of dataflow through services: it is reasonable to assume that all the servers will be updated first, and all the clients second so you only need backward compatibility on requests, and forward compatibility on responses
  * the backward and forward compatibility properties of an RPC scheme are inherited from whatever encoding it uses: 
    * Thrift, gRPC (Protocol Buffers), and Avro RPC can be evolved according to the compatibility rules of the respective encoding format
    * in SOAP, requests and responses are specified with XML schemas,  which can be evolved but has cons 
    * RESTful APIs most commonly use JSON (without a formally specified schema) for responses, and JSON or URI-encoded/form-encoded request parameters for requests and adding optional request parameters and adding new fields to response objects are usually considered changes that maintain compatibility
  * service compatibility is made harder by the fact that RPC is often used for communication across organizational boundaries, so the provider of a service often has no control over its clients and cannot force them to upgrade —> compatibility needs to be maintained for a long time, perhaps indefinitely —> if a compatibility-breaking change is required, the service provider often ends up maintaining multiple versions of the service API side by side 
  * there is no agreement on how API versioning should work (i.e., how a client can indicate which version of the API it wants to use)
    * RESTful APIs  —> common approaches are to use a version number in the URL or in the HTTP Accept header
    * for services that use API keys to identify a particular client, another option is to store a client’s requested API version on the server and to allow this version selection to be updated through a separate administrative interface  
* ### Message-Passing Dataflow
  * *asynchronous message-passing* systems - a client’s request (usually called a *message*) is delivered to another process with low latency but not sent via a direct network connection, but goes via an intermediary called a *message broker* (also called a *message queue* or *message-oriented middleware*), which stores the message temporarily. 
  * advantages of message broker: 
    * it can act as a buffer if the recipient is unavailable or overloaded, and thus improve system reliability 
    * it can automatically redeliver messages to a process that has crashed, and thus prevent messages from being lost
    * it avoids the sender needing to know the IP address and port number of the recipient (which is particularly useful in a cloud deployment where virtual machines often come and go) 
    * it allows one message to be sent to several recipients 
    * it logically decouples the sender from the recipient (the sender just publishes messages and doesn’t care who consumes them) 
    * message-passing communication is usually one-way: a sender normally doesn’t expect to receive a reply to its messages, it is possible for a process to send a response, but this would usually be done on a separate channel
    * this communication pattern is *asynchronous*: the sender doesn’t wait for the message to be delivered, but simply sends it and then forgets about it
  * #### Message brokers
    * message brokers are used as follows: 
      * one process sends a message to a named *queue* or *topic*
      * the broker ensures that the message is delivered to one or more *consumers* of or *subscribers* to that queue or topic
      * there can be many producers and many consumers on the same topic 
      * a topic provides only one-way dataflow
      * consumer may itself publish messages to another topic (so you can chain them together or to a reply queue that is consumed by the sender of the original message (allowing a request/response dataflow, similar to RPC)
      * message brokers typically don’t enforce any particular data model—a message is just a sequence of bytes with some metadata, so you can use any encoding format and if encoding is backward and forward compatible, you have the greatest flexibility to change publishers and consumers independently and deploy them in any order
  * **Distributed actor frameworks**
    * *actor model* is a programming model for concurrency in a single process
    * rather than dealing directly with threads (and the associated problems of race conditions, locking, and deadlock), logic is encapsulated in *actors*
    * each actor typically represents one client or entity, it may have some local state (which is not shared with any other actor), and it communicates with other actors by sending and receiving asynchronous messages
    * message delivery is not guaranteed: in certain error scenarios, messages will be lost
    * since each actor processes only one message at a time, it doesn’t need to worry about threads, and each actor can be scheduled independently by the framework 
    * in *distributed actor frameworks*, this programming model is used to scale an application across multiple nodes
    * the same message-passing mechanism is used, no matter whether the sender and recipient are on the same node or different nodes
    * if they are on different nodes, the message is transparently encoded into a byte sequence, sent over the network, and decoded on the other side
    * location transparency works better in the actor model than in RPC, because the actor model already assumes that messages may be lost, even within a single process
    * a distributed actor framework essentially integrates a message broker and the actor programming model into a single framework
    * three popular distributed actor frameworks: aka, orleans, erlang 
* ## Summary 
  * encodings affect not only their efficiency data structures turned into bytes on a network or bytes on disk, but also the architecture of applications and options for deploying them 
  * many services need to support rolling upgrades, where a new version of a service is gradually deployed to a few nodes at a time, rather than deploying to all nodes simultaneously
  * rolling upgrades 
    * allow new versions of a service to be released without downtime (thus encouraging frequent small releases over rare big releases)
    * make deployments less risky (allowing faulty releases to be detected and rolled back before they affect a large number of users)
    * very important for *evolvability*, the ease of making changes to an application. 
* during rolling upgrades, we must assume that different nodes are running the different versions of our application’s code 
* because of this, it is important that all data flowing around the system is encoded in a way that provides backward compatibility (new code can read old data) and forward compatibility (old code can read new data). 
* pros and cons of several data encoding formats and their compatibility properties: 
  * programming language–specific encodings are restricted to a single programming language and often fail to provide forward and backward compatibility
  * textual formats like JSON, XML, and CSV are widespread, and their compatibility depends on how you use them
    * they have optional schema languages, which are sometimes helpful and sometimes a hindrance.
    * these formats are somewhat vague about datatypes, so you have to be careful with things like numbers and binary strings 
* several modes of dataflow have different scenarios in which data encodings are important: 
  * databases, where the process writing to the database encodes the data and the process reading from the database decodes it 
  * RPC and REST APIs, where the client encodes a request, the server decodes the request and encodes a response, and the client finally decodes the response 
  * asynchronous message passing (using message brokers or actors), where nodes communicate by sending each other messages that are encoded by the sender and decoded by the recipient 


  







