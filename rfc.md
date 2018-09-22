A Simple RFC (if you could even call it that)
=============================================
_Note: Nothing here is final. ALL of it is up for debate/discussion. As a matter of fact, it is actively sought._

Connections
-----------
I believe this is all that should really be needed for developing a TCP chat with another user.
```rust,no-run
struct Request {
    to: Vec<String>,
    from: String,
    options: Vec<ConnOption>,
}
```
|Field|Description|Accepted |
|-----|-----------|---------|
| to | Takes a user@host:port/resource formatted string. | ? |
| from | Takes a user@host:port/resource formatted string. | ? |
| options | Vector of various possible options specific to protocol. | ? |

```rust,no-run
struct Acknowledge {
    id: String,
}
```
|Field|Description|Accepted |
|-----|-----------|---------|
| id | The exact string advertised from the request | ? |


```rust,no-run
struct Connection {
   id: String,
   users: Vec<String>,
   last_active: i64,
}
```
|Field|Description|Accepted |
|-----|-----------|---------|
| id | A string built from a Request to and from as well as a salt. | ? |
| users | A Vector of users involved in the Request | ? |
| last_active | A field to represent when socket was last active. | ? |

Messages
--------
This one is going to require some thought. I'd love to keep it as simple as possible though.
```rust,no-run
struct Message {
    category: String,   // Enum this
    subject: String,
    flags: Vec<String>, // Strings here so it can be easily extended clientside.
    date: i64,          // SystemTime::now() - UNIX_EPOCH maybe?
    msg: String,
}
```
|Field|Description|Accepted |
|-----|-----------|---------|
| category | Identifier for type of message. eg: "chat", "private". | ? |
| subject | A simple string representing the subject or topic of the message. | ? |
| flags | Vector of client side flags, useful for custom features and the like. | ? |
| date | Unix Timestamp (seconds) representing the date of message SENT (not recieved) | ? |
| msg | The content of the message, encrypted with a yet-to-be-decided cipher. | ? |

ConnOptions - Enum
------------------
```rust,no-run
enum ConnOption {
    Version,
    Dummy,
    TLS,
    SSL,
}
```
Here we can specify protocol options to go with the initial connection handshake. Let's be silly here and come up with everything neat.

***

Handshake
=========
The handshake to begin a connection is extremely simple (at least for now) in that it just expects a properly encoded and properly structured msgpack datum.

Sequence of events:
1. Bob connects to server and waits.
2. Alice connects to server and builds a Request datum to chat with Bob.
3. Server builds Connection datum based on Request and fires event on the participating parties line.
4. Bob receives event from Server and can choose to Acknowledge or ignore event.
5. On Acknowledge of Request, Server spawns thread and socket detailed in Connection datum and sends OK to involed parties.
6. Bob and Alice are now being relayed over Connection.