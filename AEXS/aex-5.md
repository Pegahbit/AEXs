# AEX 5

```
AEX: 5
Title: Inter-Wallet Communication
Author: Shubhendu Shekhar <@shekhar-shubhendu>
License: BSD-3-Clause
Discussions-To: https://forum.aeternity.com/t/aex-5-inter-wallet-communication/3371
Status: Draft
Type: Standards Track
Created: 2019-04-29
```

## Simple Summary

This document describes the process and messages required for inter wallet communication between aeternity supported wallets.

## Motivation

The purpose of AEXs is to provide specification and JSON-RPC compatible messages required for an inter-wallet communication channel.

Currently, if a wallet wants to talk to another wallet then each wallet has to implement its own set of messages that the wallet on the other end will understand and this gets limited to communication between the wallets of a single vendor. Although, this single vendor implementation eases the transaction and message signing part when having multiple devices but it binds users to a single wallet and single experience.

By standardization of the set of messages that should be used for inter-wallet communication this AEX will to open the communication between different types of wallet(mobile, web, extensions) from different vendors.

### Current Process (with regards to AEX-2)

1. SDK connects with a wallet
2. Wallet shares the account address with the SDK
3. When the SDK needs the wallet to sign the transaction it asks the wallet
4. Wallet signs and returns the signed transaction.

### Proposed process with inter-wallet communication

1. SDK connects with a wallet
2. Wallet returns its current address and other account addresses it may help the SDK in getting signature for.
3. When the SDK needs the wallet to sign the transaction it asks the wallet
4. Wallet performs steps in the below order
    1. checks if it can sign the transaction.
    2. else it checks if it holds a direct link to the wallet that can sign the transaction and forwards the request to it.
    3. and at last it checks if there is an indirect link to the signing address i.e. via another connected wallet and forwards the request to it.
5. Once the transaction is signed, it is returned back to the requesting SDK for further processing.

## Specification

### Protocol Messages

#### General

- `error`: Used to communicate any error occurred. Error code `1` to `8` are reserved and reused here from [AEX-2](https://github.com/aeternity/AEXs/blob/master/AEXS/aex-2.md#types-of-errors).

- `ping/pong`: ping/pong messages for liveness check.

  JSON-RPC 2.0 structure:

  ```jsonc
  Ping:
  {
    "jsonrpc": "2.0",
    "method": "ping",
    "params": {
       "id": "<unique_identifier_uuidv4>"
    },
    "id": 1,
    "version": 1
  }

  Pong:
  {
    "jsonrpc": "2.0",
    "result": {
       "id": "<unique_identifier_uuidv4>",
       "data": "pong"
    },
    "id": 1,
    "version": 1
  }
  ```

#### Start and Close

- `wallet.channel.initiate`: Initiate request to open a communication channel. The request contains an identifier that will be used by either party to recognize and process incoming messages. The `id` should match the subsequent incoming or outgoing messages between the two wallets. The generated identifier must be unique and must conform to the [UUID v4 standards](https://tools.ietf.org/html/rfc4122#page-14).

  JSON-RPC 2.0 structure:

  ```jsonc
  Request:
  {
    "jsonrpc": "2.0",
    "method": "wallet.channel.initiate",
    "params": {
       "id": "<unique_identifier_uuidv4>"
    },
    "version": 1
  }

  Response:
  {
    "jsonrpc": "2.0",
    "result": {
       "id": "<unique_identifier_uuidv4_as_request>",
       "name": "<optional_name_param>"
    },
    "id": 1,
    "version": 1
  }
  ```

- `wallet.channel.close`: Close the channel

  JSON-RPC 2.0 structure:

  ```jsonc
  Request:
  {
    "jsonrpc": "2.0",
    "method": "wallet.channel.close",
    "params": {
       "id": "<unique_identifier_uuidv4>"
    },
    "version": 1
  }

  Response:
  {
    "jsonrpc": "2.0",
    "result": {
       "id": "<unique_identifier_uuidv4_as_request>",
       "status": "ack"
    },
    "id": 1,
    "version": 1
  }
  ```

- `wallet.channel.close_incoming`: Close channel only for signing requests i.e. closing wallet can still ask the wallet on another end to sign or forward the transactions.

  JSON-RPC 2.0 structure:

  ```jsonc
  Request:
  {
    "jsonrpc": "2.0",
    "method": "wallet.channel.close_incoming",
    "params": {
       "id": "<unique_identifier_uuidv4>"
    },
    "version": 1
  }

  Response:
  {
    "jsonrpc": "2.0",
    "result": {
       "id": "<unique_identifier_uuidv4_as_request>",
       "status": "ack"
    },
    "id": 1,
    "version": 1
  }
  ```

#### Address Update

- `wallet.get.address`: Ask the connected wallet for its address

  JSON-RPC 2.0 structure:

  ```jsonc
  {
    "jsonrpc": "2.0",
    "method": "wallet.get.address",
    "params": {
       "id": "<unique_identifier_uuidv4>"
    },
    "id":1,
    "version": 1
  }
  ```

- `wallet.update.address`: Issued by connected wallets for returning addresses incl. from the connected wallets. The `connected` field is optional.

  JSON-RPC 2.0 structure:

  ```jsonc
  {
    "jsonrpc": "2.0",
    "method": "wallet.update.address",
    "params": {
       "id": "<unique_identifier_uuidv4>",
       "address": {
         "current": {
           "<address>": {
             // object with related metadata(optional)
           }
         },
         "connected": {
           "<address1>": {
             // object with related metadata(optional)
           },
           "<address2>": {
             // object with related metadata(optional)
           }
         }
       }
    },
    "version": 1
  }
  ```

#### Sign and Broadcast Transaction

- `wallet.sign.tx`: Ask the connected wallet to sign the transaction.

  JSON-RPC 2.0 structure:

  ```jsonc
  Request:
  {
    "jsonrpc": "2.0",
    "method": "wallet.sign.tx",
    "params": {
       "id": "<unique_identifier_uuidv4>",
       "tx": "<raw_unsigned_tx>"
    },
    "id":1,
    "version": 1
  }

  Response:
  {
    "jsonrpc": "2.0",
    "result": {
       "id": "<unique_identifier_uuidv4_as_request>",
       "tx": "<signed_tx>"
    },
    "id":1,
    "version": 1
  }
  ```

- `wallet.request.broadcast`: Ask connected wallet to broadcast the transaction. The connected wallet can try to broadcast the transaction itself or forward it to the SDK. If the wallet is unable to broadcast it returns the `error`  with code `3`.

  JSON-RPC 2.0 structure:

  ```jsonc
  Request:
  {
    "jsonrpc": "2.0",
    "method": "wallet.request.broadcast",
    "params": {
       "id": "<unique_identifier_uuidv4>",
       "tx": "<signed_tx>",
       "verify": true/false
    },
    "version": 1
  }

  Response:
  {
    "jsonrpc": "2.0",
    "result": {
       "id": "<unique_identifier_uuidv4_as_request>",
       "tx_id": "<tx_id>"
    },
    "id": 1,
    "version": 1
  }
  ```

### Example Flow

[WIP]