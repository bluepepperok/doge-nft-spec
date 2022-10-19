# doge-nft-spec

Spec for Dogecoin blockchain based NFTs

## Overview

An NFT is related to two UTXOs at any given block:
1. Metadata UTXO. This is the one with the `OP_RETURN` script that describes where the metadata id can be found.
2. Owner UTXO. This UTXO is meant to have a witness script, in particular, a P2PKH script.

##### NFT ids

Note that the metadata UTXO cannot be spent. Since this UTXO contains a reference to details of the NFT, we define the NFT id to be the tuple `(tx id, nft index)`. The `tx id` corresponds to the tx id of the metadata UTXO. The `nft index` must be a valid index in the array of NFT descriptions found in the metadata content. These indexes start from `0`.

##### NFT owner UTXO magic value

NFT owner UTXOs have a magic value in their associated dogecoin amount: `0x0FF415`

This is just above the current soft dust limit.

##### NFT burns

There is no explicit burn tx format but any NFT owner UTXO expenditure that doesn't conform to one of these transaction formats results in irreversible destruction of the NFT ownership.


#### IPFS metadata

NFT metadata is meant to be hosted offchain. IPFS is the only storage solution that is supported in this version of the protocol.

##### Mint descriptor

The mint descriptor must be in JSON format and conform to the `Mint` type in the following snippet.

```ts
interface Mint {
  /**
   * NFT descriptors for each NFT minted.
   */
  readonly mint: readonly NftInformation[];
  /**
   * Collection CIDv0.
   * All NFTs minted are part of this collection.
   * Encoded in base64.
   */
  readonly collection: string;
  /**
   * Signature of the mint operation.
   * This is a signature of the message formed by the nft descriptors and the collection.
   */
  readonly signature: MessageSignature;
}

interface NftInformation {
  /**
   * Name of the NFT.
   */
  readonly name: string;
  /**
   * NFT description.
   */
  readonly description: string;
  /**
   * NFT tags.
   */
  readonly tags?: readonly string[];
  /**
   * NFT image IPFS CIDv0.
   * Encoded in base64.
   */
  readonly image: string;
}
```

See [here](#messsage-signature) for a description of the `MessageSignature` type.

A mint descriptor could look like this:

```json
{
    "mint": [
        {
            "name": "Doge",
            "description": "The picture of a Doge",
            "tags": ["doge"],
            "image": "aoofSf5I4uSt2qBYbCPBpdzJJhH15vednTgGzi5nRZE="
        }
    ],
    "collection": "0inJiCXqkQ+OU08IaSdYl8Kpqkyk2MeVtTF5fesjk8k=",
    "signature": {
        "r": "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=",
        "s": "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=",
        "v": 31
    }
}
```

##### Collection descriptor

The collection descriptor must be in JSON format and conform to the `CollectionCreate` type in the following snippet.

```ts
interface CollectionCreate {
  readonly create: NftCollection;
  /**
   * Signature of the collection create operation.
   * This is a signature of the message formed by the collection descriptor.
   */
  readonly signature: MessageSignature;
}

interface NftCollection {
  /**
   * Name of the collection.
   */
  readonly name: string;
  /**
   * Social media URLs for the collection.
   */
  readonly socialMedia?: {
    readonly twitter?: string;
    readonly discord?: string;
    readonly webpage?: string;
  };
  /**
   * Collection description.
   */
  readonly description: string;
  /**
   * Collection tags.
   */
  readonly tags?: readonly string[];
  /**
   * Collection image IPFS CIDv0.
   * Encoded in base64.
   */
  readonly image: string;
  /**
   * Collection banner image IPFS CIDv0.
   * Encoded in base64.
   */
  readonly bannerImage: string;
}
```

See [here](#messsage-signature) for a description of the `MessageSignature` type.

A collection descriptor could look like this:

```json
{
    "create": [
        {
            "name": "Doges",
            "description": "Doge-themed tokens",
            "tags": ["doge"],
            "socialMedia": {
              "webpage": "https://bluepepper.io"
            },
            "image": "KXWybiE4Hc0zopv0Sl8TCp5fCrmCklcAyWSo1i2GxSc=",
            "bannerImage": "A4aHn+OB0eFKHRLjrvNNGgoFidlN7okwl2feD4x/R5Y=",
        }
    ],
    "signature": {
        "r": "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=",
        "s": "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=",
        "v": 31
    }
}
```


##### Message signature

Message signatures in the protocol must conform to the `MessageSignature` type in the following snippet.

```ts
interface MessageSignature {
  /**
   * R parameter of the signature.
   * Encoded in base64.
   */
  readonly r: string;
  /**
   * S parameter of the signature.
   * Encoded in base64.
   */
  readonly s: string;
  /**
   * Version byte of the signature.
   * Encodes the recovery ID and the type of public key.
   */
  readonly v: number;
}
```

## Protocol

We'll assume:
- The wallet software involved in the creation of these transactions won't reorder outputs nor inputs.

Note that spending an NFT owner UTXO without following the protocol described here destroys the NFT irrevocably.

### Mint tx

#### Use case

Mint allows a user to create a new NFT and associate metadata to it.

#### Tx format

- Inputs
    1. Artist’s input.
    2. Input that pays tx fee (optional, can be more than one)
- Outputs
    1. OP_RETURN of the following 45 bytes:
        1. "DogecoinNFTv" (12 bytes)
        2. varint representing `1` (1 byte)
        3. IPFS CIDv0 of mint descriptor (32 bytes, this is a sha256 hash)
    2. NFT owner UTXOs
        - Locking script is P2PKH to NFT owner’s PKH
        - Amount should be the [NFT owner UTXO magic value](#nft-owner-utxo-magic-value).
        - There should be as many consecutive NFT owner UTXOs as NFT descriptors present in the mint descriptor.
    3. Change (optional, can be more than one)
        - The first change output must not have an amount equal to the [NFT owner UTXO magic value](#nft-owner-utxo-magic-value).

Note that there's a variable number of outputs in this transaction.

The following must be true for the mint tx to be considered valid:

- There must be enough NFT owner outputs for the NFTs listed in the mint descriptor. Each one of them must have the [NFT owner UTXO magic value](#nft-owner-utxo-magic-value) for its attached amount.
- There must be at least one NFT in the mint descriptor.
- The owner of all NFT owner UTXOs minted must be the signer of the mint descriptor.
In other words, the PKH in the locking script for each of the NFT owner UTXOs generated in this tx must be the PKH of the signer of the mint descriptor.
- The signer of the mint descriptor must be the signer of the collection referenced in the mint descriptor.
- The first change output must not have an amount equal to the [NFT owner UTXO magic value](#nft-owner-utxo-magic-value).

### Sell offer tx

This is actually a partial tx meant to be included in a full sell tx.

#### Use case

Sell offers can be used to publish sales of NFTs.

#### Tx format

- Input
    1. NFT owner UTXO unlock (seller input)

- Output
    1. Value expected in exchange for NFT.
        - Locking script should be P2PKH of a PKH given by the seller.

The tx should be signed with `SINGLE | ANYONECANPAY` SIGHASH flags. Note that the seller only spends the NFT owner UTXO and, thus, the tx fee must be paid for by other inputs.

### Sell tx

#### Use case

Sell txs can be used to complete sales of NFTs.

#### Tx format

- Inputs
    1. NFT owner UTXO unlock (seller input)
    2. Other inputs (buyer inputs). Tx fee, marketplace fee and doge value offered for NFT should come from these.

- Outputs
    1. Value offered in exchange for NFT.
        - Locking script should be P2PKH of a PKH given by the seller.
        - The amount should be whatever was agreed upon.
    2. NFT owner UTXO.
        - Locking script should be P2PKH of the receiving PKH
        - The amount associated with this output should be the [NFT owner magic value](#nft-owner-utxo-magic-value).
    3. Marketplace fee output (optional)
        - Its value should be a percentage of the value offered in exchange and it should be at least big enough to not be dust.
        - Locking script should be given by the marketplace API.
    4. Change (optional)


Note that the first input and output coincide with the format of a [sell offer tx](#sell-offer-tx).
When the sell tx is built using a sell offer, the first input will be signed with `SINGLE | ANYONECANPAY` SIGHASH flags.

### Transfer tx

#### Use case

Transfer can be used to send an NFT without receiving anything in exchange. It is useful for gifts, for simple transfers to another address owned by the same user, or when compensation is offered offchain.

#### Tx format

- Inputs
    1. NFT UTXO unlock
    2. An input that pays the tx fee.
    3. Other inputs may follow.
- Outputs
    1. NFT owner UTXO.
        - Locking script should be P2PKH of the receiving PKH.
        - The amount associated with this output should be the [NFT owner magic value](#nft-owner-utxo-magic-value).
    2. (Optional) Change output.
    3. (Optional) Other outputs.
