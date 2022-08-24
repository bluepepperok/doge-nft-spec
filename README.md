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

## Protocol

We'll assume:
- The wallet software involved in the creation of these transactions won't reorder outputs nor inputs.

Note that spending an owner NFT UTXO without following the protocol described here destroys the NFT irrevocably.

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
        3. IPFS CIDv0 of metadata (32 bytes, this is a sha256 hash)
    2. NFT owner UTXO
        - Locking script is P2PKH to NFT owner’s PKH
        - Amount should be the [NFT owner magic value](#nft-owner-utxo-magic-value).
    3. Change (optional, can be more than one)


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
        - Locking script should be P2PKH of the receiving PKH
        - The amount associated with this output should be the [NFT owner magic value](#nft-owner-utxo-magic-value).
    2. (Optional) Change output.
    3. (Optional) Other outputs.
