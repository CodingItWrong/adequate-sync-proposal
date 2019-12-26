# Adequate Sync Proposal

## Goals
Adequate Sync is a standard that allows multiple clients and a single authoritative server to manipulate a data set, including both live updates and offline functionality.

Client and server applications should be able to be built with almost zero configuration by relying on reusable Adequate Sync libraries.
These libraries in turn should be able to be implemented with minimal complexity on top of existing technologies such as relational databases, to allow broad adoption across a variety of client and server languages.

## Enabling Constraints
The goals of Adequate Sync are lofty, and they are only achievable by applying a number of enabling constraints (if then!).
The constraints are:

* **The entire data set available to a client is synced down at once.**
  This is necessary so that all data is available for reading and writing offline.
  This does not mean *all the data on the server* needs to be synced down; for example, in utility applications where a user's data is only available to herself, all of *that user's* data is downloaded, but not other users' data.
* **Doesn't need to prevent conflicts.**
  Data structures that prevent conflicts (Conflict-free replicated data types, CRDTs) are not in widespread use, and so would prevent broad implementation of the standard.
  Instead, client libraries should have a mechanism for applications to be presented with conflicts to resolve automatically or with user intervention, in a way that works for that application.
* **Doesn't need to provide a snapshot of any point in time.**
  Although operations are a first-class citizen, Adequate Sync is not a full event sourcing system.
  It doesn’t provide facilities to reconstruct the state of the database as of any operation or “version”.
  To support implementing Adequate Sync on top of traditional databases, the data is still primarily stored in a “flattened” format.
  But operations are also stored, to enable clients to easily receive remote changes without needing to re-download the entire data set.
* **Doesn't need to support many clients with extremely fast changes.**
  The standard hasn't been rigorously assessed to ensure that if many operations come in quickly, that none will ever be lost or recorded out of order.
  Careful assessment would be needed, and that guarantee might require a significantly different standard.
* **This is not a document-syncing standard.**
  That is, it does not focus on syncing multiple changes within a larger text or content field.
* **There is an authoritative central server.**
  The standard is not peer-to-peer.
  This avoids having to determine consensus for operations.
  The single server is authoritative.
* **This version of the proposal does not address related data.**
  This is intended to be added soon: Adequate Sync will follow a relational data model.

## Protocol
### Data Format
Adequate Sync provides two types of data structures:

- Record: represent the data in the data set.
  A record is a JSON object.
  Must include an `id` property whose value is a UUID, that serves as a key.
  Using UUIDs ensures that records can be created offline without conflicting IDs.
  Other attributes can be set on the record object as well.
- Operation: a JSON object representing a change to data.
  Properties:
	* `id` - the UUID of the operation itself
	* `action` - the type of action the operation corresponds to. For now, this includes create, update, and delete. More may be added in the future.
	* `recordId` - the UUID of the record the operation applies to. This must always be provided by the client, even when creating records: the client that creates the record assigns the ID value.
	* `attributes` - for all operations except delete, the values to be provided for the record. This version of the proposal doesn't specify whether or not update operations may specify only the changing attributes and omit unchanged ones.

### Server
An Adequate Sync server exposes an HTTP endpoint and optionally a corresponding WebService endpoint.
Data is communicated in JSON format.
The following subpaths and HTTP methods are available:

HTTP
* `GET /` - retrieve all records in the data set.
  Returns an array of records. 
* `GET /operations?since=` - retrieves all operations that the server has received since the `since` query parameter.
  Operations should be applied in the order they are received in the array.
	* The reference implementation uses local machine timestamps for the `since` parameter, and this works running the client and server on one machine, but will not work as a final solution.
    A future version of the proposal will specify how `since` values work: probably some kind of server-provided value
* `POST /operations?since=` - applies one or more operations to change the data set.
  The Content-Type is `application/json` and the value is an array of operation objects.
  Operations are applied in the order they are listed in the array.
	* The return value is the same as that of `GET /operations` - it's the operations that have been applied to the server since the `since` parameter, excluding the `POST`ed operations.
    In other words, you can proactively `GET` the server operations, or you can `POST` an operation in which case you'll receive back any server operations anyway.
	* Operations must be applied in a transaction: that is, if a server is not able to successfully apply all operations, then it may not make changes from any of the operations.

WebService
* Receive: clients will receive messages that contain a JSON-encoded array of operations any time operations are applied to the server.
  This allows live updating of the client and makes conflicting data less likely.
* Send: clients may send messages containing a JSON-encoded array of operations, equivalent to `POST /operations`.
	* While a client is connected to the WebSocket, it should only send operations via WebSocket and not via post.
    This allows the server to skip the client that sent the operations while broadcasting them to other clients.
	* Unlike `POST /operations`, there is no need to immediately return past operations after sending operations.
    This is because those operations will have already proactively been received over the WebSocket.

### Client
Clients may provide whatever interface they want to the consuming application.
The following rules govern how clients are intended to interact with Adequate Sync servers:

* Initial data should be retrieved by a `GET` to the root (/) of the adequate sync endpoint.
* At any time after the initial data load, the client can `GET /operations` to retrieve updates to the data from the server.
  The client must apply these operations in the order they are sent in the array.
* If the server provides a WebService and the client connects to it, then any received arrays of operations should be applied to the local client data in the same way as results from `GET /operations`.
* When a change is made to data, the client should first attempt to send the operation to the server before applying it locally.
	* If the server returns an error, the server will not have made any changes to the data on the server: the operations must be applied in a single transaction.
	* If the server returns a success HTTP status and an empty array, this means that there were no server operations applied between the last `GET` and when this client's operations were `POST`ed, so the operations can be safely applied to the local data.
	* If the server returns a success HTTP status and a non-empty array of operations, this means that these operations had been applied to the server between the client's last update from the server and when it `POST`ed new operations.
    The server has already applied the POSTed operations, and it returns the preceding operations that the client has not yet seen. Applying these will be addressed below
	* If the server is not accessible, 
		* Note about persisting

How should the client handle when a server returns operations that were previously applied to the server that the client doesn't have?
The client library will call a resolution function specified by the consuming application.
The resolution function should have the following interface:

* Arguments
	* "New operations": if returning from a `POST /operations` request, an array of the operations that were POSTed.
    As per the standard these operations will not yet have been applied to the local data.
	* "Queued operations": any operations that were made while the server was unreachable.
    They will have already been applied to the local data.
	* "Remote operations": operations returned from the server that had been applied to the server since the last time the client received operations.
    They are not yet applied to the local data.
* Return value: the operations that should be applied to the local store.

What are the considerations the resolution strategy should take?
It should not return the "queued operations" in any case, as those have already been applied to the local data.

For the sake of brevity, call the different groups of records:

* A: the "queued operations"
* B: the "new operations"
* C: the "remote operations"

The order the client saw the operations is ABC. A has already been applied on the client, but B and C have not.

The server has already applied the C operations. Then it was sent AB and applied them. So the order the server has applied all the operations is CAB.

So the question is, what can the client do to end up in the same result state CAB, given that A has already been applied on the client but B and C have not?

A few general possibilities:

* It may be that simply returning CB (the remote operations followed by the new local operations that have not yet been applied) results in the same state as CAB.
  This is true if and only if C and A can be applied in either order with the same result.
  For example, if both A and C consist only of create operations, the order they are applied may not matter.
  Then again, if the user had seen the C operations already applied, she may not have chosen to apply the A operations.
* It may be that the client can calculate the end state of applying CAB, diff that with the current state of its store, and return new different operations that put the data into the correct state.
  The standard doesn't require identical operations to be applied on the client as on the server.
* It may be that there is no way to programmatically determine a good outcome of the conflict.
  In this case you can have the function return an empty array to apply no operations, and as a side effect present an interface to the user to specify which changes to keep.

The client library may make resolution functions using different strategies available, but the client library should not choose a resolution strategy by default: this risks corrupting data.
Consuming applications should make a conscious choice of what resolution strategy to apply.
Failing to specify a resolution strategy should be a client configuration error.

(Future versions of the proposal may be able to identify cases that can be automatically handled, but for simplicity this version specifies that all returned records need to be passed to a resolution function.)

## Reference Implementations
Preliminary reference implementations have been created:

* [Node](https://github.com/CodingItWrong/operative-node)
* [Client JavaScript](https://github.com/CodingItWrong/operative-client)
* [React hooks, using Client JavaScript under the hood](https://github.com/CodingItWrong/operative-react)

A sample application can be found here:

* [Todo Server](https://github.com/CodingItWrong/operative-example-node)
* [Todo Client - Web](https://github.com/CodingItWrong/operative-example-react)
* [Todo Client - React Native](https://github.com/CodingItWrong/operative-example-native)

## Future Concers
* Handling live updates over abstractions over WebSockets, such as Rails Action Cable and Phoenix Channels.

## Concerns Not Addressed
The following are outside the scope of this standard:

* Authentication and Authorization
* Static typing: it may be added to the standard and/or implementations, but it is not a core concern

## Similar Solutions
* **Google Firebase** provides support for live updates, offline data, and sync.
  It is a proprietary system that doesn't allow you to host your own data.
  It also doesn't store data relationally.
* **GraphQL** provides subscription functionality for live updates, and the popular Apollo Client and Server implement it.
  But neither the spec nor Apollo's implementations provide support for offline or sync functionality.
* **JSON:API** shares Adequate Sync's focus on relational data.
  Orbit.js is an implementation that allows configuring offline data and queuing up operations to resend later.
  But since tracking operations isn't part of the JSON:API spec, servers do not have a way to send new operations to clients.
* **Feathers.js** simplifies the process of setting up WebSocket-based clients and servers.
  It does not include support for offline or sync capability.
