seconn
===========

seconn is part of SeConn project. It's a protocol and set of libraries for secure communication. This repository contains design and protocol description. See also other repositories:

* [seconn-avr](https://github.com/kacperzuk/seconn-avr) - AVR library that implements the SeConn protocol
* [seconn-java](https://github.com/kacperzuk/seconn-java) - Java library that implements the SeConn protocol
* [seconn-android-example](https://github.com/kacperzuk/seconn-android-example) - Example Android project that uses seconn-java
* [seconn-arduino-example](https://github.com/kacperzuk/seconn-arduino-example) - Example Arduino sketch that uses seconn-avr

Proto v1
====

General frame
-----

Message Length is Little-Endian.

```
Protocol Version (2B, 0x0001 here)
Message Type (1B)
Message Length in Bytes (2B)
Message (MessageLengthBytes)
```

Maximum message length is 1056 bytes (0x04 0x20).

Establishing connection
-----

First both send HelloRequest (0x00) with payload containing the public key (secp256r1) of sender (64 bytes, first 32B of X coordinate, then 32B of Y coordinate, little endian).

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

ECDH secret is established using public key from HelloRequest. It is then hashed with sha256. First 128b of hash are used as AES-128 encryption key (CBC), second 128b of hash are used as AES-128 CBC-MAC key.

HelloRequest must be replied with HelloResponse (0x01). Before sending HelloResponse we must send our own HelloRequest. HelloResponse has payload containing:
1. a CBC-MAC signature of ciphertext that follows (128b; Encrypt-then-MAC; no padding ne2eded, as ciphertext will always be multiple of 128b)
2. encrypted public key of HelloResponse sender (ciphertext), See Message Encryption for details.

If HelloResponse is received and receiver is able to successfully verify MAC, decrypt message and obtained public key is equal to public key from HelloRequest, we consider the connection established and authenticated.

Sending data
----

Message type is EncryptedData (0x02). Payload contains:

1. a CBC-MAC signature of ciphertext that follows (128b; Encrypt-then-MAC)
2. Encrypted message (ciphertext) (max 1040B). See Message Encryption for details.

Message encryption
----

plaintext can have max 1023Bt

1. prepend plaintext with 128 bits of random data (that way initialization vector doesn't matter)
2. pad plaintext according to PKCS7
3. encrypt prepended, padded plaintext using CBC, AES-128

Receiving message
----

When EncryptedData (0x02) frame is received, the receiver should:

1. verify that payload has length that is a multiply of 128b (otherwise silently discard)
1. verify that the CBC-MAC signature (first 16B) is valid for the whole remainder of payload (ciphertext) (otherwise silently discard this frame)
2. decrypt ciphertext into plaintext
3. drop first block of plaintext (16B)
4. read value of last byte of plaintext (PN)
5. verify that last PN bytes of plaintext have value PN (otherwise silently discard this frame)
6. drop last PN bytes of plaintext
7. ...
8. profit!
