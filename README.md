# AUTHOR IDENTITY PROTOCOL
> A simple and flexible method to sign arbitrary OP_RETURN data with Bitcoin ECDSA signatures.

AIP was designed to be a modular, and flexible signing protocol. This means it can be used in conjunction with other Bitcoin Data Protocols, like the [B protocol](https://b.bitdb.network/) by Unwriter

Authors: Attila Aros, Satchmo

Special thanks to Monkeylord, Unwriter, Libitx, and Gal Buki for feedback and ideas.

Note: Use the [bitcoinfiles-sdk](https://github.com/BitcoinFiles/bitcoinfiles-sdk#sign-and-create-file) to build, sign, and verify document signatures in Javascript, and [https://github.com/rohenaz/go-aip](go-aip) for Go.

# Intro

The design goals:

1. A simple protocol to sign arbitrary OP_RETURN data in a single transaction
2. Decouple the signing with an address from the funding source address (ie: does not require any on-chain transactions from the signing identity address)
3. Allow multiple signatures to be layered on top to provide multi-party contracts.
4. Flexible enough to support future signing schemes with similar structure

# Use Cases

- Prove ownership and authoring of any file
- Add multiple signatures to form agreements and contracts
- Sign data against a paymail identity
- Decouple identity from funding addresses

The last point of being able to decouple identity from the funding source addresses means that we can now upload files and content and not have to expose our identity with an on-chain payment transaction.

An example is being able to upload a blog post and using Money Button to pay for the mining fees, yet never exposing your Identity key with an on-chain payment.

# Protocol

- The prefix for AUTHOR IDENTITY Protocol is `15PciHG22SNLQJXMoSUaWVi7WSqc7hCfva`

Here's an example of what **POST transactions** look like:

```
OP_RETURN
  19HxigV4QyBv3tHpQVcUEQyq1pzZVdoAut
  [Data]
  [Media Type]
  [Encoding]
  [Filename]
  |
  15PciHG22SNLQJXMoSUaWVi7WSqc7hCfva
  [Signing Algorithm]
  [Signing Address]
  [Signature]
  [Field Index 0] // Optional. 0 based index means the OP_RETURN (0x6a) is signed itself
  [Field Index 1] // Optional.
  ...             // If the Field Indexes are omitted, then it's assumed that all fields, beginning with and including OP_RETURN, and continuing to (but not including) the AUTHOR_IDENTITY prefix are signed. This also includes the pipe (protocol separator) just before the AIP prefix.
```

An example with signing [B:// Bitcoin Data](https://github.com/unwriter/B) is shown, however any arbitrary OP_RETURN content can be signed provided that the fields being signed are before the AUTHOR IDENTITY `15PciHG22SNLQJXMoSUaWVi7WSqc7hCfva` prefix.

We use the [Bitcom](https://bitcom.bitdb.network) convention to use the pipe '|' to indicate the protocol boundary.

Fields:

1. **Signing Algorithm:** 

    - "BITCOIN_ECDSA" refers to [Bitcoin Signed Message protocol](https://docs.moneybutton.com/docs/bsv-message.html) which is the default Bitcoin signing algorithm built into bsv.js. UTF-8 encoding.
    - "BITCOIN_ECDSA_HASH" is the same as BITCOIN_ECDSA, except the data being signed is a SHA256 hash of the raw data.
    - "BITCOIN_ECDSA_PAYMAIL" refers to Bitcoin Signed Message protocol where the address field is a paymail address, consistent with [paymail signatures](https://docs.moneybutton.com/docs/mb-signatures.html). 
    
2. **Signing Address:** Bitcoin Address, Paymail address, or public key being used to sign the content (depending on signature algorithm being used). UTF-8 encoding.
3. **Signature:** The signature of the signed content with the Signing Address. Base64 encoding.
4. **Field Index <index>:** (Optional) The specific index (relative to Field Offset) that is covered by the Signature.  Non-negative integer hex encoding. When there are no indexes provided, it is assumed that all fields to the left of the AUTHOR IDENTITY prefix are signed.

_NOTE: THE 0'th INDEX IS THE OP_RETURN(0x6a), even when OP_FALSE (0x00) precedes OP_RETURN._

Example:

```
OP_RETURN
  19HxigV4QyBv3tHpQVcUEQyq1pzZVdoAut  // B Prefix
  { "message": "Hello world!" }       // Content
  applciation/json                    // Content Type
  UTF-8                               // Encoding
  0x00                                // File name (empty in this case with 0x00 to indicate null)
  |                                   // Pipe to seperate protocols
  15PciHG22SNLQJXMoSUaWVi7WSqc7hCfva, // AUTHOR IDENTITY prefix
  BITCOIN_ECDSA                       // Signing Algorithm
  1EXhSbGFiEAZCE5eeBvUxT6cBVHhrpPWXz, // Signing Address
  0x1b3ffcb62a3bce00c9b4d2d66196d123803e31fa88d0a276c125f3d2524858f4d16bf05479fb1f988b852fe407f39e680a1d6d954afa0051cc34b9d444ee6cb0af, // Signature
  0,  // OP_RETURN 6a
  1,  // 19HxigV4QyBv3tHpQVcUEQyq1pzZVdoAut
  2,  // { "message": "Hello world!" }
  3,  // applciation/json
  4,  // UTF-8
  5,  // 0x00
  6   // |
];

```

In Hex form:
```

[
  '0x6a', /// OP_RETURN
  '0x31394878696756345179427633744870515663554551797131707a5a56646f417574',
  '0x7b20226d657373616765223a202248656c6c6f20776f726c6421227d',
  '0x6170706c69636174696f6e2f6a736f6e',
  '0x7574662d38',
  '0x00',
  '0x7c',
  '0x313550636948473232534e4c514a584d6f5355615756693757537163376843667661',
  '0x424954434f494e5f4543445341',
  '0x31455868536247466945415a4345356565427655785436634256486872705057587a',
  '0x1b3ffcb62a3bce00c9b4d2d66196d123803e31fa88d0a276c125f3d2524858f4d16bf05479fb1f988b852fe407f39e680a1d6d954afa0051cc34b9d444ee6cb0af',
  '0x00',
  '0x01',
  '0x02',
  '0x03',
  '0x04',
  '0x05',
  '0x06'
]

```

In Hex form (by implicitly signing all fields)
```

[
  '0x6a', /// OP_RETURN
  '0x31394878696756345179427633744870515663554551797131707a5a56646f417574',
  '0x7b20226d657373616765223a202248656c6c6f20776f726c6421227d',
  '0x6170706c69636174696f6e2f6a736f6e',
  '0x7574662d38',
  '0x00',
  '0x7c',
  '0x313550636948473232534e4c514a584d6f5355615756693757537163376843667661',
  '0x424954434f494e5f4543445341',
  '0x31455868536247466945415a4345356565427655785436634256486872705057587a',
  '0x1b3ffcb62a3bce00c9b4d2d66196d123803e31fa88d0a276c125f3d2524858f4d16bf05479fb1f988b852fe407f39e680a1d6d954afa0051cc34b9d444ee6cb0af'
]

```

##### Paymail Example - Signing data against a paymail address

*In this example, indicies are ommitted, meaning we sign all data up to the AIP prefix, including the OP_RETURN and the "|", but not OP_FALSE*

Example:

```
OP_FALSE
OP_RETURN
  19HxigV4QyBv3tHpQVcUEQyq1pzZVdoAut  // B Prefix
  { "message": "Hello world!" }       // Content
  applciation/json                    // Content Type
  UTF-8                               // Encoding
  0x00                                // File name (empty in this case with 0x00 to indicate null)
  |                                   // Pipe to seperate protocols
  15PciHG22SNLQJXMoSUaWVi7WSqc7hCfva, // AUTHOR IDENTITY prefix
  PAYMAIL                             // Signing Algorithm
  satchmo@moneybutton.com             // Signing Address
  0x1b3ffcb62a3bce00c9b4d2d66196d123803e31fa88d0a276c125f3d2524858f4d16bf05479fb1f988b852fe407f39e680a1d6d954afa0051cc34b9d444ee6cb0af, // Signature
];

```

# Transaction Examples

##### 1 signature

File:

https://www.bitcoinfiles.org/db61b9a0a31142825a9f2f1c48543299f72c974b5e4c44335c4357abfdeac753


Transaction:

https://whatsonchain.com/tx/db61b9a0a31142825a9f2f1c48543299f72c974b5e4c44335c4357abfdeac753


##### 2 signatures

File:

https://www.bitcoinfiles.org/d4738845dc0d045a35c72fcacaa2d4dee19a3be1cbfcb0d333ce2aec6f0de311


Transaction:

https://whatsonchain.com/tx/d4738845dc0d045a35c72fcacaa2d4dee19a3be1cbfcb0d333ce2aec6f0de311


##### File with 1 signature, but using implicit 'sign all' by omitting indexes:

File:

https://www.bitcoinfiles.org/5633bb966d9531d22df7ae98a70966eebe4379d400d74ac948bf5b4f2867092c


Transaction:

https://whatsonchain.com/tx/5633bb966d9531d22df7ae98a70966eebe4379d400d74ac948bf5b4f2867092c


# Usage and Library Examples

*Create and Sign a File*

https://github.com/BitcoinFiles/bitcoinfiles-sdk/blob/master/test/create.js#L128

*Build and Sign a File*

https://github.com/BitcoinFiles/bitcoinfiles-sdk/blob/master/test/build.js#L16

*Build and Sign a File with 2 Signatures (Contract)*

https://github.com/BitcoinFiles/bitcoinfiles-sdk/blob/master/test/build.js#L209

*Verify Signature for OP_RETURN fields*:

https://github.com/BitcoinFiles/bitcoinfiles-sdk/blob/master/test/build.js#L337

*Detect and Verify Signatures for OP_RETURN fields*:

https://github.com/BitcoinFiles/bitcoinfiles-sdk/blob/master/test/utils.js#L662
\
*Broadcast Signed File with Datapay*:

https://github.com/BitcoinFiles/bitcoinfiles-sdk/blob/master/test/build.js#L292

# Libraries

[go-aip]( (Golang)
[BitcoinFiles SDK](https://github.com/BitcoinFiles/bitcoinfiles-sdk#sign-and-create-file) (Javascript)

