A Simple RFC (if you could even call it that)
=============================================
_Note: Nothing here is final. ALL of it is up for debate/discussion. As a matter of fact, it is actively sought.  
Most of this is written from the viewpoint of the Server, which I feel is the "standard" of the protocol with most other features being client-side._

Datums
------
This enum describes a default for how to tell datums from one another as well as a brief overview of what is available/supported.
```rust,no-run
enum Datum {
    Auth = 0,
    AuthResponse = 1,
    Command = 2,
    Message = 3,
    Roster = 4,
    User = 5,
    Status = 6,
}
```  

_Note: All fields (except Message's "to" field as it's required by server) may be encrypted with your keys should you desire for E2E encryption._  

This is the "generic" description of each datum:
```rust,no-run
struct Auth {
    date: i64,
    challenge: String, // Encrypted with User's public key.
    last_seen: i64,
}
```
|Field|Description|Accepted |
|-----|-----------|---------|
| date | Date challenge was generated. | ? |
| challenge | A random string to be returned as the response. Encrypted with User's public key.| ? |
| last_seen | Last time of successful challenge. | ? |
```rust,no-run
struct AuthResponse {
    challenge: String,
}
```
|Field|Description|Accepted |
|-----|-----------|---------|
| challenge | Exact copy of challenge sent from server encrypted with servers public key. | ? |
```go,no-run
type Command struct {
	Cmd     string   `msgpack:"cmd"`
	Payload []string `msgpack:"payload"`
}
```
|Field|Description|Accepted|
|-----|-----------|---------|
| cmd | Text that indicates type of Command, eg. "AUTH" | ? |
| payload | Stuff that may be required to execute a command. | ? |

```go,no-run
type Message struct {
	To    string   `msgpack:"to"`
	From  string   `msgpack:"from"`
	Flags []string `msgpack:"flags"`
	Date  int64    `msgpack:"date"`
	Msg   string   `msgpack:"msg"`
}
```
|Field|Description|Accepted |
|-----|-----------|---------|
| to | Who should the Message go to? | ? |
| from | Who is it from? We will need some sort of spoof protection. | ? |
| flags | Vector of client side flags, useful for custom features and the like. | ? |
| date | Unix Timestamp (seconds) representing the date of message SENT (not recieved) | ? |
| msg | The content of the message, encrypted with a yet-to-be-decided cipher. | ? |

```go,no-run
var Roster []User
```
|Field|Description|Accepted |
|-----|-----------|---------|
| n/a |    n/a    |    n/a  |

```go,no-run
type User struct {
	Aliases   []string `msgpack:"aliases"`
	Key       []byte   `msgpack:"key"`
	LastSeen int64    `msgpack:"last_seen"`
}
```
|Field|Description|Accepted |
|-----|-----------|---------|
| alias | The names which this user has claimed for their use. | ? |
| key | An array of u8 bytes with a max size of 32 indexes. This represents the public key | ? |
| last_seen | A UNIX timestamp representing the last time a successful Auth/AuthResponse occured | ? |
```rust,no-run
struct Status {
    status: i8,
    payload: String,
}
```
|Field|Description|Accepted |
|-----|-----------|---------|
| status | Negative value is an error, positive is "OK" | ? |
| payload | A place to stick other useful data, readable by humans | ? |
Connections
-----------
_NOTE: All example Datums in this section use information that matches the type but may not be actual values. Example being the "challenge" field in Auth datum. This will be longer than 20 characters, and not just alphanumeric._  
All packet datums are assumed to be Big-Endian unless stated otherwise.  
All MSGPACK encoded datums should be preceeded with a single byte representing type of datum followed by a 2 byte unsigned integer indicating the length of the datum in bytes.


Start with a flexim header:`\xa4FLEX`

That's 5 bytes: 0xA4, 'F', 'L', 'E', 'X'.

This also decodes as a msgpack string: "FLEX" (0xA4 means 4-character string). If this header is not received by the server at the very begining, it will abort the connection (TCP RST).


To continue the connection process send an "AUTH" Command datum:
```rust,no-run
struct Command {
    cmd: "AUTH"
    payload: [PUBLIC_KEY, optional_alias], // The key you'd like to AUTH yourself for, sent as hex string in index 0
}
```

The server will (ideally) immediately respond with an Auth datum to challenge you, all field encrypted by requested public key:
```rust,no-run
struct Auth {
    date: 1539890190,
    challenge: "V8PmITTU3DGtYHTL1kUz",
    last_seen: 1539890190,
}
```
To which you will respond with an AuthResponse:
```rust,no-run
struct AuthResponse {
    challenge: "V8PmITTU3DGtYHTL1kUz", // This string will be encrypted with server public key.
}
```

If all went according to plan the server will not terminate your connection. Any datum may follow from this point as connection and authentication has been established.

Messages
--------
As far as Message processing goes, beyond the "to" and "from" field, it is almost entirely client-side. The server only cares where to send it and should not read or modify the messages in any way whatsoever. 

Here is a small example to illustrate how sending of messages looks.

```rust,no-run
struct Message {
    to: "Bob",
    from: "Alice",
    flags: ["NOTIFY"], ", // Strings here so it can be easily extended clientside.
    date: 1539890190,          // SystemTime::now() - UNIX_EPOCH maybe?
    msg: "HEY, LISTEN!",
}
```
Above, Alice has sent a message to Bob utilizing a custom client feature called "NOTIFY" that prompts a more "urgent" version of a message to the other client. Not all clients must support this, but the "HEY, LISTEN!" message goes through regardless as it is forwarded by the server. The extra urgency is pure client-side.
