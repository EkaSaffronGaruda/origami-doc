# Preamble

This protocol is known as Origami. I designed it because I wanted something for data transmission that supports multiplexing over a single TCP connection, and this protocol is exactly for that purpose. It supports data transmission over multiple channels, bidirectional streams.

# Definitions

## Server and Client

"Server" and "Client" are variable. As this protocol is bidirectional, regardless of which party establishes the connection, "Server" and "Client" depends on the header type.

# 1. Header Format

## 1.1. Client Header

```
 <--------------- 6 bytes --------------->
 <- 2 bytes ->    <- 2 byte -> <- 2 byte ->
+---------------+------------+------------+
| TYPE          | VERSION    | SIZE       |
+---------------+------------+------------+
 <-------------- 16 bytes -------------->
+-----------------------------------------+
| Channel ID                              |
+-----------------------------------------+
 <--------------- <=0xff ---------------->
+-----------------------------------------+
| DATA                                    |
| ....                                    |
+-----------------------------------------+
```

## 1.2. Server Header

```
 <---------- 4 bytes ----------->
 <- 2 bytes ->   <- 2 bytes ->     <- 0 ->
+-------------+-----------------+---------+
| STAT        | SIZE            | 0       |
+-------------+-----------------+---------+
 <------------ 16 bytes ---------------->
+-----------------------------------------+
| Channel ID                              |
+-----------------------------------------+
 <--------------- <=0xff ---------------->
+-----------------------------------------+
| DATA                                    |
| ....                                    |
+-----------------------------------------+
```

## 1.3. Data Block
```
 <---------------- <= 65535 bytes ----------------->
 <- 32 bytes ->   <- 4 bytes ->  <- <=65499 bytes ->
+----------------+--------------+-------------------+
| Checksum       | SIZE         | MESSAGE           |
+----------------+--------------+-------------------+
```

- Checksum is any 256 bit checksum, generally sha256.
- Data Block SIZE is the size of the actual data being transmitted.
- MESSAGE is the actual data being transmitted.

# 2. Header fields

## 2.0 General Fields

### 2.0.1 SIZE

`u16` value. SIZE field is to define the size of the packet body, max size is 0xff, i.e. 65535.

### 2.0.2 Channel ID

Channel ID field is simply used to specify which channel this buffer should go to. Channel ID is obtained by a Create Channel action type. If no channel is created, which is normal during the first few stages, the parties may use white spaces or zeroes for padding.

## 2.1. Client Request Fields

### 2.1.1. TYPE

TYPE field is to define what type of action the client wants the server to perform. Below is an table of system action types.
| Hex | Name |
| :---: | :-------------------: |
| 0x00 | Ping |
| 0x01 | Request Handshake |
| 0x02 | Channel Create |
| 0x03 | Channel Delete |
| 0x04 | Data Append |
| 0x05 | Begin TLS |

### 2.1.2. VERSION

VERSION field is simply to define the version of Origami in use.

### 2.1.3. SIZE

Refer to 2.0.1

### 2.1.4. Channel ID

Refer to 2.0.2

## 2.2. Server Answer Fields

### 2.1.1. STAT

Stands for STATUS, this indicates the status of the action asked by the other party. Here's a table of system statuses.
| Hex | Name |
| :---: | :-------------------: |
| 0x00 | NULL_STAT |
| 0x01 | ACK |
| 0x02 | DISREGARDED |
| 0x03 | ACCEPTED |
| 0x04 | REJECTED |
| 0x05 | AWAITING |
| 0x06 | KEEP_ALIVE |
| 0x07 | CLIENT_ERROR |
| 0x08 | SYSTEM_ERROR |
| 0x09 | CORRUPTED_DATA |

### 2.1.2. SIZE
Refer to 2.0.1

### 2.1.3. 0
Just a 0 field.

### 2.1.4. Channel ID
Refer to 2.0.2

# 3. Multiplexing Mechanism

> How does Origami achieve multiplexing?

Multiplexing of data transmission is achieved by "channels." Channels are where your data stream goes in, each stream having an unique channel for its bidirectional transmission. Channels are assigned UUIDs as mentioned in 2.1.5 and 4.

# 4. Channel Management

Channels are managed using Action Type Channel\*. These include 0x02,0x03.

## 4.1 Channel Create Action Type

# 5. Packet Structuring and Examples

## 5.0 General Concepts about Packet Structuring

### 5.0.1 Padding

Padding is used to fill up empty field(s) which have a fixed length, and only if there's no value for the respective field(s). The generally used character for padding is whitespaces and/or zeroes, however, depends on the implementation of the protocol.

## 5.1. Examples

### 5.1.1. Ping requests

The client packet should look like:

```
+---------------+------------+------------+
| 0x00          | 0.1        | [SIZE]     |
+---------------+------------+------------+

+-----------------------------------------+
| 000000000000000000000000000000000000    |
+-----------------------------------------+

+-----------------------------------------+
| [DATA] OR NULL                          |
+-----------------------------------------+
```

and the server packet should look like

```
+---------------+------------+------------+
| 0x00          | [SIZE]     | 0          |
+---------------+------------+------------+

+-----------------------------------------+
| 000000000000000000000000000000000000    |
+-----------------------------------------+

+-----------------------------------------+
| [DATA] OR NULL                          |
+-----------------------------------------+
```

## 5.1.2. Handshakes

The client packet should look like:

```
+---------------+------------+------------+
| 0x01          | 0.1        | [SIZE]     |
+---------------+------------+------------+

+-----------------------------------------+
| 000000000000000000000000000000000000    |
+-----------------------------------------+

+-----------------------------------------+
# This may be used to supply additional   |
# information such as identifiers, etc.   |
| [DATA] OR NULL                          |
+-----------------------------------------+
```

### 5.1.2.1. Server acknowledges the handshake

```
+---------------+------------+------------+
| 0x01          | [SIZE]     | 0          |
+---------------+------------+------------+

+-----------------------------------------+
| 000000000000000000000000000000000000    |
+-----------------------------------------+

+-----------------------------------------+
# This may be used to supply additional   |
# information such as identifiers, etc.   |
| [DATA] OR NULL                          |
+-----------------------------------------+
```

### 5.1.2.2. Server disregards the handshake

```
+---------------+------------+------------+
| 0x02          | [SIZE]     | 0          |
+---------------+------------+------------+

+-----------------------------------------+
| 000000000000000000000000000000000000    |
+-----------------------------------------+

+-----------------------------------------+
# This may be used to supply additional   |
# information on why the handshake was    |
# disregarded.                            |
| [DATA] OR NULL                          |
+-----------------------------------------+
```

### 5.1.3. Creating a Channel

This is how the client packet should look like

```
+---------------+------------+------------+
| 0x02          | 0.1        | [SIZE]     |
+---------------+------------+------------+

+-----------------------------------------+
| 000000000000000000000000000000000000    |
+-----------------------------------------+

+-----------------------------------------+
| [DATA] OR NULL                          |
+-----------------------------------------+
```

#### 5.1.3.1. Server accepts the request and creates a channel

On accepting the channel create request, the server should answer with:

```
+---------------+------------+------------+
| 0x03          | [SIZE]     | 0          |
+---------------+------------+------------+

+-----------------------------------------+
# [Generated Channel ID]                  #
| 95d31c8f-f19c-44f2-aa7e-066d71409688    |
+-----------------------------------------+

+-----------------------------------------+
| [DATA] OR NULL                          |
+-----------------------------------------+
```

With this, a channel is created with the ID `95d31c8f-f19c-44f2-aa7e-066d71409688`

#### 5.1.3.2. Server rejects the request

On rejecting the channel create request, the server should answer with:

```
+---------------+------------+------------+
| 0x04          | [SIZE]     | 0          |
+---------------+------------+------------+

+-----------------------------------------+
| 000000000000000000000000000000000000    |
+-----------------------------------------+

+-----------------------------------------+
| [DATA] OR NULL                          |
+-----------------------------------------+
```

## 5.1.4. Deleting a channel
The client should initiate the request like this:
```
+---------------+------------+------------+
| 0x03          | 0.1        | [SIZE]     |
+---------------+------------+------------+

+-----------------------------------------+
| [Channel ID]                            |
+-----------------------------------------+

+-----------------------------------------+
| [DATA] OR NULL                          |
+-----------------------------------------+
```

### 5.1.4.1. Server accepts the request
```
+---------------+------------+------------+
| 0x03          | [SIZE]     | 0          |
+---------------+------------+------------+

+-----------------------------------------+
| 000000000000000000000000000000000000    |
+-----------------------------------------+

+-----------------------------------------+
| [DATA] OR NULL                          |
+-----------------------------------------+
```

### 5.1.4.2. Server rejects the request
The server may reject the request due to reasons like the channel already not existing, etc.
```
+---------------+------------+------------+
| 0x04          | [SIZE]     | 0          |
+---------------+------------+------------+

+-----------------------------------------+
| 000000000000000000000000000000000000    |
+-----------------------------------------+

+-----------------------------------------+
| [DATA] OR NULL                          |
+-----------------------------------------+
```

## 5.1.5. Writing to a channel
To write to a channel, the Channel ID must not be null.

The packet should look like:
```
+---------------+------------+------------+
| 0x04          | 0.1        | [SIZE]     |
+---------------+------------+------------+

+-----------------------------------------+
| [Channel ID]                            |
+-----------------------------------------+

+-----------------------------------------+
| [DATA]                                  |
+-----------------------------------------+
```

In this case, the conditions for the request to be accepted are as follows:
- `Channel ID` must not be null
- `SIZE` must be greater than 0 and less than 65535

These conditions must be filled or else the server will produce a client error and ignore the stream.

Upon successful reading of the buffer, the server shall answer with `ACCEPTED` status.

In case of corruption, refer to 5.1.6.

## 5.1.6. Data corruption status
Upon detecting a corrupt data packet, the receiving party answers with a `CORRUPTED_DATA` status, along with the checksum provided by the transmitting party, such as:
```
+---------------+------------+------------+
| 0x09          | [SIZE]     | 0          |
+---------------+------------+------------+

+-----------------------------------------+
| [Channel ID]                            |
+-----------------------------------------+

+-----------------------------------------+
| [Checksum]  | 0           | [0]         |
+-----------------------------------------+
```

which in return the transmitting party should send:
```
+---------------+------------+------------+
| 0x04          | 0.1        | [SIZE]     |
+---------------+------------+------------+

+-----------------------------------------+
| [Channel ID]                            |
+-----------------------------------------+

+-----------------------------------------+
| [DATA]                                  |
+-----------------------------------------+
```

# 6. Corruption of Data - Prevention and Recovery
Data corruption might happen, so this is how origami is supposed to prevent data corruption, and recover in case of corruption:
## 6.1. Prevention by Checksum
Receiving party, when data block is not null, must expect two fixed-length fields, `Checksum` and `SIZE` in the Data block, as mentioned in 1.3. `Checksum` is generally a SHA256 hash of the buffer while `SIZE` mentions the size of the data being transmitted.

## 6.2. Recovery Request
Receiving party, upon detecting a corruption, must answer with a `CORRUPTED_DATA` status, indicating the transmitting party to retransmit the data. Refer to 5.1.6.


# 7. Flow Control

To avoid congestion, Origami has a header field "SIZE" as mentioned in 2.0.1. SIZE field takes in 4 bytes (integer) and cannot be more than 0xff, i.e. 65535, as maximum body size can be 65535 to avoid congestion over channels. This means, on one go, at max of approximately 64KiB can be transmitted.