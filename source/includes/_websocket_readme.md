# WebSocket API Specification

DEMO/STAGE site

`wss://stgapi.coinflex.com/v1`

LIVE site

`wss://api.coinflex.com/v1`

CoinFLEX's application programming interface (API) provides our clients programmatic access to control aspects of their accounts and to place orders on the CoinFLEX trading platform. The API is accessible via [WebSocket][IETF RFC 6455] connection to the URIs listed above. Commands, replies, and notifications traverse the WebSocket in text frames with [JSON][IETF RFC 4627]-formatted payloads.

WebSocket connections to the API will time out after 60 seconds if no traffic passes in either direction. To prevent timeouts, send a [Ping frame](https://tools.ietf.org/html/rfc6455#section-5.5.2) approximately every 45 seconds while the connection is otherwise idle. You do not need to send Ping frames if you are otherwise sending or receiving data frames on the socket.

To protect the performance of the system, CoinFLEX imposes certain limits on the rates at which you may issue commands to the API. Please see [Request Limits](#request-limits).

All quantities and prices are transmitted and received as integers with implicit scale factors. For scale information, please see [Scaling](#scaling).

CoinFLEX has published [client libraries](https://github.com/coinflex-exchange) for several popular languages to aid you in implementing your client application.

<aside class="notice">
Throughout this documentation, "iff" means "if and only if" - the biconditional logical connective.
</aside>
[Wikipedia of "if and only if"](https://en.wikipedia.org/wiki/If_and_only_if)

[IETF RFC 4627]: https://tools.ietf.org/html/rfc4627
[IETF RFC 6455]: https://tools.ietf.org/html/rfc6455
[scaled]: #scaling

## Message Flow

The WebSocket connection can be seen as logically carrying two independent communications channels: a control channel and an asynchronous notifications channel. Messages within each channel are ordered, but the two channels are not strictly synchronized with respect to each other. The distinction between the channels is implicit in the fields contained in the messages they carry; there are no explicit channel markers.

The control channel carries a series of command/reply pairs, where each command specifies a `method` and each reply specifies an `error_code`. Commands always flow from client to server, and replies always flow from server to client. A client may pipeline commands - that is, the client need not wait to receive the reply for one command before transmitting another command. However, because the server may reorder replies to the client with respect to the order in which the client submitted the commands that elicited those replies, a pipelining client should specify a unique `tag` in each command, to which it should correlate the matching `tag` in the corresponding reply. The server executes all commands that alter the state of the system in the order in which it receives them, but it may reply to information requests ahead of commands that alter state. For example, pipelining an `EstimateMarketOrder` command after a `PlaceOrder` command may mean that the returned estimate does not reflect any order book changes that result from the placed order.

The asynchronous notifications channel carries a series of notices, where each notice specifies a `notice` field. These notices may be delivered to the client at any time, and the timing of their delivery may not tightly correlate with command/reply messages in the control channel. For example, the `OrderOpened` notice that results from a `PlaceOrder` command may be delivered either before or after the command reply. However, notices are guaranteed to be delivered to the client in the same order as their respective events occurred in the trade engine. For example, an `OrdersMatched` notice that references a particular order will never be delivered _after_ an `OrderClosed` notice that references the same order.


## Contents

Currently, the WebSocket API supports the following:

* Commands:
	* [Authenticate](#authenticate)
	* [GetBalances](#getbalances)
	* [GetOrders](#getorders)
	* [EstimateMarketOrder](#estimatemarketorder)
	* [PlaceOrder](#placeorder)
	* [ModifyOrder](#modifyorder)
	* [CancelOrder](#cancelorder)
	* [CancelAllOrders](#cancelallorders)
	* [GetTradeVolume](#gettradevolume)
	* [WatchOrders](#watchorders)
	* [WatchTicker](#watchticker)

* Notifications:
	* [BalanceChanged](#balancechanged)
	* [OrderOpened](#orderopened)
	* [OrderModified](#ordermodified)
	* [OrdersMatched](#ordersmatched)
	* [OrderClosed](#orderclosed)
	* [TickerChanged](#tickerchanged)
	* [TradeVolumeChanged](#tradevolumechanged)


## Authenticate

> **Request**

```json
{
    "tag": <integer>,
    "method": "Authenticate",
    "user_id": <integer>,
    "cookie": <string>,
    "nonce": <string>,
    "signature": [
        <string>,
        <string>
    ]
}
```

> * `tag` is optional. Iff given and non-zero, it will be echoed in the reply.
> * `user_id` is the numeric identifier of the user who is attempting to authenticate.
> * `cookie` is the base64-encoded login cookie which is unique to the specified user.
> * `nonce` is a base64-encoded string of 16 bytes that have been randomly selected by the client.
> * `signature` is an array containing two base64-encoded strings, which encode the *r* and *s* components of the signature. Please see [CoinFLEX Authentication Process](#coinflex-authentication-process) for details of the authentication scheme.

> **Success Reply**

```json
{
    "tag": <integer>,
    "error_code": 0
}
```

> * `tag` is present iff `tag` was given and non-zero in the request.

> **Error Reply**

```json
{
    "tag": <integer>,
    "error_code": <integer>,
    "error_msg": <string>
}
```

> * `tag` is present iff `tag` was given and non-zero in the request.

Authenticates as a user by signing a challenge.

**Authorization:** None required.


error_code | error_msg
-------------|------------------------------------------------------------------
1            | "There is no such user."
6            | "You are making authentication attempts too rapidly."
7            | "You sent an incorrect login cookie."
7            | "You sent an incorrect signature. This probably means you used a wrong passphrase."
8            | *(varies)*


## GetBalances



> **Request**

```json
{
    "tag": <integer>,
    "method": "GetBalances"
}
```

> * `tag` is optional. Iff given and non-zero, it will be echoed in the reply.

> **Success Reply**

```json
{
    "tag": <integer>,
    "error_code": 0,
    "balances": [
        {
            "asset": <integer>,
            "balance": <integer>,
            "reserved_balance": <integer>,
            "total_balance": <integer>
        },
        …
    ]
}
```

> * `tag` is present iff `tag` was given and non-zero in the request.
> * `asset` is an asset code for which a balance is given.
> * `balance` is the user's [scaled][] available balance in the specified asset.
> * `reserved_balance` is the user's [scaled][] reserved balance in the specified asset.
> * `total_balance` is the user's [scaled][] total balance in the specified asset.

> **Error Reply**

```json
{
    "tag": <integer>,
    "error_code": <integer>,
    "error_msg": <string>
}
```

> * `tag` is present iff `tag` was given and non-zero in the request.

Retrieves the available balances of the authenticated user.

**Authorization:** Any authenticated user may invoke this command.

error_code | error_msg
-------------|------------------------------------------------------------------
6            | "You are making information requests too rapidly."
7            | "You are not authenticated."
8            | *(varies)*


## GetOrders

> **Request**

```json
{
    "tag": <integer>,
    "method": "GetOrders"
}
```

> * `tag` is optional. Iff given and non-zero, it will be echoed in the reply.

> **Success Reply**

```json
{
    "tag": <integer>,
    "error_code": 0,
    "orders": [
        {
            "id": <integer>,
            "tonce": <integer>,
            "base": <integer>,
            "counter": <integer>,
            "quantity": <integer>,
            "price": <integer>,
            "time": <integer>
        },
        …
    ]
}
```

> * `tag` is present iff `tag` was given and non-zero in the request.
> * `id` is the numeric identifier of an order.
> * `tonce` is the tonce given in the `PlaceOrder` command that opened the order, or **null** if that command did not specify a tonce.
> * `base` and `counter` are the asset codes of the base and counter assets of the order.
> * `quantity` is the [scaled][] amount of the base asset that is to be traded. It is negative for a sell order and positive for a buy order.
> * `price` is the [scaled][] price at which the order offers to trade.
> * `time` is the micro-timestamp at which the order was opened.

> **Error Reply**

```json
{
    "tag": <integer>,
    "error_code": <integer>,
    "error_msg": <string>
}
```

> * `tag` is present iff `tag` was given and non-zero in the request.

Retrieves the open orders of the authenticated user.

**Authorization:** Any authenticated user may invoke this command.

error_code | error_msg
-------------|------------------------------------------------------------------
6            | "You are making information requests too rapidly."
7            | "You are not authenticated."
8            | *(varies)*


## EstimateMarketOrder

> **Request**

```json
{
    "tag": <integer>,
    "method": "EstimateMarketOrder",
    "base": <integer>,
    "counter": <integer>,
    "quantity": <integer>,
    "total": <integer>
}
```

> * `tag` is optional. Iff given and non-zero, it will be echoed in the reply.
> * `base` and `counter` are the asset codes of the base and counter assets of the order.
> * `quantity` is the [scaled][] amount of base asset that is to be traded. It is negative for a sell order and positive for a buy order. This field must be omitted if `total` is supplied.
> * `total` is the [scaled][] amount of the counter asset that is to be traded. It is negative for a sell order and positive for a buy order. This field must be supplied if `quantity` is omitted.

> **Success Reply**

```json
{
    "tag": <integer>,
    "error_code": 0,
    "quantity": <integer>,
    "total": <integer>
}
```

> * `tag` is present iff `tag` was given and non-zero in the request.
> * `quantity` is the [scaled][] amount of the base asset that would have been traded if the market order really had been executed. It is always positive.
> * `total` is the [scaled][] amount of the counter asset that would have been traded if the market order really had been executed. It is always positive.

> **Error Reply**

```json
{
    "tag": <integer>,
    "error_code": <integer>,
    "error_msg": <string>
}
```

> * `tag` is present iff `tag` was given and non-zero in the request.

Simulates the execution of a market order and returns the quantity and total that would have been traded. This estimation does not take into account trading fees.

**Authorization:** None required.

error_code | error_msg
-------------|------------------------------------------------------------------
1            | "You specified an invalid asset pair."
6            | "You are sending orders too rapidly."
8            | "Quantity must not be zero."
8            | "Total must not be zero."
8            | "You must specify either quantity or total for a market order."
8            | *(varies)*


## PlaceOrder

> **Request**

```json
{
    "tag": <integer>,
    "method": "PlaceOrder",
    "tonce": <integer>,
    "base": <integer>,
    "counter": <integer>,
    "quantity": <integer>,
    "price": <integer>,
    "total": <integer>,
    "persist": <boolean>|"fill_or_kill",
    "post_only": <boolean>
}
```

> * `tag` is optional. Iff given and non-zero, it will be echoed in the reply.
> * `tonce` is optional. If given, it must be non-zero and unique to any tonce currently active with any live open orders.
The purpose of the tonce is to allow the user to resubmit a `PlaceOrder` command (e.g., after a connection failure) without risk of creating a duplicate order or to request cancellation of a just-placed order without needing to wait for the `PlaceOrder` reply containing the server-assigned order identifier.
> * `base` and `counter` are the asset codes of the base and counter assets of the order.
> * `quantity` is the [scaled][] amount of base asset that is to be traded. It is negative for a sell order and positive for a buy order. This field must be supplied if `price` is supplied and must be omitted if `total` is supplied.
> * `price` is the [scaled][] price at which the order offers to trade. It is optional; if omitted, the order will be executed as a market order.
> * `total` is the [scaled][] amount of the counter asset that is to be traded. It is negative for a sell order and positive for a buy order. This field must be omitted if `price` is supplied and must be supplied if `quantity` is omitted.
> * `persist` is optional. If **true** or omitted, the order will remain on the order book until canceled or fulfilled. If **false**, the order will be canceled automatically when the client disconnects. If `"fill_or_kill"`, the order will be canceled automatically after any immediate matches. This flag has no effect on market orders.
> * `post_only` is optional. If **true** the limit order will never take liquidity from the order book and can only become a non-aggressing maker order. If the limit price would result in an immediate match then the order will not be opened and hence rejected.

> **Success Reply**

```json
{
    "tag": <integer>,
    "error_code": 0,
    "id": <integer>,
    "time": <integer>,
    "remaining": <integer>
}
```

> * `tag` is present iff `tag` was given and non-zero in the request.
> * `id` is the numeric identifier assigned to the opened order. It is present only if `price` was supplied in the request.
> * `time` is the micro-timestamp at which the order was opened. It is present only if `price` was supplied in the request.
> * `remaining` is the [scaled][] amount of the base asset (if `quantity` was supplied in the request), or the [scaled][] amount of the counter asset (if `total` was supplied in the request), that could not be traded, either because insufficient liquidity existed on the order book or because the user's balance was exhausted. It is present only if `price` was omitted from the request.

> **Error Reply**

```json
{
    "tag": <integer>,
    "error_code": <integer>,
    "error_msg": <string>
}
```

> `tag` is present iff `tag` was given and non-zero in the request.

Places an order to execute a specified trade. This command is used to place both limit orders and market orders.

**Authorization:** Any authenticated user may invoke this command.

`quantity` | `price`  | `total`
-----------|----------|---------
supplied   | supplied | omitted
supplied   | omitted  | omitted
omitted    | omitted  | supplied


error_code | error_msg
-------------|------------------------------------------------------------------
1            | "You specified an invalid asset pair."
3            | "Tonce is out of sequence."
4            | "You have insufficient funds."
5            | "You have too many outstanding orders."
6            | "You are sending orders too rapidly."
7            | "You are not authenticated."
8            | "Tonce must not be zero."
8            | "Price must not be zero."
8            | "Quantity must not be zero."
8            | "Order total would overflow."
8            | "You must specify either quantity or total for a market order."
8            | *(varies)*
9            | "Post-only order with these parameters would result in an immediate match."


## ModifyOrder

> **Request**

```json
{
    "tag": <integer>,
    "method": "ModifyOrder",
    "id": <integer>,
    "tonce": <integer>,
    "quantity_delta": <integer>,
    "price": <integer>,
    "post_only": <boolean>
}
```

> * `tag` is optional. Iff given and non-zero, it will be echoed in the reply.
> * `id` is the numeric identifier of the order to be modified. This field must be omitted if `tonce` is supplied.
> * `tonce` is the tonce given in the `PlaceOrder` command that opened the order to be modified. This field must be supplied if `id` is omitted.
> * `quantity_delta` is the [scaled][] amount by which to adjust the quantity of the open order. It is negative to increase the quantity of a sell order or to decrease the quantity of a buy order. It is positive to increase the quantity of a buy order or to decrease the quantity of a sell order. It is optional; if omitted, the order's quantity will not be adjusted. If the requested adjustment would cause the order's quantity to reach or cross zero, then this command has the effect of canceling the order. If the order quantity is enlarged, then the order is moved to the end of the queue at its price level in the order book; otherwise, the order remains at its present position in the queue.
> * `price` is the new [scaled][] price at which the order offers to trade. It is optional; if omitted, the order's price will not be modified.
> * `post_only` is optional. If **true** the modified limit order will never take liquidity from the order book and can only become a non-aggressing maker order. If the limit price would result in an immediate match then the order will not be modified and the order remains unchanged.

> **Success Reply**

```json
{
    "tag": <integer>,
    "error_code": 0,
    "id": <integer>,
    "tonce": <integer>,
    "base": <integer>,
    "counter": <integer>,
    "quantity": <integer>,
    "price": <integer>,
    "time": <integer>
}
```

> * `tag` is present iff `tag` was given and non-zero in the request.
> * `id` is the numeric identifier of the modified order.
> * `tonce` is the tonce given in the `PlaceOrder` command that opened the order, or **null** if that command did not specify a tonce.
> * `base` and `counter` are the asset codes of the base and counter assets of the modified order.
> * `quantity` is the new [scaled][] amount of the base asset that remains to be traded in the order after the modification. It is negative for a sell order and positive for a buy order.
> * `price` is the new [scaled][] price at which the modified order offers to trade.
> * `time` is the micro-timestamp at which the order was modified.

> **Error Reply**

```json
{
    "tag": <integer>,
    "error_code": <integer>,
    "error_msg": <string>
}
```

> * `tag` is present iff `tag` was given and non-zero in the request.

Modifies an open limit order. Only the quantity and price may be modified.

**Authorization:** Any authenticated user may invoke this command.

error_code | error_msg
-------------|------------------------------------------------------------------
1            | "The specified order was not found."
4            | "You have insufficient funds."
6            | "You are sending orders too rapidly."
7            | "You are not authenticated."
8            | "You must specify either order ID or tonce."
8            | "Tonce must not be zero."
8            | "You must specify quantity delta and/or price."
8            | "Quantity delta must not be zero."
8            | "Price must not be zero."
8            | *(varies)*
9	     | "Post-only order with these parameters would result in an immediate match."


## CancelOrder

> **Request**

```json
{
    "tag": <integer>,
    "method": "CancelOrder",
    "id": <integer>,
    "tonce": <integer>
}
```

> * `tag` is optional. Iff given and non-zero, it will be echoed in the reply.
> * `id` is the numeric identifier of the order to be canceled. This field must be omitted if `tonce` is supplied.
> * `tonce` is the tonce given in the `PlaceOrder` command that opened the order to be canceled. This field must be supplied if `id` is omitted.

> **Success Reply**

```json
{
    "tag": <integer>,
    "error_code": 0,
    "id": <integer>,
    "tonce": <integer>,
    "base": <integer>,
    "counter": <integer>,
    "quantity": <integer>,
    "price": <integer>,
    "time": <integer>
}
```

> * `tag` is present iff `tag` was given and non-zero in the request.
> * `id` is the numeric identifier of the canceled order.
> * `tonce` is the tonce given in the `PlaceOrder` command that opened the order, or **null** if that command did not specify a tonce.
> * `base` and `counter` are the asset codes of the base and counter assets of the canceled order.
> * `quantity` is the [scaled][] amount of the base asset that was remaining to be traded when the order was canceled. It is negative for a sell order and positive for a buy order.
> * `price` is the [scaled][] price at which the canceled order had offered to trade.
> * `time` is the micro-timestamp at which the order was opened.

> **Error Reply**

```json
{
    "tag": <integer>,
    "error_code": <integer>,
    "error_msg": <string>
}
```

> * `tag` is present iff `tag` was given and non-zero in the request.

Cancels an open order belonging to the authenticated user.

**Authorization:** Any authenticated user may invoke this command.

error_code | error_msg
-------------|------------------------------------------------------------------
1            | "The specified order was not found."
6            | "You are sending orders too rapidly."
7            | "You are not authenticated."
8            | "You must specify either order ID or tonce."
8            | *(varies)*


## CancelAllOrders

> **Request**

```json
{
    "tag": <integer>,
    "method": "CancelAllOrders"
}
```

> * `tag` is optional. Iff given and non-zero, it will be echoed in the reply.

> **Success Reply**

```json
{
    "tag": <integer>,
    "error_code": 0,
    "orders": [
        {
            "id": <integer>,
            "tonce": <integer>,
            "base": <integer>,
            "counter": <integer>,
            "quantity": <integer>,
            "price": <integer>,
            "time": <integer>
        },
        ...
    ]
}
```

> * `tag` is present iff `tag` was given and non-zero in the request.
> * `id` is the numeric identifier of a canceled order.
> * `tonce` is the tonce given in the `PlaceOrder` command that opened the order, or **null** if that command did not specify a tonce.
> * `base` and `counter` are the asset codes of the base and counter assets of the canceled order.
> * `quantity` is the [scaled][] amount of the base asset that was remaining to be traded when the order was canceled. It is negative for a sell order and positive for a buy order.
> * `price` is the [scaled][] price at which the canceled order had offered to trade.
> * `time` is the micro-timestamp at which the order was opened.

> **Error Reply**

```json
{
    "tag": <integer>,
    "error_code": <integer>,
    "error_msg": <string>
}
```

> * `tag` is present iff `tag` was given and non-zero in the request.

Cancels all open orders belonging to the authenticated user.
Also resets the user's tonce counter to zero.

**Authorization:** Any authenticated user may invoke this command.

error_code | error_msg
-------------|------------------------------------------------------------------
6            | "You are sending orders too rapidly."
7            | "You are not authenticated."
8            | *(varies)*


## GetTradeVolume

> **Request**

```json
{
    "tag": <integer>,
    "method": "GetTradeVolume",
    "asset": <integer>
}
```

> * `tag` is optional. Iff given and non-zero, it will be echoed in the reply.
> * `asset` is the asset code of the asset whose trade volume is to be retrieved.

> **Success Reply**

```json
{
    "tag": <integer>,
    "error_code": 0,
    "volume": <integer>
}
```

> * `tag` is present iff `tag` was given and non-zero in the request.
> * `volume` is the user's 30-day trailing [scaled][] trade volume in the specified asset.

> **Error Reply**

```json
{
    "tag": <integer>,
    "error_code": <integer>,
    "error_msg": <string>
}
```

> * `tag` is present iff `tag` was given and non-zero in the request.

Retrieves the 30-day trailing trade volume for the authenticated user.

**Authorization:** Any authenticated user may invoke this command.

error_code | error_msg
-------------|------------------------------------------------------------------
6            | "You are making information requests too rapidly."
7            | "You are not authenticated."
8            | *(varies)*


## WatchOrders

> **Request**

```json
{
    "tag": <integer>,
    "method": "WatchOrders",
    "base": <integer>,
    "counter": <integer>,
    "watch": <boolean>
}
```

> * `tag` is optional. Iff given and non-zero, it will be echoed in the reply.
> * `base` and `counter` are the asset codes of the base and counter assets of the order book whose orders feed the client is to be subscribed to or unsubscribed from.
> * `watch` is a flag indicating whether to subscribe (**true**) or unsubscribe (**false**).

> **Success Reply**

```json
{
    "tag": <integer>,
    "error_code": 0,
    "orders": [
        {
            "id": <integer>,
            "quantity": <integer>,
            "price": <integer>,
            "time": <integer>
        },
        ...
    ]
}
```

> * `tag` is present iff `tag` was given and non-zero in the request.
> * `orders` is present iff `watch` was **true** in the request.
> * `id` is the numeric identifier of an order.
> * `quantity` is the [scaled][] amount of the base asset that is to be traded. It is negative for a sell order and positive for a buy order.
> * `price` is the [scaled][] price at which the order offers to trade.
> * `time` is the micro-timestamp at which the order was opened.

> **Error Reply**

```json
{
    "tag": <integer>,
    "error_code": <integer>,
    "error_msg": <string>
}
```

> * `tag` is present iff `tag` was given and non-zero in the request.

Subscribes/unsubscribes the requesting client to/from the orders feed of a specified order book.
When subscribing, up to 1000 bids and 1000 asks from the top of the book are returned in the response.

**Authorization:** None required.

error_code | error_msg
-------------|------------------------------------------------------------------
1            | "You specified an invalid asset pair."
1            | "You are not watching the order book for the specified asset pair."
2            | "You are already watching the order book for the specified asset pair."
8            | *(varies)*


## WatchTicker

> **Request**

```json
{
    "tag": <integer>,
    "method": "WatchTicker",
    "base": <integer>,
    "counter": <integer>,
    "watch": <boolean>
}
```

> * `tag` is optional. Iff given and non-zero, it will be echoed in the reply.
> * `base` and `counter` are the asset codes of the base and counter assets of the order book whose ticker feed the client is to be subscribed to or unsubscribed from.
> * `watch` is a flag indicating whether to subscribe (**true**) or unsubscribe (**false**).

> **Success Reply**

```json
{
    "tag": <integer>,
    "error_code": 0,
    "last": <integer>,
    "bid": <integer>,
    "ask": <integer>,
    "low": <integer>,
    "high": <integer>,
    "volume": <integer>
}
```

> * `tag` is present iff `tag` was given and non-zero in the request.
> * `last`, `bid`, `ask`, `low`, `high`, and `volume` are present iff `watch` was **true** in the request.
> * `last` is the [scaled][] price at which the last trade in the specified order book executed, or **null** if no such trade has yet executed.
> * `bid` and `ask` are the highest [scaled][] bid price and lowest [scaled][] ask price in the specified order book, respectively, or **null** if there is no such order.
> * `low` and `high` are the lowest and highest [scaled][] prices at which any trade executed in the specified order book in the trailing 24-hour period, or **null** if no such trade executed in that period.
> * `volume` is the [scaled][] quantity of the base asset that has been traded in the specified order book in the trailing 24-hour period.

> **Error Reply**

```json
{
    "tag": <integer>,
    "error_code": <integer>,
    "error_msg": <string>
}
```

> * `tag` is present iff `tag` was given and non-zero in the request.

Subscribes/unsubscribes the requesting client to/from the ticker feed of a specified order book.
When subscribing, the current ticker values of the book are returned in the response.

**Authorization:** None required.

error_code | error_msg
-------------|------------------------------------------------------------------
1            | "You specified an invalid asset pair."
1            | "You are not watching the ticker for the specified asset pair."
2            | "You are already watching the ticker for the specified asset pair."
8            | *(varies)*


## BalanceChanged

> **Notification**

```json
{
    "notice": "BalanceChanged",
    "asset": <integer>,
    "balance": <integer>
}
```

> * `asset` is the asset code of the balance that changed.
> * `balance` is the user's new [scaled][] available balance in the specified asset.

One of the authenticated user's balances has changed.

**Recipients:** Authenticated users receive notice of changes to their own balances.


## OrderOpened

> **Notification**

```json
{
    "notice": "OrderOpened",
    "id": <integer>,
    "tonce": <integer>,
    "base": <integer>,
    "counter": <integer>,
    "quantity": <integer>,
    "price": <integer>,
    "time": <integer>
}
```

> * `id` is the numeric identifier of the order.
> * `tonce` is the tonce given in the `PlaceOrder` command that opened the order, or **null** if that command did not specify a tonce. It is present only if the recipient of the notification is the owner of the order.
> * `base` and `counter` are the asset codes of the base and counter assets of the order.
> * `quantity` is the [scaled][] amount of the base asset that is to be traded. It is negative for a sell order and positive for a buy order.
> * `price` is the [scaled][] price at which the order offers to trade.
> * `time` is the micro-timestamp at which the order was opened.

A new order has been opened.

**Recipients:** Authenticated users receive notice of their own orders. Users that have subscribed to the orders feed of an order book receive notice of all orders in that book.


## OrderModified

> **Notification**

```json
{
    "notice": "OrderModified",
    "id": <integer>,
    "tonce": <integer>,
    "base": <integer>,
    "counter": <integer>,
    "quantity": <integer>,
    "price": <integer>,
    "time": <integer>
}
```

> * `id` is the numeric identifier of the order.
> * `tonce` is the tonce given in the `PlaceOrder` command that opened the order, or **null** if that command did not specify a tonce. It is present only if the recipient of the notification is the owner of the order.
> * `base` and `counter` are the asset codes of the base and counter assets of the order.
> * `quantity` is the new [scaled][] amount of the base asset that is to be traded. It is negative for a sell order and positive for a buy order.
> * `price` is the new [scaled][] price at which the order offers to trade.
> * `time` is the micro-timestamp at which the order was modified.

An order has been modified.

**Recipients:** Authenticated users receive notice of their own orders. Users that have subscribed to the orders feed of an order book receive notice of all orders in that book.


## OrdersMatched

> **Notification**

```json
{
    "notice": "OrdersMatched",
    "bid": <integer>,
    "bid_tonce": <integer>,
    "ask": <integer>,
    "ask_tonce": <integer>,
    "base": <integer>,
    "counter": <integer>,
    "quantity": <integer>,
    "taker_side": <string>,
    "taker": <boolean>,
    "price": <integer>,
    "total": <integer>,
    "bid_rem": <integer>,
    "ask_rem": <integer>,
    "time": <integer>,
    "bid_base_fee": <integer>,
    "bid_counter_fee": <integer>,
    "ask_base_fee": <integer>,
    "ask_counter_fee": <integer>
}
```

> * `bid` and `ask` are the numeric identifiers of the bid and ask orders, respectively, that matched. Either (but not both) may be omitted if the corresponding side of the trade was a market order.
> * `bid_tonce` and `ask_tonce` are the tonces given in the `PlaceOrder` commands that opened the bid and ask orders, respectively, or **null** if the respective command did not specify a tonce. These fields are present only if the recipient of the notification is the buyer or seller, respectively.
> * `base` and `counter` are the asset codes of the base and counter assets of the orders.
> * `quantity` is the [scaled][] amount of the base asset that was traded. It is always positive.
> * `taker_side` is the string identifier showing **bid** or **ask** to indentify which side of the matched trade was the taker/aggressor.
> * `taker` is the boolean identifier showing True for a taker trade or False for maker trade.  This field is currently only visibile for the private authenticated OrdersMatched message.
> * `price` is the [scaled][] price at which the trade executed.
> * `total` is the [scaled][] amount of the counter asset that was traded. It is always positive.
> * `bid_rem` and `ask_rem` are the [scaled][] quantities remaining in the bid and ask orders, respectively, after the trade. Either (but not both) may be omitted if the corresponding side of the trade was a market order.
> * `time` is the micro-timestamp at which the trade executed.
> * `bid_base_fee`, `bid_counter_fee`, `ask_base_fee`, and `ask_counter_fee` are the [scaled][] fees levied on the buyer and seller, respectively, in the base and counter assets, respectively. These fields are present only if the recipient of the notification is the buyer or seller, respectively.

Two orders matched, resulting in a trade.

**Recipients:** Authenticated users receive notice of their own orders. Users that have subscribed to the orders feed of an order book receive notice of all orders in that book.


## OrderClosed

> **Notification**

```json
{
    "notice": "OrderClosed",
    "id": <integer>,
    "tonce": <integer>,
    "base": <integer>,
    "counter": <integer>,
    "quantity": <integer>,
    "price": <integer>,
    "time_closed": <integer>
}
```

> * `id` is the numeric identifier of the order.
> * `tonce` is the tonce given in the `PlaceOrder` command that opened the order, or **null** if that command did not specify a tonce. It is present only if the recipient of the notification is the owner of the order.
> * `base` and `counter` are the asset codes of the base and counter assets of the order.
> * `quantity` is the [scaled][] amount of the base asset that was remaining to be traded when the order was closed. It is negative for a sell order and positive for a buy order. It is zero if the order was completely fulfilled.
> * `price` is the [scaled][] price at which the order had offered to trade.
> * `time_closed` is the microsecond timestamp of when the order was closed.

An order was removed from the order book.

**Recipients:** Authenticated users receive notice of their own orders. Users that have subscribed to the orders feed of an order book receive notice of all orders in that book.


## TickerChanged

> **Notification**

```json
{
    "notice": "TickerChanged",
    "base": <integer>,
    "counter": <integer>,
    "last": <integer>,
    "bid": <integer>,
    "ask": <integer>,
    "low": <integer>,
    "high": <integer>,
    "volume": <integer>
}
```

> * `base` and `counter` are the asset codes of the base and counter assets of the order book whose ticker changed.
> * `last` is the [scaled][] price at which the last trade in the specified order book executed, or **null** if no such trade has yet executed. It is present only if it has changed since the previous notice in which it was present.
> * `bid` and `ask` are the highest [scaled][] bid price and lowest [scaled][] ask price in the specified order book, respectively, or **null** if there is no such order. Each is present only if it has changed since the previous notice in which it was present.
> * `low` and `high` are the lowest and highest [scaled][] prices at which any trade executed in the specified order book in the trailing 24-hour period, or **null** if no such trade executed in that period. Each is present only if it has changed since the previous notice in which it was present.
> * `volume` is the [scaled][] quantity of the base asset that has been traded in the specified order book in the trailing 24-hour period. It is present only if it has changed since the previous notice in which it was present.

The ticker for an order book changed.

**Recipients:** Users that have subscribed to the ticker feed of an order book receive notice of all ticker changes in that book.


## TradeVolumeChanged

> **Notification**

```json
{
    "notice": "TradeVolumeChanged",
    "asset": <integer>,
    "volume": <integer>
}
```

> * `asset` is the asset code of the asset whose 30-day trailing trade volume changed.
> * `volume` is the user's updated 30-day trailing trade volume in the specified asset.

A 30-day trailing trade volume for the authenticated user changed.

**Recipients:** Authenticated users receive notice of changes to their own trade volumes.
