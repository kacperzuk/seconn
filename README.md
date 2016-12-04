seconn
===========

seconn is part of SeConn project. It's a protocol and set of libraries for secure communication. This repository contains design and protocol description. See also other repositories:

* [seconn](https://github.com/kacperzuk/seconn) - description of design and protocol, you should read it.
* [seconn-avr](https://github.com/kacperzuk/seconn-avr) - AVR library that implements the SeConn protocol
* [seconn-java](https://github.com/kacperzuk/seconn-java) - Java library that implements the SeConn protocol
* [seconn-android-example](https://github.com/kacperzuk/seconn-android-example) - Example Android project that uses seconn-java
* [seconn-arduino-example](https://github.com/kacperzuk/seconn-arduino-example) - Example Arduino sketch that uses seconn-avr

Design goals
============

1. usable with hardware-constained embedded devices
2. network agnostic
3. provides authentication and encryption

Protocol
====

General frame
-----

Every frame follows this structure:

```
Protocol Version (2B, 0x0001)
Message Type (1B)
Message Length in Bytes (2B)
Message (MessageLengthBytes)
```

Message Length is Little-Endian.  
Maximum message length is 65535B (0xFFFF), but it can be limited by implementations (especially implementations for embedded devices).

Establishing connection
-----

```
Connection initiator                   Connection receiver
        +                                       +
        |   HelloRequest                        |
        | +-----------------------------------> |
        |                                       |
        |                                       |
        |                        HelloRequest   |
        | <-----------------------------------+ |
        |                                       |
        |                                       |
        |   HelloResponse                       |
        | +-----------------------------------> |
        |                                       |
        |                                       |
        |                       HelloResponse   |
        | <-----------------------------------+ |
        +                                       +
```

First a HelloRequest (0x00) with payload containing the public key (secp256r1) of sender (64 bytes, first 32B of X coordinate, then 32B of Y coordinate, little endian) is sent.

Example HelloRequest frame:

```
0x00 0x01 # protocol version
0x00      # message type HelloRequest
0x00 0x40 # length of 64 bytes
# 32 bytes of X coordinate of public key, little endian
0x12 0x34 0x56 0x78 0x9A 0xBC 0xDE 0xF0
0x12 0x34 0x56 0x78 0x9A 0xBC 0xDE 0xF0
0x12 0x34 0x56 0x78 0x9A 0xBC 0xDE 0xF0
0x12 0x34 0x56 0x78 0x9A 0xBC 0xDE 0xF0
# 32 bytes of Y coordinate of public key, little endian
0x12 0x34 0x56 0x78 0x9A 0xBC 0xDE 0xF0
0x12 0x34 0x56 0x78 0x9A 0xBC 0xDE 0xF0
0x12 0x34 0x56 0x78 0x9A 0xBC 0xDE 0xF0
0x12 0x34 0x56 0x78 0x9A 0xBC 0xDE 0xF0
```

Other side replies with HelloRequest containing their public key.

ECDH secret is established using public key from HelloRequest. It is then hashed with sha256. First 16B of hash are used as AES-128 encryption key (CBC and encrypting last block in ECBC-MAC), second 16B of hash are used as AES-128 CBC-MAC key.

HelloResponse (0x01) has payload containing:

1. a ECBC-MAC signature of ciphertext that follows (16B; Encrypt-then-MAC; no padding needed as ciphertext will always be multiple of 16B)
2. encrypted public key of HelloResponse sender (that's the signed ciphertext), See Message Encryption for details.

If HelloResponse is received and receiver is able to successfully verify MAC, decrypt message and obtained public key is equal to public key from HelloRequest, we consider the connection established and authenticated.

Sending data
----

Message type is EncryptedData (0x02). Payload contains:

1. a ECBC-MAC signature of ciphertext that follows (16B; Encrypt-then-MAC, no padding)
2. Encrypted message (ciphertext). See Message Encryption for details.

Message encryption
----

1. prepend plaintext with one block (16B) of random data (that way initialization vector doesn't matter or is sent with data, depends how you look at this)
2. pad plaintext according to PKCS7 (if `N` bytes must be inserted, insert `N` bytes of value `N`)
3. encrypt prepended, padded plaintext using CBC, AES-128. IV doesn't matter, you can use anything.

Receiving message
----

When EncryptedData (0x02) frame is received, the receiver should:

1. verify that payload has length that is a multiply of 16B (otherwise silently discard)
1. verify that the ECBC-MAC signature (first 16B) is valid for the remainder of payload (ciphertext) (otherwise silently discard this frame)
2. decrypt ciphertext using CBC decryption with AES
3. drop first block of plaintext (16B)
4. read value of last byte of plaintext (PN)
5. verify that last PN bytes of plaintext have value PN (otherwise silently discard this frame)
6. drop last PN bytes of plaintext
7. return plaintext to app
