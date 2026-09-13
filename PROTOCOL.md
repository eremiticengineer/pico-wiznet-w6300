# pico_wiznet_w6300

A Pico SDK C++ library for using the **WIZnet W6300 Ethernet controller** on the **W6300-EVB-Pico2** with HTTPS via mbedTLS.

The library splits the network stack between the RP2350 and the W6300:

* the **RP2350** handles HTTP, TLS, certificate validation, cryptography, and application logic
* the **W6300** handles TCP, IP, ARP, Ethernet framing, and the physical Ethernet interface

The W6300 supports both IPv4 and IPv6. The current implementation of this library uses the IPv4 path through DHCP, DNS, and the WIZnet socket API.

The W6300 does not understand HTTPS or TLS.

It provides a reliable TCP byte stream over Ethernet, while mbedTLS on the RP2350 performs the TLS protocol and encrypts/decrypts the application data.

---

## Architecture

A typical HTTPS request flows through the system like this:

```text
Application code on RP2350
        |
        v
HTTP / JSON
        |
        v
mbedTLS
        |
        v
tls_send() / tls_recv()
        |
        | QSPI via RP2350 PIO
        v
W6300 socket TX/RX buffers
        |
        v
TCP
        |
        v
IPv4
        |
        v
Ethernet MAC
        |
        v
Ethernet PHY
        |
        v
Magnetics / RJ45 / CAT cable
        |
        v
Network / Internet
        |
        v
Remote server
```

The W6300 itself is capable of IPv4 and IPv6 dual-stack networking.

The current project uses IPv4, so the examples below follow the actual IPv4 path used by the library.

---

## OSI Model Mapping

The OSI model is not a perfect one-to-one description of the modern TCP/IP stack, particularly at Layers 5 and 6, but it is useful for visualising how the system works.

| OSI layer        | This project                                             |
| ---------------- | -------------------------------------------------------- |
| 7 - Application  | HTTP + JSON                                              |
| 6 - Presentation | TLS encryption / mbedTLS                                 |
| 5 - Session      | Effectively handled within TLS, TCP, and the application |
| 4 - Transport    | TCP - W6300 hardware                                     |
| 3 - Network      | IPv4 - W6300 hardware; W6300 also supports IPv6          |
| 2 - Data Link    | Ethernet MAC - W6300 hardware                            |
| 1 - Physical     | 10/100 Ethernet PHY - W6300 + magnetics + cable          |

---

# Opening the TCP Connection

Before TLS begins, the W6300 opens a plain TCP socket:

```cpp
socket(TLS_SOCKET, Sn_MR_TCP, 0, 0);

connect(TLS_SOCKET, ip, port);
```

The W6300 then performs the TCP connection setup with the remote server:

```text
W6300                              Server

    SYN -------------------------->

        <------------------ SYN + ACK

    ACK -------------------------->
```

At this point the TCP connection is established.

Nothing is encrypted yet.

The W6300 knows about:

* local IP address
* local TCP port
* remote IP address
* remote TCP port
* TCP sequence numbers
* acknowledgements
* retransmissions

It does **not** know about:

* HTTPS
* TLS
* certificates
* public keys
* AES
* session keys
* HTTP
* JSON

Those belong to mbedTLS and the application running on the RP2350.

---

# Connecting mbedTLS to the W6300

mbedTLS is connected to the W6300 using transport callbacks:

```cpp
mbedtls_ssl_set_bio(
    &ssl,
    reinterpret_cast<void*>(
        static_cast<uintptr_t>(TLS_SOCKET)
    ),
    &WiznetW6300::tls_send,
    &WiznetW6300::tls_recv,
    nullptr
);
```

This tells mbedTLS:

* call `tls_send()` when TLS bytes need to be transmitted
* call `tls_recv()` when TLS bytes need to be received
* pass `TLS_SOCKET` to those callbacks as their context

The callbacks then call the WIZnet socket functions:

```cpp
send(socket_number, buf, len);

recv(socket_number, buf, len);
```

The resulting data path is:

```text
mbedTLS
   |
   | TLS protocol bytes
   v
tls_send()
   |
   v
WIZnet send()
   |
   | QSPI via RP2350 PIO
   v
W6300 TX buffer
   |
   v
TCP / IPv4 / Ethernet
```

Incoming data follows the reverse path.

A useful point here is that `send()` and `recv()` are WIZnet socket API functions.

They hide the W6300's registers, internal socket RAM, and QSPI host interface from the application.

From the point of view of mbedTLS, they simply provide a way to send and receive bytes over an already-established TCP connection.

---

# HTTPS Data Flow

## Layer 7 - HTTP and JSON

Suppose the application creates an HTTP request containing JSON:

```http
POST /weather HTTP/1.1
Host: example.com
Content-Type: application/json
X-API-KEY: example-key
Content-Length: ...

{"temperature":12.4}
```

At this point the request is simply plaintext bytes in RP2350 RAM.

Conceptually:

```text
JSON
  |
  v
HTTP request
```

No Ethernet or TCP packet exists yet at the application layer.

---

## Layer 6 - TLS

The request bytes are passed to:

```cpp
mbedtls_ssl_write(...);
```

By this point the TLS handshake has already completed, so mbedTLS knows how to protect the HTTP request.

mbedTLS turns the plaintext HTTP request into one or more encrypted TLS records:

```text
+----------------------+
| TLS record header    |
+----------------------+
| encrypted HTTP data  |
| ******************** |
| ******************** |
+----------------------+
```

To the W6300, the encrypted TLS record is simply a blob of bytes.

The W6300 does not know that those bytes contain HTTP.

It also does not know the encryption keys used to create them.

---

# Layer 4 - TCP

The encrypted TLS bytes reach the W6300 through `send()` and are placed into the W6300's TX socket buffer.

The W6300 TCP engine may divide a larger TLS record across several TCP segments.

For example:

```text
Segment 1

+----------------+
| TCP header     |
+----------------+
| payload        |
+----------------+


Segment 2

+----------------+
| TCP header     |
+----------------+
| remaining data |
+----------------+
```

The TCP header contains information such as:

* source port
* destination port
* sequence number
* acknowledgement number
* flags
* receive window
* checksum

Conceptually:

```text
+-------------------------+
| TCP header              |
+-------------------------+
| encrypted TLS record    |
+-------------------------+
```

The protocol nesting is therefore:

```text
JSON
  |
  v
HTTP
  |
  v
TLS
  |
  v
TCP
```

TCP has no knowledge of HTTP semantics.

It simply provides a reliable, ordered stream of bytes.

---

# Layer 3 - IP

In the current project, the W6300's hardware TCP/IP engine constructs an IPv4 packet containing the TCP segment:

```text
+-------------------------+
| IPv4 header             |
+-------------------------+
| TCP header              |
+-------------------------+
| TLS encrypted data      |
+-------------------------+
```

The IPv4 header contains information such as:

* source IP address
* destination IP address
* protocol = TCP
* TTL
* packet length
* header checksum

The W6300 also supports IPv6.

If the application were extended to use its IPv6 capabilities, the equivalent packet would instead contain an IPv6 header:

```text
+-------------------------+
| IPv6 header             |
+-------------------------+
| TCP header              |
+-------------------------+
| TLS encrypted data      |
+-------------------------+
```

The HTTPS/TLS architecture above this layer would remain essentially unchanged.

mbedTLS does not care whether the underlying TCP connection travelled over IPv4 or IPv6.

---

# Layer 2 - Ethernet

The W6300 places the IP packet inside an Ethernet frame:

```text
+--------------------------+
| Ethernet header          |
+--------------------------+
| IPv4 header              |
+--------------------------+
| TCP header               |
+--------------------------+
| TLS encrypted data       |
+--------------------------+
| Ethernet FCS             |
+--------------------------+
```

The Ethernet header contains:

* destination MAC address
* source MAC address
* EtherType

For IPv4:

```text
EtherType = 0x0800
```

For IPv6:

```text
EtherType = 0x86DD
```

The destination MAC address is normally **not** the MAC address of the remote web server.

MAC addresses are local to a Layer 2 network.

For a server elsewhere on the Internet, the frame is normally sent to the local router:

```text
Destination IP:

203.0.113.20


Destination Ethernet MAC:

router MAC address
```

For the IPv4 case used by this project, the W6300 uses ARP to determine the MAC address associated with the local gateway:

```text
192.168.1.1 -> AA:BB:CC:DD:EE:FF
```

The IP address identifies the final network destination.

The Ethernet MAC address identifies the next Layer-2 destination on the local link.

---

# Layer 1 - Ethernet PHY

The W6300 includes the Ethernet PHY.

The PHY converts the digital Ethernet frame into the electrical signalling used on twisted-pair Ethernet.

Conceptually:

```text
W6300 digital logic
       |
       v
Ethernet MAC
       |
       v
Ethernet PHY
       |
       v
TX/RX differential signals
       |
       v
Ethernet magnetics
       |
       v
RJ45
       |
       v
CAT cable
```

The W6300 includes a 10/100 Ethernet PHY and supports Ethernet auto-negotiation.

---

# QSPI Between the RP2350 and W6300

QSPI is **not** part of the network OSI stack.

It is the local communication interface between the RP2350 and the W6300.

On the W6300-EVB-Pico2, WIZnet's Pico software uses the RP2350's **PIO hardware** to implement the W6300 host interface.

The W6300 supports:

```text
QSPI single mode
QSPI dual mode
QSPI quad mode
```

This project uses:

```text
QSPI_QUAD_MODE
```

Conceptually:

```text
RP2350
   |
   | PIO state machine
   |
   | QSPI
   v
W6300
```

The WIZnet driver uses this interface to:

* read W6300 registers
* write W6300 registers
* read socket RX buffers
* write socket TX buffers
* configure network parameters
* configure sockets
* inspect socket status

For example, when the application eventually performs:

```cpp
send(socket_number, buf, len);
```

the library stack ultimately transfers the data from RP2350 RAM into the W6300's internal socket TX memory.

Conceptually:

```text
RP2350 RAM
    |
    | QSPI via PIO
    v
W6300 TX buffer
    |
    | TCP/IP processing
    v
W6300 Ethernet MAC
    |
    v
W6300 Ethernet PHY
    |
    v
Ethernet cable
```

Similarly, received network data travels:

```text
Ethernet cable
    |
    v
W6300 PHY
    |
    v
W6300 TCP/IP engine
    |
    v
W6300 RX buffer
    |
    | QSPI via PIO
    v
RP2350 RAM
```

---

# Why the Function Is Still Called `wizchip_spi_initialize()`

The WIZnet Pico port exposes:

```cpp
wizchip_spi_initialize();
```

even when the W6300 is being operated through QSPI.

For the W6300 configuration, the port layer is compiled with the PIO/QSPI implementation.

Conceptually:

```text
wizchip_spi_initialize()
        |
        v
WIZnet Pico port
        |
        | USE_PIO
        v
wizchip_qspi_pio
        |
        v
RP2350 PIO
        |
        v
W6300 QSPI
```

The name therefore reflects the common WIZnet host-interface abstraction rather than meaning that the W6300-EVB-Pico2 is using one of the RP2350's ordinary hardware SPI peripherals for its actual W6300 communication.

---

# W6300 Socket Memory

The W6300 has eight independent hardware sockets.

It provides:

```text
32 KB TX socket memory
32 KB RX socket memory
----------------------
64 KB total socket memory
```

The WIZnet socket API abstracts this memory from the application.

For example:

```cpp
send(TLS_SOCKET, buf, len);
```

does not mean the RP2350 sends an Ethernet frame directly.

Instead, conceptually:

```text
RP2350 buffer
     |
     | QSPI
     v
W6300 socket TX RAM
     |
     v
W6300 TCP engine
     |
     v
IP
     |
     v
Ethernet
```

Likewise:

```cpp
recv(TLS_SOCKET, buf, len);
```

copies data from the W6300 socket RX buffer into RP2350 memory.

---

# Across the Network

Once the Ethernet frame leaves the device, it may pass through several network devices:

```text
RP2350
   |
   | QSPI via PIO
   v
W6300
   |
   | Ethernet
   v
network switch
   |
   v
home router
   |
   v
ISP router
   |
   v
Internet
   |
   v
web server
```

At every router, the Layer 2 framing can change while the Layer 3 IP packet continues towards its destination.

On the local LAN:

```text
[W6300 MAC -> router MAC]
    [IP: client -> server]
        [TCP]
            [TLS]
```

After the router forwards the packet through another network:

```text
[different Layer-2 framing]
    [IP: client -> server]
        [TCP]
            [TLS]
```

Layer 2 is hop-local.

Layer 3 allows the packet to travel between networks.

---

# Receiving the Request at the Server

At the destination, the process happens in reverse.

```text
electrical Ethernet signal
        |
        v
Ethernet frame
        |
        v
IP packet
        |
        v
TCP segment
        |
        v
TLS byte stream
        |
        v
TLS decrypts the data
        |
        v
HTTP request
        |
        v
application / API
```

The server eventually sees something like:

```http
POST /weather HTTP/1.1
Host: ...
X-API-KEY: ...
```

and the application can read the request body.

For example, in PHP:

```php
$raw_json = file_get_contents('php://input');
```

The PHP application never needs to know that the client happens to be an RP2350 connected to a W6300.

By the time the request reaches PHP, the web server's TLS and HTTP layers have already reconstructed the request.

---

# TLS Handshake

Once:

```cpp
connect(TLS_SOCKET, ip, port);
```

has succeeded, there is a normal TCP connection between the W6300 and the server.

Then:

```cpp
mbedtls_ssl_handshake(&ssl);
```

causes mbedTLS on the RP2350 to conduct the TLS handshake through that TCP connection.

The W6300 does not know that these bytes represent TLS.

For a modern TLS 1.3 connection, the handshake is approximately:

```text
RP2350 / mbedTLS                         Server

       |                                   |
       |        TCP already connected      |
       |<=================================>|
       |                                   |
       | ------ ClientHello -------------> |
       |                                   |
       | <----- ServerHello -------------- |
       | <----- EncryptedExtensions -------|
       | <----- Certificate ---------------|
       | <----- CertificateVerify ---------|
       | <----- Finished ------------------|
       |                                   |
       |    validate certificate           |
       |    derive session keys            |
       |                                   |
       | ------ Finished ----------------> |
       |                                   |
       | ===== encrypted application ===== |
       | ------ HTTP POST ---------------> |
       | <----- HTTP response ------------ |
       |                                   |
```

The exact handshake depends on the negotiated TLS version and cipher suite.

---

## 1. TCP Exists Before TLS Begins

The W6300 has already created the TCP connection:

```cpp
socket(TLS_SOCKET, Sn_MR_TCP, 0, 0);

connect(TLS_SOCKET, ip, port);
```

Nothing is encrypted yet.

TLS runs **on top of** this TCP connection.

Conceptually:

```text
RP2350 / mbedTLS
       |
       | TLS
       v
W6300 TCP socket
       |
       v
Network
```

---

## 2. `mbedtls_ssl_handshake()` Starts

mbedTLS already knows how to access the network because `mbedtls_ssl_set_bio()` has registered the W6300 send and receive callbacks.

Conceptually:

```text
mbedTLS
   |
   | TLS protocol bytes
   v
tls_send()
   |
   v
WIZnet send()
   |
   v
W6300 TCP
```

The reverse path is used for incoming handshake data.

---

## 3. ClientHello

The RP2350 sends the first TLS handshake message:

```text
ClientHello
```

mbedTLS constructs it.

Conceptually the client says:

> I want to establish a secure connection. Here are the TLS versions and cryptographic algorithms I support, together with the information required to establish keys.

A TLS 1.3 ClientHello can contain:

* supported TLS versions
* supported cipher suites
* supported cryptographic groups
* signature algorithms
* random data
* a key share
* SNI hostname
* other TLS extensions

The SNI hostname comes from:

```cpp
mbedtls_ssl_set_hostname(
    &ssl,
    host.c_str()
);
```

For example:

```text
weather.example.com
```

A conceptual ClientHello might look like:

```text
ClientHello

├── TLS versions
│   └── TLS 1.3
│
├── random
│
├── cipher suites
│   ├── TLS_AES_128_GCM_SHA256
│   └── ...
│
├── supported groups
│   ├── X25519
│   └── secp256r1
│
├── key_share
│   └── client's ephemeral public key
│
└── server_name
    └── weather.example.com
```

---

## 4. ClientHello Reaches the Server

mbedTLS serialises the ClientHello and calls:

```cpp
WiznetW6300::tls_send(...);
```

which ultimately calls:

```cpp
send(TLS_SOCKET, buf, len);
```

The path is:

```text
RP2350 RAM
   |
   | ClientHello bytes
   v
tls_send()
   |
   v
WIZnet send()
   |
   | QSPI via PIO
   v
W6300 TX buffer
   |
   v
TCP engine
   |
   v
IPv4
   |
   v
Ethernet
   |
   v
wire
```

The W6300 may split the data across one or more TCP segments.

It does not know that the data is a ClientHello.

As far as the W6300 is concerned, the application asked it to transmit a sequence of bytes reliably over TCP.

---

## 5. ServerHello

The server receives the ClientHello, selects compatible parameters, and sends:

```text
ServerHello
```

This communicates information such as:

* selected TLS version
* selected cipher suite
* server random data
* server ephemeral key share

Conceptually:

```text
ClientHello

"I support A, B and C.
 Here is my ephemeral public key."

             |
             v

ServerHello

"Use B.
 Here is my ephemeral public key."
```

At this point both sides have enough information to independently derive shared cryptographic secret material.

---

## 6. Ephemeral Key Agreement

With a common modern TLS 1.3 configuration, both sides generate ephemeral private/public key pairs.

Conceptually:

```text
Client:

    private key: a
    public key:  A


Server:

    private key: b
    public key:  B
```

The public values are exchanged.

The private values are never transmitted.

Each side combines its own private value with the other side's public value to derive the same shared secret.

Conceptually:

```text
client derives secret S

server derives secret S

network observer cannot feasibly derive S
```

The resulting encryption keys are not sent across Ethernet.

Both endpoints derive them independently.

The W6300 never receives either endpoint's private key.

---

## 7. TLS Derives Handshake Keys

TLS 1.3 does not use the raw shared secret directly as an encryption key.

Instead it passes cryptographic secret material through the TLS key schedule, based around HKDF.

Conceptually:

```text
shared secret
      |
      v
     HKDF
      |
      +--> client handshake traffic secret
      |
      +--> server handshake traffic secret
```

These secrets are used to derive encryption keys and IVs.

`ClientHello` and `ServerHello` are observable on the network.

Subsequent handshake messages are protected.

---

## 8. Server Sends Its Certificate Chain

The server sends its certificate chain.

The certificate contains identity and public-key information used to authenticate the server.

The certificate data arrives as TLS data over the TCP connection.

The W6300 does not parse or validate it.

That happens inside mbedTLS on the RP2350.

---

## 9. RP2350 Validates the Certificate

The project configures mbedTLS with trusted CA information:

```cpp
mbedtls_x509_crt_parse(...);

mbedtls_ssl_conf_ca_chain(...);

mbedtls_ssl_conf_authmode(...);

mbedtls_ssl_set_hostname(...);
```

The intention is not merely:

> Encrypt the connection.

It is also:

> Prove that the server is genuinely the server I intended to contact.

mbedTLS validates the certificate chain and hostname according to the configured trust information.

The W6300 plays no part in certificate verification.

---

## 10. CertificateVerify

In TLS 1.3 the server sends:

```text
CertificateVerify
```

This proves that the server possesses the private key associated with its authenticated identity.

Conceptually:

```text
handshake transcript
        |
        v
server signs transcript information
        |
        v
CertificateVerify
```

The client verifies that signature using the corresponding public key.

Again, all of this cryptography occurs on the RP2350 inside mbedTLS.

---

## 11. Server Finished

The server then sends:

```text
Finished
```

This proves that the server:

* participated in this exact handshake
* derived the expected handshake secrets
* saw the same handshake transcript

Conceptually:

```text
ClientHello
ServerHello
EncryptedExtensions
Certificate
CertificateVerify
        |
        v
cryptographic transcript hash
        |
        +
derived key
        |
        v
Finished
```

This protects the integrity of the handshake.

---

## 12. RP2350 Verifies Server Finished

The data arrives through:

```cpp
tls_recv()
```

and ultimately:

```cpp
recv(TLS_SOCKET, ...);
```

The full path is approximately:

```text
server
   |
   v
Ethernet
   |
   v
W6300
   |
   v
W6300 RX socket buffer
   |
   | QSPI via PIO
   v
tls_recv()
   |
   v
mbedTLS
```

mbedTLS decrypts the TLS record and verifies the expected Finished value.

If this verification fails, the TLS handshake fails.

---

## 13. RP2350 Sends Its Finished

The client sends its own:

```text
Finished
```

Conceptually it says:

> I derived the same secrets and observed the same authenticated handshake.

At this point both sides agree on:

```text
server identity               ✓
cryptographic algorithms      ✓
shared secret                 ✓
handshake integrity           ✓
traffic keys                  ✓
```

The TLS connection is established.

---

## 14. Application Traffic Keys

TLS derives keys for encrypted application data.

Conceptually:

```text
shared cryptographic state
            |
            v
           HKDF
            |
        +---+---+
        |       |
        v       v
 client app   server app
 traffic key  traffic key
```

The two directions use distinct cryptographic material:

```text
client -> server key

server -> client key
```

The session keys remain inside mbedTLS on the RP2350.

They are never given to the W6300.

---

## 15. Sending the HTTP Request

After:

```cpp
mbedtls_ssl_handshake(&ssl);
```

returns successfully, the application can call:

```cpp
mbedtls_ssl_write(...);
```

The data path becomes:

```text
HTTP plaintext
      |
      v
mbedTLS
      |
      | encrypt + authenticate
      v
TLS application-data record
      |
      v
tls_send()
      |
      | QSPI via PIO
      v
W6300
      |
      v
TCP / IPv4 / Ethernet
```

The W6300 only sees the TLS bytes.

It does not possess the TLS application traffic keys required to decrypt them.

---

# What a Packet Capture Can See

A packet capture on the Ethernet network can still see metadata such as:

```text
Ethernet

    source MAC
    destination MAC


IPv4

    source IP
    destination IP


TCP

    source port
    destination port 443


TLS

    encrypted application data
```

It can therefore infer information such as:

* client IP address
* server IP address
* TCP port
* packet sizes
* packet timing
* that TLS is being used

It cannot ordinarily see the protected application data, such as:

```text
POST /weather

X-API-KEY

JSON body

temperature value
```

Those values exist as plaintext inside the RP2350 before encryption and inside the server after decryption.

Between those endpoints they are protected by TLS.

---

# The W6300's View of the TLS Handshake

The W6300 has no knowledge of TLS semantics.

During the handshake, its view is approximately:

```text
ClientHello:

    "send these bytes"


ServerHello:

    "I received some bytes"


Certificate:

    "I received more bytes"


Key establishment:

    "I have no idea what these bytes mean"


HTTP POST:

    "send these bytes"
```

Its job is simply to provide a reliable, ordered TCP byte stream.

mbedTLS gives those bytes meaning.

---

# QSPI During the TLS Handshake

QSPI is used repeatedly while the handshake runs because TLS data must be transferred between RP2350 RAM and the W6300's socket buffers.

Outbound:

```text
mbedTLS on RP2350
        |
        | TLS handshake / encrypted bytes
        v
    tls_send()
        |
        v
   WIZnet send()
        |
        | QSPI via PIO
        v
    W6300 TX RAM
        |
        v
 TCP / IPv4 / Ethernet
        |
        v
      server
```

Inbound:

```text
server
   |
   v
Ethernet / TCP
   |
   v
W6300 RX RAM
   |
   | QSPI via PIO
   v
WIZnet recv()
   |
   v
tls_recv()
   |
   v
mbedTLS
   |
   | decrypt / verify / parse
   v
RP2350
```

During:

```cpp
mbedtls_ssl_handshake(&ssl);
```

there may be many calls to `tls_send()` and `tls_recv()`.

mbedTLS may return:

```cpp
MBEDTLS_ERR_SSL_WANT_READ
```

or:

```cpp
MBEDTLS_ERR_SSL_WANT_WRITE
```

Depending on how the transport callback and retry logic are implemented, these indicate that the TLS state machine needs additional transport I/O before it can make further progress rather than necessarily indicating a failed TLS connection.

---

# Complete TCP + TLS Flow

```text
            RP2350                        W6300                    SERVER

              |                             |                        |
              | socket/connect ------------>|                        |
              |                             |--- TCP SYN ----------->|
              |                             |<-- SYN/ACK ------------|
              |                             |--- ACK --------------->|
              |                             |                        |
              |                             |    TCP ESTABLISHED     |
              |                             |                        |
        mbedtls_ssl_handshake()             |                        |
              |                             |                        |
create        |                             |                        |
ClientHello   |                             |                        |
              | tls_send() ---------------->|                        |
              |    QSPI via PIO             |--- ClientHello ------->|
              |                             |                        |
              |                             |<-- ServerHello --------|
              |<--------------- tls_recv() -|                        |
              |    QSPI via PIO             |                        |
              |                             |                        |
        derive handshake secrets            |                        |
              |                             |                        |
              |                             |<-- Certificate --------|
              |<--------------- tls_recv() -|                        |
              |                             |                        |
        validate CA chain                   |                        |
        validate hostname                   |                        |
        verify server signature             |                        |
              |                             |                        |
              |                             |<-- Finished -----------|
              |<--------------- tls_recv() -|                        |
              |                             |                        |
        verify Finished                     |                        |
        derive application keys             |                        |
              |                             |                        |
              | tls_send() ---------------->|--- Finished ---------->|
              |                             |                        |
        TLS ESTABLISHED                     |                        |
              |                             |                        |
HTTP POST     |                             |                        |
plaintext     |                             |                        |
      |       |                             |                        |
      v       |                             |                        |
   encrypt    |                             |                        |
      |       |                             |                        |
      v       |                             |                        |
TLS ciphertext                              |                        |
              | tls_send() ---------------->|                        |
              |    QSPI via PIO             |--- TCP/TLS ---------->|
              |                             |                        |
              |                             |<-- encrypted HTTP -----|
              |<--------------- tls_recv() -|                        |
              |                             |                        |
        decrypt                             |                        |
              |                             |                        |
        HTTP response                       |                        |
```

The important point is that the horizontal boundary between the RP2350 and W6300 is a **host-controller interface**, not another layer of the Internet protocol stack.

The QSPI traffic consists of W6300 commands, register accesses, and socket-buffer transfers.

The actual TCP/IP/Ethernet processing happens inside the W6300.

---

# Division of Responsibility

## RP2350 / mbedTLS

The RP2350 is responsible for:

```text
HTTP

JSON

server identity

certificates

certificate validation

TLS state machine

cryptography

session keys

encryption

decryption
```

It creates plaintext HTTP messages and receives plaintext HTTP responses.

mbedTLS converts between those plaintext messages and protected TLS records.

---

## W6300

For the current IPv4 project, the W6300 is responsible for:

```text
TCP connection

sequencing

acknowledgements

retransmission

IPv4

ARP

Ethernet framing

MAC

PHY
```

The W6300 hardware also provides capabilities beyond those currently used by this particular library, including:

```text
IPv6

ICMPv4

ICMPv6

UDP

IGMP

MLD
```

It provides eight independent hardware sockets and dedicated TX/RX socket memory.

---

# The Most Important Separation

The most important architectural boundary is:

> **TLS session keys never travel through the W6300 or across the network.**

The RP2350 and server independently derive matching cryptographic secrets during the TLS handshake.

The W6300 merely transports the handshake messages and subsequent ciphertext reliably over TCP.

Conceptually:

```text
RP2350                                      Server

TLS keys                                     TLS keys
   |                                            |
   |                                            |
encrypt                                      decrypt
   |                                            |
   v                                            ^
ciphertext                                    ciphertext
   |                                            |
   v                                            |
W6300  ==================================>  network
```

The W6300 participates in the TCP/IP connection but not in the TLS security protocol.

---

# W6300 Compared with Wi-Fi + lwIP

With a Pico W or Pico 2 W using lwIP, considerably more of the TCP/IP stack runs in software around the RP2040/RP2350 and Wi-Fi driver.

With the W6300, much of the lower network stack is implemented by the Ethernet controller itself.

Conceptually:

```text
Pico 2 W / lwIP

RP2350
 |
 +-- application
 |
 +-- TLS
 |
 +-- TCP/IP software stack
 |
 +-- Wi-Fi networking
```

Compared with:

```text
W6300-EVB-Pico2

RP2350
 |
 +-- application
 |
 +-- TLS
 |
 +-- WIZnet socket API
        |
        | QSPI via PIO
        v
      W6300
        |
        +-- TCP
        +-- IPv4 / IPv6
        +-- Ethernet MAC
        +-- Ethernet PHY
```

That leaves the RP2350 primarily responsible for the application and security layers while the W6300 handles much of the lower transport, network, data-link, and physical networking functionality.

---

# W6300-EVB-Pico2 Summary

The resulting architecture can be summarised as:

```text
                    W6300-EVB-Pico2

+----------------------------------------------------------+
|                         RP2350                           |
|                                                          |
|   Application                                            |
|       |                                                  |
|       v                                                  |
|   HTTP / JSON                                            |
|       |                                                  |
|       v                                                  |
|   mbedTLS                                                |
|       |                                                  |
|       | TLS ciphertext                                   |
|       v                                                  |
|   tls_send() / tls_recv()                                |
|       |                                                  |
+-------|--------------------------------------------------+
        |
        | QSPI using RP2350 PIO
        |
+-------|--------------------------------------------------+
|       v                       W6300                      |
|                                                          |
|   socket TX/RX memory                                    |
|       |                                                  |
|       v                                                  |
|   TCP                                                    |
|       |                                                  |
|       v                                                  |
|   IPv4 / IPv6                                            |
|       |                                                  |
|       v                                                  |
|   Ethernet MAC                                           |
|       |                                                  |
|       v                                                  |
|   Ethernet PHY                                           |
|                                                          |
+-------|--------------------------------------------------+
        |
        | Ethernet
        v
      RJ45
        |
        v
      LAN
        |
        v
    Internet
        |
        v
   HTTPS server
```

For the current library, the active network path is:

```text
HTTP
  ↓
TLS / mbedTLS
  ↓
WIZnet TCP socket
  ↓
IPv4
  ↓
Ethernet
```

while the W6300 hardware itself is capable of extending the network layer to IPv6 without fundamentally changing the application's TLS architecture.
