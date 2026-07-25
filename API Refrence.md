# API Reference

---

# Constructor

## `CommerceHub.new()`

Creates a new CommerceHub instance.

### Syntax

```lua
local CommerceHub = require(path.To.CommerceHub)

local Commerce = CommerceHub.new()
Commerce:Init()
```

### Returns

`CommerceHub`

### Notes

- Must be initialized using `:Init()`.
- Creates all internal caches, listeners, signals, and state containers.

---

# Initialization

## `:Init()`

Initializes CommerceHub and connects all Marketplace-related services.

### Parameters

None.

### Returns

`nil`

### Automatically

- Connects MarketplaceService events.
- Loads pending gifts for joining players.
- Cleans temporary player data when players leave.
- Starts internal purchase tracking.
- Initializes receipt processing.

### Best Used For

Call **once** after creating the CommerceHub instance.

### Example

```lua
local Commerce = CommerceHub.new()
Commerce:Init()
```

---

# Product Information

## `:GetProductInfoAsync(assetId)`

Retrieves Marketplace information for an asset.

If the asset has already been requested, CommerceHub returns the cached result instead of requesting it again.

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `assetId` | `number` | Marketplace Asset ID. |

### Returns

`Promise<ProductInfo>`

### Best Used For

- Shop UI
- Item descriptions
- Icons
- Product metadata

### Notes

- Automatically caches results.
- Rejects if the asset does not exist.

---

## `:GetProductPriceAsync(userId, productId)`

Returns the localized price of a Developer Product.

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `userId` | `number` | User requesting the price. |
| `productId` | `number` | Developer Product ID. |

### Returns

`Promise<number>`

### Best Used For

- Purchase buttons
- Dynamic shop interfaces
- Regional pricing

### Notes

- Uses Roblox localized pricing.
- Cached whenever possible.

---

# Ownership

## `:IsGamepassOwnedAsync(userId, gamepassId)`

Checks whether a player owns a Gamepass.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |
| `gamepassId` | `number` |

### Returns

`Promise<boolean>`

### Best Used For

- Locked content
- VIP areas
- Cosmetic unlocks

---

## `:IsProductOwnedAsync(userId, productId)`

Checks whether a player owns a specific product.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |
| `productId` | `number` |

### Returns

`Promise<boolean>`

### Best Used For

- Purchase validation
- Unlock systems

---

## `:GetUserGamepassesAsync(userId)`

Returns every Gamepass owned by a player.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |

### Returns

`Promise<{number}>`

### Best Used For

- Player loading
- Permission systems
- Bulk ownership checks

---

# Purchases

## `:PromptGamepassPurchaseAsync(userId, gamepassId)`

Prompts a Gamepass purchase.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |
| `gamepassId` | `number` |

### Returns

`Promise`

### Automatically

- Validates player
- Validates Gamepass
- Creates purchase ticket
- Applies rate limiting
- Tracks completion
- Fires purchase signals

### Best Used For

- Shop buttons
- Upgrade menus
- Premium features

---

## `:PromptProductPurchaseAsync(userId, productId, expectedPrice?)`

Prompts a Developer Product purchase.

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `userId` | `number` | Purchasing player. |
| `productId` | `number` | Developer Product ID. |

### Returns

`Promise`

### Notes

- Supports optional price verification.
- Automatically validates receipts.

---

## `:PurchaseWithSignal(userId, productId)`

Starts a purchase and returns a Signal that fires once completed.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |
| `productId` | `number` |

### Returns

`Signal`

### Best Used For

Developers who prefer Signals over Promises.

---

# Gifting

## `:GiftGamepassAsync(fromUserId, toUserId, gamepassId)`

Creates and stores a Gamepass gift.

### Parameters

| Name | Type |
|------|------|
| `fromUserId` | `number` |
| `toUserId` | `number` |
| `gamepassId` | `number` |

### Returns

`Promise<Gift>`

### Best Used For

- Holiday events
- Friend gifting
- Reward systems

---

## `:GetPendingGiftsAsync(userId)`

Loads all pending gifts for a player.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |

### Returns

`Promise<{Gift}>`

---

## `:ClaimGiftAsync(userId, giftIndex)`

Claims a pending gift.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |
| `giftIndex` | `number` |

### Returns

`Promise<Gift>`

### Notes

Removes the gift after a successful claim.

---

## `:LoadPlayerGifts(userId)`

Loads and processes all pending gifts for a player.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |

### Returns

`Promise`

### Automatically

- Loads gifts
- Fires gift signals
- Updates internal cache

---

# Receipts

## `:ValidatePurchaseReceipt(receiptId, userId, productId)`

Validates a pending Developer Product receipt.

### Parameters

| Name | Type |
|------|------|
| `receiptId` | `string` |
| `userId` | `number` |
| `productId` | `number` |

### Returns

`Promise`

### Best Used For

Internal receipt verification.

---

# Tickets

## `:CreatePurchaseTicket(userId, info)`

Creates an internal purchase ticket.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |
| `info` | `table` |

### Returns

`Ticket`

### Notes

Used internally for purchase tracking.

---

## `:GetTicket(jobId)`

Retrieves a purchase ticket.

### Parameters

| Name | Type |
|------|------|
| `jobId` | `string` |

### Returns

`Ticket?`

---

# Events

## `:OnPurchaseComplete()`

Returns the global purchase Signal.

### Returns

`Signal`

### Fires When

Any purchase successfully completes.

---

## `:OnGamepassAcquired()`

Returns the Gamepass acquisition Signal.

### Returns

`Signal`

---

## `:OnProductPurchased()`

Returns the Developer Product Signal.

### Returns

`Signal`

---

## `:OnGiftReceived()`

Returns the gift received Signal.

### Returns

`Signal`

---

# Listeners

## `:ListenToPurchases(callback)`

Registers a purchase listener.

### Parameters

| Name | Type |
|------|------|
| `callback` | `function` |

### Returns

`Connection`

---

## `:ListenToGamepasses(callback)`

Registers a Gamepass listener.

### Parameters

| Name | Type |
|------|------|
| `callback` | `function` |

### Returns

`Connection`

---

## `:ListenToProductPurchases(callback)`

Registers a Developer Product listener.

### Parameters

| Name | Type |
|------|------|
| `callback` | `function` |

### Returns

`Connection`

---

## `:ListenToGifts(userId, callback)`

Registers a listener for gifts sent to a player.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |
| `callback` | `function` |

### Returns

`Connection`

---

# Callback Helpers

## `:PromptGamepassPurchaseWithCallback(...)`

Callback alternative to `PromptGamepassPurchaseAsync()`.

### Returns

`Maid`

### Best Used For

Projects that use callbacks instead of Promises.

---

## `:PromptProductPurchaseWithCallback(...)`

Callback alternative to `PromptProductPurchaseAsync()`.

### Returns

`Maid`

---

# Cleanup

## `:Cleanup(userId)`

Removes all temporary player-specific data.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |

### Automatically Removes

- Purchase tickets
- Cached player data
- Temporary listeners
- Rate limit data

---

## `:ClearCache()`

Clears all Marketplace caches.

### Best Used For

- Debugging
- Forced refreshes
- Development

---

## `:ClearAllListeners()`

Disconnects every registered CommerceHub listener.

### Notes

Primarily intended for cleanup or testing.

---

## `:Destroy()`

Completely shuts down the CommerceHub instance.

### Automatically

- Disconnects Marketplace events
- Destroys Signals
- Clears caches
- Removes listeners
- Cleans internal state

### Notes

This should only be called when CommerceHub is no longer needed.
