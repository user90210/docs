# TezTok Token Indexer

An indexer that aggregates and normalizes NFT related data on the Tezos Blockchain and provides a GraphQL API for developers. Currently supported marketplaces: hicetnunc, teia, objkt.com\*, versum\*\*, fxhash.

The indexer is in an experimental/unstable state and is not meant to be used in production yet. The current goal is to collect some feedback from developers in order to determine if something essential is missing and to find out who would be interested in using it. The idea is to run the indexer as a service and provide the source code on GitHub (but it still needs a bit more polishing first). We would love to hear if you are interested in using the indexer. [You can reach us on Discord](https://discord.gg/KBHxf4RbzC).


## Use cases

* Applications showcasing Tezos NFTs including price information
* Services pushing notifications on mints, swaps or price drops
* Tax software reporting on transactions in specific time frames
* General data analysis on Tezos NFTs

## How it works

The indexer aggregates and stores an event history for all supported tokens. 
Every time a new event occurs on a token, the indexer processes the event history and re-creates a token model along with related models like listings, offers, tags, and holdings. 
Besides that, the indexer fetches the meta data of each token and creates a set of thumbnails.


*\*except newest auctions contract*

*\*\*except auctions*

- - -

**GraphiQL Playground:** https://graphiql.teztok.com/

**GraphQL API:** https://unstable-do-not-use-in-production-api.teztok.com/v1/graphql

**Status:** Experimental/Unstable

**Source Code:** Yet to be released (still needs some polishing)

- - -

## Example GraphQL Queries

*Get a token with its current price, including active listings, active offers, and holders:*

```graphql
query GetToken {
  tokens(where: {fa2_address: {_eq: "KT1RJ6PbjHpwc3M5rw5s2Nbmefwbuwbdxton"}, token_id: {_eq: "135489"}}) {
    name
    description
    artist_address
    price
    highest_sales_price
    offers(where: {status: {_eq: "active"}}, order_by: {price: desc}) {
      type
      contract_address
      price
      buyer_address
      currency
    }
    listings(where: {status: {_eq: "active"}}, order_by: {price: asc}) {
      type
      contract_address
      amount_left
      price
      seller_address
      currency
    }
    holdings(where: {amount: {_gt: "0"}}) {
      holder_address
      amount
    }
  }
}
```

- - -

*Get latest sales across marketplaces:*

```graphql
query GetLatestSales {
  events(where: {implements: {_eq: "SALE"}}, order_by: {opid: desc}, limit: 100) {
    type
    timestamp
    seller_address
    buyer_address
    price
    token {
      fa2_address
      token_id
      name
      artist_address
    }
  }
}
```

- - -

*Get Tezzardz ranked by their highest sales price:*

```graphql
query GetBestSellingTezzardz {
  tokens(where: {fa2_address: {_eq: "KT1LHHLso8zQWQWg1HUukajdxxbkGfNoHjh6"}}, order_by: {highest_sales_price: desc_nulls_last}, limit: 10) {
    name
    description
    metadata_status
    highest_sales_price
    attributes
  }
}
```

- - -

*Get all tokens that had a sale in 2022 owned by a specified address:*

```graphql
query GetLiquidHoldings {
  holdings(where: {holder_address: {_eq: "tz1P9djZkQKQdvt68eixoyBf8ezNYjgrEdue"}, token: {last_sale_at: {_gte: "2022-01-01", _lt: "2023-01-01"}}}, order_by: {token: {last_sale_at: desc}}) {
    amount
    holder_address
    token {
      fa2_address
      token_id
      editions
      name
      description
      price
      last_sale_at
      last_sales_price
    }
  }
}
```
- - -

*Get the sales volume of hicetnunc in march 2022:*

```graphql
query VolumeInMarch {
  events_aggregate(where: {fa2_address: {_eq: "KT1RJ6PbjHpwc3M5rw5s2Nbmefwbuwbdxton"}, implements: {_eq: "SALE"}, timestamp: {_gte: "2022-03-01", _lt: "2022-04-01"}}) {
    aggregate {
      sum {
        price
      }
    }
  }
}
```

- - -

*Example execution of a GraphQL query in pure JavaScript:*

```javascript
(async () => {
  const rawResponse = await fetch('https://unstable-do-not-use-in-production-api.teztok.com/v1/graphql', {
    method: 'POST',
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      query: `
        query GetLatestSales {
          events(where: {implements: {_eq: "SALE"}}, order_by: {opid: desc}, limit: 100) {
          type
          timestamp
          seller_address
          buyer_address
          price
          token {
              fa2_address
              token_id
              name
              artist_address
            }
          }
        }
      `
    })
  });

  const result = await rawResponse.json();
  console.log(result);
})();
```

- - -

## Events

A list of token related events that are currently captured by the indexer.

**FA2_TRANSFER**

*A token got transferred from one address to another.*

```javascript
{
    type: 'FA2_TRANSFER',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    from_address: TezosAddress,
    to_address: TezosAddress,
    amount: PgBigInt, // the amount of editions that were transferred
}
```
- - -

**SET_LEDGER**

*The balance of a ledger entry got updated.*

```javascript
{
    type: 'SET_LEDGER',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    holder_address: TezosAddress,
    amount: PgBigInt, // the balance
    is_mint: boolean(),
    price: PgBigInt, // this can be used to determ the mint price
}
```
- - -

**SET_METADATA**

*The metadata uri of a token was changed.*

```javascript
{
    type: 'SET_METADATA',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    metadata_uri: MetadataUri, // e.g. ipfs://QmeaqRBUiw4cJiNKEcW2noc7egLd5GgBqLcHHqUhauJAHN
}
```
- - -

**HEN_MINT**

*A token was minted on hicetnunc.*

```javascript
{
    type: 'HEN_MINT',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    artist_address: TezosAddress,
    royalties: PgBigInt,
    editions: PgBigInt,
    metadata_uri: MetadataUri,
}
```
- - -

**HEN_SWAP**

*A token was swapped on hicetnunc (marketplace contract: KT1Hkg5qeNhfwpKW4fXvq7HGZB9z2EnmCCA9).*

```javascript
{
    type: 'HEN_SWAP',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    seller_address: TezosAddress,
    swap_id: PgBigInt,
    price: PgBigInt,
    amount: PgBigInt,
}
```
- - -

**HEN_COLLECT**

*A token was collected on hicetnunc (marketplace contract: KT1Hkg5qeNhfwpKW4fXvq7HGZB9z2EnmCCA9).*

```javascript
{
    type: 'HEN_COLLECT',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    seller_address: TezosAddress,
    buyer_address: TezosAddress,
    swap_id: PgBigInt,
    price: PgBigInt,
    amount: PgBigInt,
}
```
- - -

**HEN_CANCEL_SWAP**

*A swap was cancelled on hicetnunc (marketplace contract: KT1Hkg5qeNhfwpKW4fXvq7HGZB9z2EnmCCA9).*

```javascript
{
    type: 'HEN_CANCEL_SWAP',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    seller_address: TezosAddress,
    swap_id: PgBigInt,
}
```
- - -

**HEN_SWAP_V2**

*A token was swapped on hicetnunc (marketplace contract: KT1HbQepzV1nVGg8QVznG7z4RcHseD5kwqBn).*

```javascript
{
    type: 'HEN_SWAP_V2',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    artist_address: TezosAddress,
    seller_address: TezosAddress,
    swap_id: PgBigInt,
    price: PgBigInt,
    royalties: PgBigInt,
    amount: PgBigInt,
}
```
- - -

**HEN_COLLECT_V2**

*A token was collected on hicetnunc (marketplace contract: KT1HbQepzV1nVGg8QVznG7z4RcHseD5kwqBn).*

```javascript
{
    type: 'HEN_COLLECT_V2',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    seller_address: TezosAddress,
    buyer_address: TezosAddress,
    artist_address: TezosAddress,
    swap_id: PgBigInt,
    price: PgBigInt,
}
```
- - -

**HEN_CANCEL_SWAP_V2**

*A swap was cancelled on hicetnunc (marketplace contract: KT1HbQepzV1nVGg8QVznG7z4RcHseD5kwqBn).*

```javascript
{
    type: 'HEN_CANCEL_SWAP_V2',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    seller_address: TezosAddress,
    artist_address: TezosAddress,
    swap_id: PgBigInt,
}
```
- - -

**TEIA_SWAP**

*A token was swapped on teia (marketplace contract: KT1PHubm9HtyQEJ4BBpMTVomq6mhbfNZ9z5w).*

```javascript
{
    type: 'TEIA_SWAP',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    artist_address: TezosAddress,
    seller_address: TezosAddress,
    swap_id: PgBigInt,
    price: PgBigInt,
    royalties: PgBigInt,
    amount: PgBigInt,
}
```
- - -

**TEIA_COLLECT**

*A token was collected on teia (marketplace contract: KT1PHubm9HtyQEJ4BBpMTVomq6mhbfNZ9z5w).*

```javascript
{
    type: 'TEIA_COLLECT',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    buyer_address: TezosAddress,
    seller_address: TezosAddress,
    artist_address: TezosAddress,
    swap_id: PgBigInt,
    price: PgBigInt,
}
```
- - -

**TEIA_CANCEL_SWAP**

*A swap was cancelled on teia (marketplace contract: KT1PHubm9HtyQEJ4BBpMTVomq6mhbfNZ9z5w).*

```javascript
{
    type: 'TEIA_CANCEL_SWAP',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    seller_address: TezosAddress,
    artist_address: TezosAddress,
    swap_id: PgBigInt,
}
```
- - -

**OBJKT_MINT_ARTIST**

*A token was minted in an artist collection on objkt.com.*

```javascript
{
    type: 'OBJKT_MINT_ARTIST',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    artist_address: TezosAddress,
    collection_id: PgBigInt,
    editions: PgBigInt,
    metadata_uri: MetadataUri,
}
```
- - -

**OBJKT_ASK**

*An ask was created on objkt.com (marketplace contract: KT1FvqJwEDWb1Gwc55Jd1jjTHRVWbYKUUpyq).*

```javascript
{
    type: 'OBJKT_ASK',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    ask_id: PgBigInt,
    seller_address: TezosAddress,
    artist_address: TezosAddress,
    royalties: PgBigInt,
    price: PgBigInt,
    amount: PgBigInt,
}
```
- - -

**OBJKT_FULFILL_ASK**

*An ask was fulfilled on objkt.com (marketplace contract: KT1FvqJwEDWb1Gwc55Jd1jjTHRVWbYKUUpyq).*

```javascript
{
    type: 'OBJKT_FULFILL_ASK',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    ask_id: PgBigInt,
    seller_address: TezosAddress,
    buyer_address: TezosAddress,
    artist_address: TezosAddress,
    price: PgBigInt,
}
```
- - -

**OBJKT_RETRACT_ASK**

*An ask was cancelled on objkt.com (marketplace contract: KT1FvqJwEDWb1Gwc55Jd1jjTHRVWbYKUUpyq).*

```javascript
{
    type: 'OBJKT_RETRACT_ASK',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    artist_address: TezosAddress,
    seller_address: TezosAddress,
    ask_id: PgBigInt,
}
```
- - -

**OBJKT_BID**

*A bid was created on objkt.com (marketplace contract: KT1FvqJwEDWb1Gwc55Jd1jjTHRVWbYKUUpyq).*

```javascript
{
    type: 'OBJKT_BID',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    bid_id: PgBigInt,
    buyer_address: TezosAddress,
    artist_address: TezosAddress,
    royalties: PgBigInt,
    price: PgBigInt,
}
```
- - -

**OBJKT_FULFILL_BID**

*A bid was fulfilled on objkt.com (marketplace contract: KT1FvqJwEDWb1Gwc55Jd1jjTHRVWbYKUUpyq).*

```javascript
{
    type: 'OBJKT_FULFILL_BID',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    bid_id: PgBigInt,
    price: PgBigInt,
    seller_address: TezosAddress,
    buyer_address: TezosAddress,
    artist_address: TezosAddress,
}
```
- - -

**OBJKT_RETRACT_BID**

*A bid was cancelled on objkt.com (marketplace contract: KT1FvqJwEDWb1Gwc55Jd1jjTHRVWbYKUUpyq).*

```javascript
{
    type: 'OBJKT_RETRACT_BID',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    artist_address: TezosAddress,
    buyer_address: TezosAddress,
    bid_id: PgBigInt,
}
```
- - -

**OBJKT_ASK_V2**

*An ask was created on objkt.com (marketplace contract: KT1WvzYHCNBvDSdwafTHv7nJ1dWmZ8GCYuuC).*

```javascript
{
    type: 'OBJKT_ASK_V2',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    ask_id: PgBigInt,
    seller_address: TezosAddress,
    currency: string(),
    price: PgBigInt,
    amount: PgBigInt,
    end_time: optional(IsoDateString),
}
```
- - -

**OBJKT_FULFILL_ASK_V2**

*An ask was fulfilled on objkt.com (marketplace contract: KT1WvzYHCNBvDSdwafTHv7nJ1dWmZ8GCYuuC).*

```javascript
{
    type: 'OBJKT_FULFILL_ASK_V2',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    ask_id: PgBigInt,
    seller_address: TezosAddress,
    buyer_address: TezosAddress,
    price: PgBigInt,
    amount: PgBigInt,
}
```
- - -

**OBJKT_RETRACT_ASK_V2**

*An ask was cancelled on objkt.com (marketplace contract: KT1WvzYHCNBvDSdwafTHv7nJ1dWmZ8GCYuuC).*

```javascript
{
    type: 'OBJKT_RETRACT_ASK_V2',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    seller_address: TezosAddress,
    ask_id: PgBigInt,
}
```
- - -

**OBJKT_OFFER**

*An offer was created on objkt.com (marketplace contract: KT1WvzYHCNBvDSdwafTHv7nJ1dWmZ8GCYuuC).*

```javascript
{
    type: 'OBJKT_OFFER',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    offer_id: PgBigInt,
    buyer_address: TezosAddress,
    currency: string(),
    price: PgBigInt,
    end_time: optional(IsoDateString),
}
```
- - -

**OBJKT_FULFILL_OFFER**

*An offer was fulfilled on objkt.com (marketplace contract: KT1WvzYHCNBvDSdwafTHv7nJ1dWmZ8GCYuuC).*

```javascript
{
    type: 'OBJKT_FULFILL_OFFER',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    offer_id: PgBigInt,
    seller_address: TezosAddress,
    buyer_address: TezosAddress,
    artist_address: optional(TezosAddress),
    price: PgBigInt,
}
```
- - -

**OBJKT_RETRACT_OFFER**

*An offer was cancelled on objkt.com (marketplace contract: KT1WvzYHCNBvDSdwafTHv7nJ1dWmZ8GCYuuC).*

```javascript
{
    type: 'OBJKT_RETRACT_OFFER',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    buyer_address: TezosAddress,
    offer_id: PgBigInt,
}
```
- - -

**OBJKT_CREATE_ENGLISH_AUCTION**

*An english auction was created on objkt.com (contracts: KT1Wvk8fon9SgNEPQKewoSL2ziGGuCQebqZc and KT1XjcRq5MLAzMKQ3UHsrue2SeU2NbxUrzmU).*

```javascript
{
    type: 'OBJKT_CREATE_ENGLISH_AUCTION',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    seller_address: TezosAddress,
    artist_address: TezosAddress,
    reserve: PgBigInt,
    start_time: IsoDateString,
    end_time: IsoDateString,
    extension_time: PgBigInt,
    royalties: PgBigInt,
    price_increment: PgBigInt,
    auction_id: PgBigInt,
}
```
- - -

**OBJKT_BID_ENGLISH_AUCTION**

*Someone made a bid on an english auction that was created on objkt.com (contracts: KT1Wvk8fon9SgNEPQKewoSL2ziGGuCQebqZc and KT1XjcRq5MLAzMKQ3UHsrue2SeU2NbxUrzmU).*

```javascript
{
    type: 'OBJKT_BID_ENGLISH_AUCTION',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    seller_address: TezosAddress,
    artist_address: TezosAddress,
    bidder_address: TezosAddress,
    reserve: PgBigInt,
    start_time: IsoDateString,
    end_time: IsoDateString,
    current_price: PgBigInt,
    extension_time: PgBigInt,
    highest_bidder_address: TezosAddress,
    royalties: PgBigInt,
    price_increment: PgBigInt,
    auction_id: PgBigInt,
    bid: PgBigInt,
}
```
- - -

**OBJKT_CONCLUDE_ENGLISH_AUCTION**

*An english auction was concluded on objkt.com (contracts: KT1Wvk8fon9SgNEPQKewoSL2ziGGuCQebqZc and KT1XjcRq5MLAzMKQ3UHsrue2SeU2NbxUrzmU).*

```javascript
{
    type: 'OBJKT_CONCLUDE_ENGLISH_AUCTION',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    seller_address: TezosAddress,
    buyer_address: TezosAddress,
    artist_address: TezosAddress,
    reserve: PgBigInt,
    start_time: IsoDateString,
    end_time: IsoDateString,
    price: PgBigInt,
    extension_time: PgBigInt,
    royalties: PgBigInt,
    price_increment: PgBigInt,
    auction_id: PgBigInt,
}
```

- - -

**OBJKT_CREATE_DUTCH_AUCTION**

*A dutch auction was created on objkt.com (contracts: KT1ET45vnyEFMLS9wX1dYHEs9aCN3twDEiQw and KT1QJ71jypKGgyTNtXjkCAYJZNhCKWiHuT2r).*

```javascript
{
    type: 'OBJKT_CREATE_DUTCH_AUCTION',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    seller_address: TezosAddress,
    artist_address: TezosAddress,
    start_time: IsoDateString,
    end_time: IsoDateString,
    start_price: PgBigInt,
    end_price: PgBigInt,
    royalties: PgBigInt,
    auction_id: PgBigInt,
}
```


- - -

**OBJKT_BUY_DUTCH_AUCTION**

*Someone bought a token that was auctioned through a dutch auction on objkt.com (contracts: KT1ET45vnyEFMLS9wX1dYHEs9aCN3twDEiQw and KT1QJ71jypKGgyTNtXjkCAYJZNhCKWiHuT2r).*

```javascript
{
    type: 'OBJKT_BUY_DUTCH_AUCTION',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    seller_address: TezosAddress,
    buyer_address: TezosAddress,
    artist_address: TezosAddress,
    start_time: IsoDateString,
    end_time: IsoDateString,
    start_price: PgBigInt,
    end_price: PgBigInt,
    price: PgBigInt,
    royalties: PgBigInt,
    auction_id: PgBigInt,
}
```
- - -


**OBJKT_CANCEL_DUTCH_AUCTION**

*A dutch auction was cancelled on objkt.com (contracts: KT1ET45vnyEFMLS9wX1dYHEs9aCN3twDEiQw and KT1QJ71jypKGgyTNtXjkCAYJZNhCKWiHuT2r).*

```javascript
{
    type: 'OBJKT_CANCEL_DUTCH_AUCTION',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    seller_address: TezosAddress,
    artist_address: TezosAddress,
    start_time: IsoDateString,
    end_time: IsoDateString,
    start_price: PgBigInt,
    end_price: PgBigInt,
    royalties: PgBigInt,
    auction_id: PgBigInt,
}
```

- - -

**FX_MINT**

*A token was minted on fxhash (mint contract: KT1AEVuykWeuuFX7QkEAMNtffzwhe1Z98hJS).*

```javascript
{
    type: 'FX_MINT',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    artist_address: TezosAddress,
    buyer_address: TezosAddress,
    issuer_id: PgBigInt,
    royalties: PgBigInt,
    editions: PgBigInt,
    iteration: PgBigInt,
    metadata_uri: MetadataUri,
    price: PgBigInt,
}
```

- - -

**FX_MINT_V2**

*A token was minted on fxhash (mint contract: KT1XCoGnfupWk7Sp8536EfrxcP73LmT68Nyr).*

```javascript
{
    type: 'FX_MINT_V2',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    artist_address: TezosAddress,
    buyer_address: TezosAddress,
    issuer_id: PgBigInt,
    royalties: PgBigInt,
    editions: PgBigInt,
    iteration: PgBigInt,
    metadata_uri: MetadataUri,
    price: PgBigInt,
}
```

- - -

**FX_OFFER**

*An offer was created on fxhash (marketplace contract: KT1Xo5B7PNBAeynZPmca4bRh6LQow4og1Zb9). On fxhash, an offer is like a swap or an ask.*

```javascript
{
    type: 'FX_OFFER',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    seller_address: TezosAddress,
    artist_address: TezosAddress,
    offer_id: PgBigInt,
    royalties: PgBigInt,
    price: PgBigInt,
}
```

- - -

**FX_COLLECT**

*A token was collected on fxhash (marketplace contract: KT1Xo5B7PNBAeynZPmca4bRh6LQow4og1Zb9).*

```javascript
{
    type: 'FX_COLLECT',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    offer_id: PgBigInt,
    price: PgBigInt,
    buyer_address: TezosAddress,
    seller_address: TezosAddress,
    artist_address: TezosAddress,
}
```

- - -

**FX_CANCEL_OFFER**

*An offer was cancelled on fxhash (marketplace contract: KT1Xo5B7PNBAeynZPmca4bRh6LQow4og1Zb9).*

```javascript
{
    type: 'FX_CANCEL_OFFER',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    seller_address: TezosAddress,
    artist_address: TezosAddress,
    offer_id: PgBigInt,
}
```
- - -

**VERSUM_MINT**

*A token was minted on versum.*

```javascript
{
    type: 'VERSUM_MINT',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    artist_address: TezosAddress,
    royalties: PgBigInt, // TODO: add splits
    editions: PgBigInt,
    metadata_uri: MetadataUri,
}
```
- - -

**VERSUM_SWAP**

*A token was swapped on versum.*

```javascript
{
    type: 'VERSUM_SWAP',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    seller_address: TezosAddress,
    swap_id: PgBigInt,
    start_price: PgBigInt,
    end_price: PgBigInt,
    end_time: optional(IsoDateString),
    amount: PgBigInt,
    burn_on_end: boolean(),
}
```

- - -

**VERSUM_COLLECT_SWAP**

*A token was collected on versum.*

```javascript
{
    type: 'VERSUM_COLLECT_SWAP',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    buyer_address: TezosAddress,
    seller_address: TezosAddress,
    swap_id: PgBigInt,
    price: PgBigInt,
    amount: PgBigInt,
}
```
- - -

**VERSUM_CANCEL_SWAP**

*A swap was cancelled on versum.*

```javascript
{
    type: 'VERSUM_CANCEL_SWAP',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    seller_address: TezosAddress,
    artist_address: optional(TezosAddress),
    swap_id: PgBigInt,
}
```

- - -

**VERSUM_MAKE_OFFER**

*An offer was created on versum.*

```javascript
{
    type: 'VERSUM_MAKE_OFFER',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    offer_id: PgBigInt,
    buyer_address: TezosAddress,
    price: PgBigInt,
    amount: PgBigInt,
}
```

- - -

**VERSUM_ACCEPT_OFFER**

*An offer was accepted on versum.*

```javascript
{
    type: 'VERSUM_ACCEPT_OFFER',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    offer_id: PgBigInt,
    seller_address: TezosAddress,
    buyer_address: TezosAddress,
    price: PgBigInt,
    amount: PgBigInt,
}
```

- - -

**VERSUM_CANCEL_OFFER**

*An offer was cancelled on versum.*

```javascript
{
    type: 'VERSUM_CANCEL_OFFER',
    id: string(),
    opid: PositiveInteger,
    timestamp: IsoDateString,
    level: PositiveInteger,
    fa2_address: ContractAddress,
    token_id: string(),

    buyer_address: TezosAddress,
    offer_id: PgBigInt,
    amount: PgBigInt,
    price: PgBigInt,
}
```
