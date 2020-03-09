# Public API for Orion

## General API Information
* The base endpoint is: **https://demo.orionprotocol.io/backend**

## Public API
### Order book
```
GET /api/v1/orderBook
```
Order Book is a list of orders (price levels) for a given asset pair.

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES | Specific trading pair (symbol) to return. E.g. `ETH-BTC`
depth | INT | YES | Max number of price levels to return.

**Response:**
```json
{
  "asks": [
    {
      "price": 0.025866,
      "size": 0.218,
      "exchanges": [
        "binance"
      ]
    }
  ],
  "bids": [
    {
      "price": 0.0258598,
      "size": 0.16238718,
      "exchanges": [
        "poloniex"
      ]
    }
  ]
}
```

### Orion Cost Benefits
```
GET /api/v1/order-benefits
```
Calculate how executing a specified order on Orion is more profitable than on any single exchange 

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES | Specific trading pair (symbol) to calculate benefits. E.g. `ETH-BTC`.
ordQty | LONG | YES | Order quantity.
side | STRING | YES | Order side: `buy` or `sell`

**Response:**
```json
{
  "poloniex": {
    "benefitBtc": "0.00301081", // Benefits in quote asset
    "benefitPct": "0.11609" // Benefits in percent
  },
  "bittrex": {
    "benefitBtc": "0.00690512",
    "benefitPct": "0.26624"
  },
  "binance": {
    "benefitBtc": "0.00000108",
    "benefitPct": "0.00004"
  }
}
```

## Authenticated API (Signed)
### Deposit
```
CALL Exchange.depositAsset and CALL Exchange.depositWan
```
Deposit ERC-20 tokens or ETH to the Orion Exchange Contract to be available for trading.
For ERC-20 tokens a user has to first call `approve` method to allow Orion Contract to spend a specified value of tokens on his behalf. 

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
assetAddress | ADRESS | YES | Asset address of the currency ERC-20 token to deposit
amount | LONG | YES |  Asset amount to deposit in its base unit multiplied by `10^8` for ERC-20, or wei amount for ETH. Previously, the same amount or more has to approve to Orion Contract.
address | ADDRESS | YES | Caller address corresponding to the private key that sign the transaction 

**Example sing web3.js:**
```javascript
{
    const approveRes = await IERC20.methods
                    .approve(ORION_ADDRESS, amount)
                    .send({ from: address });
    const res = await exchange.methods
    				.deposit(assetAddress, amount)
    				.send({ from: address });
}
```
OR for ETH:
```javascript
const res = await exchange.methods
                .depositWan()
                .send({ from: address, value: amountInWei });
```

### Withdraw
```
CALL Exchange.withdraw
```
Withdrawal of remaining assets from the Orion Exchange Contract.
Specified funds will be transferred back to the address that sign calling method transaction. 

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
assetAddress | ADRESS | YES | Asset address of the currency ERC-20 token to withdraw
amount | LONG | YES |  Asset amount to withdraw in its base unit multiplied by `10^8` (both for ETH and ERC-20)
address | ADDRESS | YES | Caller address corresponding to the private key that sign the transaction

**Example sing web3.js:**
```javascript
{
    const res = await exchange.methods
    				.withdraw(assetAddress, amount)
    				.send({ from: address });
}
```

### Create order
```
POST /eth/api/order
```
Create a new order.

**Parameters:**
				
Name | Type | Description
------------ | ------------ | ------------
senderAddress | ADDRESS | Ethereum address of order's creator
matcherAddress | ADDRESS | Address of Orion exchange smart contract to which the creator to fulfill a trade
baseAsset | ADDRESS | Asset address that is the amount of a symbol
quoteAsset | ADDRESS | Asset address that is is the price of a symbol
side | String | `buy` or `sell`
amount | LONG | Amount of `baseAsset` multiplied by `10^8`
price | LONG | Limit price for order multiplied by `10^8`
matcherFee | LONG | Max fee amount that Orion smart contract can take for filling the whole order
matcherFeeAsset | ADDRESS | Asset address in which to charge fee
expiration | LONG | Unix time in milliseconds when the order will be expired. Should be more than one minute and less than one month from the current time.
nonce | LONG | Nonce for ensuring uniqueness of the order. Can be Unix timestamp of the order creation.
signature | HEX | The signature, see below.

*Signing an order:*

To create a signature for a specific order a user must:
1. Create SHA3 hash of all order fields
2. Sign hash data with private key for a specific account using Ethereum ECDSA function.

**Example using web3.js:**
```javascript
function longToBytes(long)  {
    return web3.utils.bytesToHex(Long.fromNumber(long).toBytesBE());
}

const message = web3.utils.soliditySha3(
		'0x03',
		order.senderAddress,
		order.matcherAddress,
		order.baseAsset,
		order.quoteAsset,
		order.matcherFeeAsset,
		longToBytes(order.amount),
		longToBytes(order.price),
		longToBytes(order.matcherFee),
		longToBytes(order.nonce),
		longToBytes(order.expiration),
		order.side === 'buy' ? '0x00' : '0x01'
	);

web3.eth.personal.sign(message, order.senderAddress).then(console.log);
```

**Request:**
```json
{
  "senderAddress":"0xc19d917a88e07e5040cd2443fb3a026838c3b852",
  "matcherAddress":"0xfbCAd2c3A90FBD94C335FBdF8E22573456Da7F68",
  "baseAsset":"0x0000000000000000000000000000000000000000", // ETH
  "quoteAsset":"0x335123EB7029030805864805fC95f1AB16A64D61", // BTC
  "side":"sell",
  "amount":50000000,
  "price":2591293,
  "matcherFee":300000,
  "matcherFeeAsset":"0x0000000000000000000000000000000000000000",
  "nonce":1583762939756,
  "expiration":1586268539756,
  "signature":"0x00c82bcbbe59d0ac418362b4ceeb915cc69acff681f61b4b4c4d2c640557ddd1502c4de373960dc6d5982b1bf59324b588657a1dbbe6c1def85f597dcafad48d1c"
}
```
**Response:**
```json
{
  "clientId":"0xc19d917a88e07e5040cd2443fb3a026838c3b852",
  "symbol":"ETH-BTC",
  "side":"sell",
  "orderQty":"0.5",
  "price":"0.02591293",
  "ordType":"LIMIT"
}
```

### Orders History
```
GET /api/v1/orderHistory
```
Get all orders for a specified addredd, by a symbol, time period.

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
address | ADDRESS | YES | Address of the orders owner to return. 
symbol | STRING | NO | Specific trading pair (symbol) to limit orders result. E.g. `ETH-BTC`
startTime | Long | NO | Starting date filter for orders in Unix timestamp.
endTime | STRING | NO | Ending date filter for orders in Unix timestamp.
limit | LONG | NO | Max number of orders to return. Default 500.

* If the symbol is not sent, orders for all symbols will be returned in an array.

**Response:**
```json
[
    {
        "id":156,
        "clientId":"0xc19d917a88e07e5040cd2443fb3a026838c3b852",
        "symbol":"ETH-BTC",
        "clientOrderId":null,
        "side":"buy",
        "orderQty":0.03,
        "price":0.02555132,
        "time":1583326030304,
        "updateTime":1583326030478,
        "ordType":"LIMIT",
        "filledQty":0.03,
        "status":"FILLED",
        "subOrders": [
            {
                "id":157,
                "orderId":156,
                "exchange":"BITTREX",
                "price":0.02554528,
                "subOrdQty":0.03,
                "fee":null,
                "reserved":true,
                "brokerId":"5e58170446e0fb00012238b7",
                "status":"FILLED",
                "spentQty":0.0
            }
        ]
    }
]
```
