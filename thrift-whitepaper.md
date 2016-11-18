Thrift whitepage

## Types

### The goal of the Thrift type system is to enable programmers to develop using completely natively defined types, no matter what programming language they use. 

#### 1. Base type
* bool
* type
* i16
* i32
* i64
* double
* string

#### 2. Struct
```json
struct User {
	1: required i32 user_id,
	2: required string username,
	3: optional i16 age,
}
```

#### 3. Containers
 * **list\<type\>** An ordered list of elements. Translates directly into an STL vector, Java ArrayList, or native array in script- ing languages. May contain duplicates.
 * **set\<type\>** An unordered set of unique elements. Translates into an STL set, Java HashSet, set in Python, or native dictionary in PHP/Ruby.
 * **map\<type1, type2\>** A map of strictly unique keys to values Translates into an STL map, Java HashMap, PHP associative array, or Python/Ruby dictionary.
 
```json
 struct Class {
 	1: required i16 class_id,
	2: required list<User> users
 }
```

#### 4. Exception
```json
exception Error {
	1: required i16 error_code,
	2: optional string error_msg
}
```

#### 5. Service
* Services are defined using Thrift types. Definition of a service is semantically equivalent to defining an interface (or a pure virtual abstract class) in object oriented programming. 

```
service <name> {	<returntype> <name>(<arguments>)		[throws (<exceptions>)]
	...}
```

```json
service UserService {
	void set_username(1:i32 user_id, 2:string value),
	string get_username(1:i32 user_id) throws (1:Error err)
}

```

> Additionally, an async modifier keyword may be added to a void function, which will generate code that does not wait for a response from the server. Note that a pure void function will return a response to the client which guar- antees that the operation has completed on the server side. With async method calls the client will only be guaranteed that the re- quest succeeded at the transport layer. 


## Transport

### The transport layer is used by the generated code to facilitate data transfer.

#### 1. Interface
Fundamentally, generated Thrift code only needs to know how to read and write data. The origin and destination of the data are irrelevant; it may be a socket, a segment of shared memory, or a file on the local disk. The Thrift transport interface supports the following methods:

- **open** Opens the tranpsort- **close** Closes the tranport- **isOpen** Indicates whether the transport is open • read Reads from the transport- **write** Writes to the transport- **flush** Forces any pending writes

In addition to the above TTransport interface, there is a TServerTransport interface used to accept or create primitive transport objects. Its interface is as follows:

- **open** Opens the transport- **listen** Begins listening for connections 
- **accept** Returns a new client transport- **close** Closes the transport

#### 2. Implementation
- TSocket
	
	The TSocket class is implemented across all target languages. It provides a common, simple interface to a TCP/IP stream socket.
	
- TFileTransport
	
	The TFileTransport is an abstraction of an on-disk file to a data stream. It can be used to write out a set of incoming Thrift requests to a file on disk. The on-disk data can then be replayed from the log, either for post-processing or for reproduction and/or simulation of past events.
	
> The Transport interface is designed to support easy extension us- ing common OOP techniques, such as composition. Some sim- ple utilites include the TBufferedTransport, which buffers the writes and reads on an underlying transport, the TFramedTransport, which transmits data with frame size headers for chunking op- timization or nonblocking operation, and the TMemoryBuffer, which allows reading and writing directly from the heap or stack memory owned by the process.


## Protocol
### A second major abstraction in Thrift is the separation of data structure from transport representation.
#### 1. Interface
* The Thrift Protocol interface is very straightforward. It fundamen- tally supports two things: 
	* 1) bidirectional sequenced messaging
	* 2) encoding of base types, containers, and structs.
	
#### 2. Structure
* Thrift structures are designed to support encoding into a streaming protocol. The implementation should never need to frame or com- pute the entire data length of a structure prior to encoding it. *This is critical to performance in many scenarios.*
* Similarly, structs do not encode their data lengths a priori. Instead, they are encoded as a sequence of fields, with each field having a type specifier and a unique field identifier. *Note that the inclusion of type specifiers allows the protocol to be safely parsed and decoded without any generated code or access to the original IDL file.*

## Versioning
### Thrift is robust in the face of versioning and data definition changes. This is critical to enable staged rollouts of changes to deployed services. The system must be able to support reading of old data from log files, as well as requests from out-of-date clients to new servers, and vice versa.

#### 1. Field Identifier
* Versioning in Thrift is implemented via field identifiers. 

* The field header for every member of a struct in Thrift is encoded with a unique field identifier.
 
* **The combination of this field identifier and its type specifier** is used to uniquely identify the field. 

> To avoid conflicts between manually and automatically assigned identifiers, fields with identifiers omitted are assigned identifiers decrementing from -1, and the language only supports the manual assignment of positive identifiers.

* Field identifiers can (and should) also be specified in function argument lists.

* Field identifiers internally use the i16 Thrift type. Note, however, that the TProtocol abstraction may encode identifiers in any format.

#### 2. Isset
* When an unexpected field is encountered, it can be safely ignored and discarded. When an expected field is not found, there must be some way to signal to the developer that it was not present. This is implemented via an inner isset structure inside the defined objects. (Isset functionality is implicit with a null value in PHP, None in Python and nil in Ruby.)

#### 3. Case Analysis
* Added field, old client, new server. In this case, the old client does not send the new field. The new server recognizes that the field is not set, and implements default behavior for out-of-date requests.
* Removed field, old client, new server. In this case, the old client sends the removed field. The new server simply ignores it.
* Added field, new client, old server. The new client sends a field that the old server does not recognize. The old server simply ignores it and processes as normal.
* Removed field, new client, old server. This is the most danger- ous case, as the old server is unlikely to have suitable default behavior implemented for the missing field. **It is recommended that in this situation the new server be rolled out prior to the new clients.**

#### 4. Protocol/Transport Versioning
* The TProtocol abstractions are also designed to give protocol implementations the freedom to version themselves in whatever manner they see fit. Specifically, any protocol implementation is free to send whatever it likes in the **writeMessageBegin()** call. It is entirely up to the implementor how to handle versioning at the protocol level. The key point is that protocol encoding changes are safely isolated from interface definition version changes.


## RPC Implementation

#### 1. TProcessor

* The last core interface in the Thrift design is the TProcessor, perhaps the most simple of the constructs. The interface is as follows:

```jsoninterface TProcessor {	bool process(TProtocol in, TProtocol out)    	throws TException}
```
* The key design idea here is that the complex systems we build can fundamentally be broken down into agents or services that operate on inputs and outputs. In most cases, there is actually just one input and output (an RPC client) that needs handling.

#### 2. TServer
* The TServer object generally works as follows.
	* Use the TServerTransport to get a TTransport	* Use the TTransportFactory to optionally convert the primi- tive transport into a suitable application transport (typically the TBufferedTransportFactory is used here)	* Use the TProtocolFactory to create an input and output protocol for the TTransport	* Invoke the process() method of the TProcessor object
	
	
## Implementation Details