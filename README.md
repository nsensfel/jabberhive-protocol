# JabberHive Protocol - Version 1
Additional content may be added to this specification should ambiguities need to
be removed, but the set of Inter-component Communication it defines below
is final and constitutes the version 1 of the protocol.

## Table of Contents
1. [Introduction] (#introduction)
2. [Components] (#components)
3. [Inter-component Communication] (#inter-component-communication)
4. [Examples] (#examples)
5. [Copyright] (#copyright)

## 1. Introduction
* *Make each program do one thing well. To do a new job, build afresh rather
   than complicate old programs by adding new "features".*
* *Expect the output of every program to become the input to another, as yet
   unknown, program.*

\- Doug MCIlroy

* *Write programs that do one thing and do it well.*
* *Write programs to work together.*
* *Write programs to handle text streams, because that is a universal
   interface.*

\- Peter H. Salus

Such is the base of the Unix Philosophy, and such is what tries to follow the
JabberHive protocol. Indeed, there may are plenty of Markov Chain Chatbots
available, but nearly all of them are made of a single program, for a single
communication protocol, with optional features making their code grow in
complexity. Instead of a single program, a JabberHive Chatbot is a network of
interconnected Components that all use the same text-based protocol. This lets
developer add functionalities without having to even look at the existing code,
by simply making a new component (in pretty much any programming language).

## 2. Components
There are three types of components: servers, filters, and gateways.

Generally, servers handle requests, filters transform that pass through, and
gateways generate the requests.

The number of connections accepted by (and initialized by) components may be
higher than one. Refer to a component's manual to know how it embeds itself in
the network.

### Server
Must accept at least one connection.
Those components are assumed to be the knowledge database. This is where the
actual reply creation, and the learning, are done.

### Filter
Must accept, and generate, at least one connection.

### Gateway
Must generate at least one connection. Gateways allow other communication
protocols to communicate with the JabberHive network. Those are expected to be
the main initiator of requests.

## 3. Inter-component Communication
All messages are text based. Messages structure is as follows:
```
++++++++++++++++++++++++++++++
+ TAG + ' ' + CONTENT + '\n' +
++++++++++++++++++++++++++++++
```
* TAG is not allowed to contain white spaces.
* TAG and CONTENT are not allowed to contain '\n'.

Any request leads to at least one reply. Indeed, each sequence of replies is
terminated by either a "Positive" or a "Negative" reply.

A connection between two components is directional in the sense that
only one of the two component makes requests and the other one gives replies.

Unless Pipelining is supported by the component to which requests are made,
simultaneous requests on a connection are not permitted: the request making
component must wait for its previous request to be completed before making a
new one.

Exchanges should be considered as happening between two directly connected
components, not from a gateway to a server through filters. Indeed, filters
may generate any number of requests of their own to handle an incoming request,
as long as it is transparent to the component that made said incoming request.

Even if there is no CONTENT, the TAG and '\n' are separated by a space.
"Any string" includes empty string. Strings are not null terminated.

### Requests
#### Request Protocol Version
* **TAG:** `?RPV`.
* **CONTENT:** List of comma separated unsigned integers, in a ascending order.
* **Example(s):** `?RPV 1,66,2910`, `?RPV 1`.

Upon reception of one such message, one should start using the latest JabberHive
protocol version listed in CONTENT that one can. If none are supported, a
"Negative" reply is sent. Otherwise, a "Confirm Protocol Version" message is
sent, followed by a "Positive" reply.

#### Request Pipelining Support
* **TAG:** `?RPS`
* **CONTENT:** None.
* **Example(s):** `?RPS `.

Asks the component if it supports handling simultaneous requests from the same
connection. A "Confirm Pipelining Support" reply is returned, followed by a
"Positive reply" (regardless of whether pipelining is supported or not).

#### Request Learn
* **TAG:** `?RL`
* **CONTENT:** Any string.
* **Example(s):** `?RL Anything the bot may have to learn.`, `?RL Other.`.

Requests the learning of CONTENT. Response is either a "Positive" message, or a
"Negative" message.

#### Request Reply
* **TAG:** `?RR`
* **CONTENT:** Any string.
* **Example(s):** `?RR Generated replies will be based on this.`, `?RR hello`.

Requests the generation of a reply to CONTENT. Response is be either a
"Positive" message preceded by a "Generated Reply" message, or a "Negative"
message.

#### Request Learn & Reply
* **TAG:** `?RLR`
* **CONTENT:** Any string.
* **Example(s):** `?RLR Generated replies will be based on this.`,
   `?RLR Other.`.

Requests the learning of CONTENT, followed by the generation of a reply.
Response is either a "Positive: message preceded by a "Generated Reply" message,
or a "Negative" message.

### Replies
#### Confirm Protocol Version
* **TAG:** `!CPV`
* **CONTENT:** Unsigned integer.
* **Example(s):** `!CPV 1`, `!CPV 29110`.

Indicates the JabberHive protocol version that will henceforth be used.

#### Confirm Pipelining Support
* **TAG:** `!CPS`
* **CONTENT:** 1 or 0.
* **Example(s):** `!CPS 0`, `!CPS 1`.

Indicates that the component supports (if CONTENT is set to 1), or does not
support (if CONTENT is set to 0) handling simultaneous requests from the
same connection.

#### Generated Reply
* **TAG:** `!GR`
* **CONTENT:** Any string.
* **Example(s):** `!GR I am not a robot.`, `!GR hi dan`.

This is a string that has been generated because of the previous request.

#### Additional Information
* **TAG:** `!AI`
* **CONTENT:** Any string.
* **Example(s):** `!AI [E] Would overflow.`, `!AI [W] Could not store data.`.

Allows the transmission of information for debugging purposes.

#### Positive
* **TAG:** `!P`
* **CONTENT:** None.
* **Example(s):** `!P `.

Previous request has been completed (no more messages related to it will be
sent).

#### Negative
* **TAG:** `!N`
* **CONTENT:** None.
* **Example(s):** `!N `.

Previous request could not be completed (no more messages related to it will be
sent).

### Dealing With Unknown Tags
| Component   | Request   | Reply     |
| ----------- | :-------: | :-------: |
| **Server**  | Reply !N  | N/A       |
| **Filter**  | Propagate | Propagate |
| **Gateway** | N/A       | Discard   |

## 4. Examples
### Networks
```
++++++++++++++++   +++++++++++++++++
+ SERVER (RAM) +---+ GATEWAY (IRC) +
++++++++++++++++   +++++++++++++++++
```
```
                     +++++++++++++++++      +++++++++++++++++++++
                     + GATEWAY (CLI) +      + GATEWAY (DISCORD) +
                     +++++++++++++++++      +++++++++++++++++++++
                             |                     |
                             |                     |
                             |                     |
++++++++++++++++   ++++++++++++++++++++   ++++++++++++++++++++
+ SERVER (RAM) +---+ FILTER (STORAGE) +---+ FILTER (LIMITER) +
++++++++++++++++   ++++++++++++++++++++   ++++++++++++++++++++
       |                                           |
       |                                           |
       |                                           |
+++++++++++++++++++++                      +++++++++++++++++
+ GATEWAY (STORAGE) +                      + GATEWAY (IRC) +
+++++++++++++++++++++                      +++++++++++++++++
```
```
++++++++++++++++
+ SERVER (RAM) +---|
++++++++++++++++   |
                   |
++++++++++++++++   ++++++++++++++++++++++++++   +++++++++++++++++
+ SERVER (RAM) +---+ FILTER (ROUND ROBIN)   +---+ GATEWAY (CLI) +
++++++++++++++++   ++++++++++++++++++++++++++   +++++++++++++++++
                   |
++++++++++++++++   |
+ SERVER (RAM) +---|
++++++++++++++++
```
