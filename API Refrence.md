# API Reference

---

## Constructor

### `CommerceHub.new()`

Creates a new CommerceHub instance.

Returns:

```lua
CommerceHub
```

---

## Initialization

### `:Init()`

Initializes CommerceHub.

Automatically:

- Connects Marketplace listeners
- Registers ProcessReceipt
- Loads pending gifts
- Cleans player data
- Registers shutdown cleanup

Returns

```lua
nil
```

---

# Product Information

---

## `:GetProductInfoAsync(assetId)`

Returns Marketplace information.

```lua
Promise<ProductInfo>
```

Features

- Cached
- Auto expires after 5 minutes

---

## `:GetProductPriceAsync(userId, productId)`

Returns the current product price.

```lua
Promise<number>
```

Features

- Cached per user
- Auto expires
- Rate limited

---

# Ownership

---

## `:IsGamepassOwnedAsync(userId, gamepassId)`

Checks Gamepass ownership.

Returns

```lua
Promise<boolean>
```

---

## `:IsProductOwnedAsync(userId, productId)`

Checks whether your game's ProfileStore marks the Developer Product as owned.

Returns

```lua
Promise<boolean>
```

> **Note**
>
> Roblox Developer Products are consumables and Roblox does **not** provide an ownership API for them.
> CommerceHub checks your own ProfileStore data instead.

---

## `:GetUserGamepassesAsync(userId, gamepassIds)`

Checks multiple Gamepasses and returns only the owned ones.

Returns

```lua
Promise<{number}>
```

Example

```lua
Commerce:GetUserGamepassesAsync(
	player.UserId,
	{
		123,
		456,
		789
	}
)
```

---

# Purchasing

---

## `:PromptGamepassPurchaseAsync(userId, gamepassId)`

Shows a Gamepass purchase prompt.

Returns

```lua
Promise
```

Automatically

- validates IDs
- validates player
- checks rate limits

---

## `:PromptProductPurchaseAsync(userId, productId)`

Shows a Developer Product purchase prompt.

Returns

```lua
Promise
```

Automatically

- validates IDs
- validates player
- checks rate limits

---

## `:PurchaseWithSignal(userId, productId)`

Alternative to Promises.

Returns

```lua
Signal
```

The Signal fires:

```lua
true
```

or

```lua
false
```

depending on whether the purchase prompt successfully opened.

---

# Product Handlers

---

## `:RegisterProductHandler(productId, callback)`

Registers the reward callback for a Developer Product.

```lua
Commerce:RegisterProductHandler(
	PRODUCT_ID,
	function(player, receiptInfo)

	end
)
```

The callback is invoked from `MarketplaceService.ProcessReceipt`.

---

# Gifting

---

## `:GiftGamepassAsync(fromUserId, toUserId, gamepassId)`

Creates a pending gift.

Returns

```lua
Promise<Gift>
```

Automatically

- validates users
- prevents gifting yourself
- stores using ProfileStore

---

## `:GetPendingGiftsAsync(userId)`

Returns pending gifts.

```lua
Promise<{Gift}>
```

---

## `:ClaimGiftAsync(userId, giftIndex)`

Claims a gift.

Returns

```lua
Promise<Gift>
```

---

## `:LoadPlayerGifts(userId)`

Loads gifts and fires the GiftReceived signal.

Returns

```lua
Promise
```

---

# Receipts

---

## `:ValidatePurchaseReceipt(receiptId, userId, productId)`

Validates an internally tracked receipt.

Returns

```lua
Promise
```

---

# Purchase Tickets

---

## `:CreatePurchaseTicket(userId, info)`

Creates a temporary purchase ticket.

Returns

```lua
Ticket
```

Example

```lua
{
	Info = ...,
	Owners = {
		userId
	},
	Job = "..."
}
```

Tickets automatically expire after one hour.

---

## `:GetTicket(jobId)`

Returns

```lua
Ticket?
```

---

# Signals

---

## `:OnPurchaseComplete()`

Returns

```lua
Signal
```

Data

```lua
{
	UserId,
	AssetId,
	AssetType,
	Timestamp
}
```

---

## `:OnGamepassAcquired()`

Returns

```lua
Signal
```

Data

```lua
{
	UserId,
	GamepassId,
	Timestamp
}
```

---

## `:OnProductPurchased()`

Returns

```lua
Signal
```

Data

```lua
{
	UserId,
	ProductId,
	ReceiptId,
	Timestamp
}
```

---

## `:OnGiftReceived()`

Returns

```lua
Signal
```

Data

```lua
{
	ToUserId,
	FromUserId,
	GamepassId,
	Timestamp
}
```

---

# Listener API

---

## `:ListenToPurchases(callback)`

Listens for all purchase events.

Returns

```lua
Connection
```

---

## `:ListenToGamepasses(callback)`

Returns

```lua
Connection
```

---

## `:ListenToProductPurchases(callback)`

Returns

```lua
Connection
```

---

## `:ListenToGifts(userId, callback)`

Returns

```lua
Connection
```

---

# Callback Helpers

---

## `:PromptGamepassPurchaseWithCallback(...)`

Callback-based alternative.

```lua
callback(
	success,
	userId,
	gamepassId,
	error
)
```

Returns

```lua
Maid
```

---

## `:PromptProductPurchaseWithCallback(...)`

Callback-based alternative.

Returns

```lua
Promise
```

Callback

```lua
callback(
	success,
	userId,
	productId,
	error
)
```

---

# Cleanup

---

## `:Cleanup(userId)`

Removes

- Purchase tickets
- Rate limits

---

## `:ClearCache()`

Clears

- Product cache
- Price cache

---

## `:ClearAllListeners()`

Removes all callback listeners.

Does **not** disconnect Signal connections.

---

## `:Destroy()`

Destroys CommerceHub.

Automatically

- clears caches
- clears tickets
- clears listeners
- clears pending data
- destroys signals

---

# Notes

## Promises

Functions ending in `Async` return Promises.

Calling them **does not yield** your thread.

Instead, the asynchronous work happens internally.

Example

```lua
Commerce:GetProductInfoAsync(id)
	:andThen(function(info)

	end)
```

---

## Developer Products

Developer Products are consumables.

CommerceHub stores ownership information using your own ProfileStore rather than relying on Roblox ownership APIs.

---

## Rate Limiting

CommerceHub automatically rate limits purchase-related requests to help prevent accidental spam and abuse.

---
