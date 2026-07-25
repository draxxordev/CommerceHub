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
- Creates a new CommerceHub controller instance.
- Internal caches, signals, and tracking systems are shared across instances.

---

# Initialization

## `:Init()`

Initializes CommerceHub and connects all Marketplace-related systems.

### Parameters

None.

### Returns

`nil`

### Automatically

- Connects Gamepass purchase listeners.
- Registers Developer Product receipt processing.
- Loads pending gifts when players join.
- Cleans player data when players leave.
- Registers game shutdown cleanup.

### Best Used For

Call once after creating the CommerceHub instance.

### Example

```lua
local Commerce = CommerceHub.new()

Commerce:Init()
```

---

# Product Information

## `:GetProductInfoAsync(assetId)`

Retrieves Marketplace information for an asset.

If information has already been requested, CommerceHub returns the cached result.

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `assetId` | `number` | Marketplace Asset ID. |

### Returns

`Promise<ProductInfo>`

### Best Used For

- Shop interfaces
- Product descriptions
- Asset metadata
- Product thumbnails

### Notes

- Automatically caches results.
- Cache expires after 5 minutes.
- Rejects when the asset cannot be fetched.

---

## `:GetProductPriceAsync(userId, productId)`

Retrieves the price of a Developer Product.

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `userId` | `number` | User requesting the price. |
| `productId` | `number` | Developer Product ID. |

### Returns

`Promise<number>`

### Best Used For

- Purchase buttons
- Shop displays
- Dynamic pricing UI

### Notes

- Uses MarketplaceService product pricing.
- Caches prices per user and product combination.
- Cache expires after 5 minutes.

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

- VIP systems
- Locked areas
- Cosmetic unlocks
- Permission checks

### Notes

- Uses Roblox's Gamepass ownership API.
- Includes rate limiting.

---

## `:IsProductOwnedAsync(userId, productId)`

Checks whether a player owns a Developer Product asset.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |
| `productId` | `number` |

### Returns

`Promise<boolean>`

### Best Used For

- Ownership validation
- Developer Product checks

### Notes

- Uses Roblox asset ownership lookup.
- Includes rate limiting.

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

### Notes

- Returns a table containing Gamepass IDs.
- Uses Roblox asset pagination internally.
- Includes rate limiting.

---

# Purchases

## `:PromptGamepassPurchaseAsync(userId, gamepassId)`

Prompts a Gamepass purchase for a player.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |
| `gamepassId` | `number` |

### Returns

`Promise`

### Automatically

- Validates User ID.
- Validates Gamepass ID.
- Checks rate limits.
- Validates player existence.
- Creates purchase prompt.

### Best Used For

- Shop buttons
- Premium upgrades
- Gamepass unlocks

### Example

```lua
Commerce:PromptGamepassPurchaseAsync(
	player.UserId,
	GAMEPASS_ID
)
:andThen(function()
	print("Prompt opened!")
end)
```

---

## `:PromptProductPurchaseAsync(userId, productId)`

Prompts a Developer Product purchase.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |
| `productId` | `number` |

### Returns

`Promise`

### Automatically

- Validates User ID.
- Validates Product ID.
- Checks rate limits.
- Validates player existence.
- Creates purchase prompt.

### Best Used For

- Currency purchases
- Consumable items
- Developer Products

### Notes

- Developer Product rewards must be handled using `:RegisterProductHandler()`.

---

## `:PurchaseWithSignal(userId, productId)`

Prompts a Developer Product purchase and returns a Signal.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |
| `productId` | `number` |

### Returns

`Signal`

### Fires When

The purchase prompt succeeds or fails.

### Best Used For

Developers who prefer Signals over Promises.

### Example

```lua
local PurchaseSignal = Commerce:PurchaseWithSignal(
	player.UserId,
	PRODUCT_ID
)

PurchaseSignal:Connect(function(success)
	if success then
		print("Purchase started!")
	end
end)
```

---

# Developer Product Handlers

## `:RegisterProductHandler(productId, callback)`

Registers a function that grants a Developer Product reward.

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `productId` | `number` | Developer Product ID. |
| `callback` | `function` | Function that grants the reward. |

### Returns

`nil`

### Callback Parameters

```lua
callback(player, receiptInfo)
```

| Name | Type |
|------|------|
| `player` | `Player` |
| `receiptInfo` | `table` |

### Best Used For

- Currency rewards
- Items
- Boosts
- Consumables

### Example

```lua
Commerce:RegisterProductHandler(
	PRODUCT_ID,
	function(player, receiptInfo)

		print(
			player.Name,
			"bought",
			receiptInfo.ProductId
		)

	end
)
```

---

# Gifting

## `:GiftGamepassAsync(fromUserId, toUserId, gamepassId)`

Creates a Gamepass gift for another player.

### Parameters

| Name | Type |
|------|------|
| `fromUserId` | `number` |
| `toUserId` | `number` |
| `gamepassId` | `number` |

### Returns

`Promise<Gift>`

### Automatically

- Validates users.
- Prevents gifting to yourself.
- Stores gift data using ProfileStore.
- Creates pending gift data.

### Best Used For

- Friend rewards
- Events
- Gift shops

---

## `:GetPendingGiftsAsync(userId)`

Retrieves pending gifts from storage.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |

### Returns

`Promise<{Gift}>`

### Best Used For

- Loading player gifts
- Gift menus
- Reward systems

---

## `:ClaimGiftAsync(userId, giftIndex)`

Claims and removes a stored gift.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |
| `giftIndex` | `number` |

### Returns

`Promise<Gift>`

### Automatically

- Loads gift storage.
- Removes claimed gift.
- Saves updated data.

---

## `:LoadPlayerGifts(userId)`

Loads and fires pending gifts for a player.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |

### Returns

`Promise`

### Automatically

- Loads stored gifts.
- Fires `OnGiftReceived()`.
- Updates gift state.

---

# Receipts

## `:ValidatePurchaseReceipt(receiptId, userId, productId)`

Validates a stored Developer Product receipt.

### Parameters

| Name | Type |
|------|------|
| `receiptId` | `string` |
| `userId` | `number` |
| `productId` | `number` |

### Returns

`Promise`

### Best Used For

Internal receipt validation.

### Notes

- Checks receipt ownership.
- Checks product ID.
- Marks purchase as validated.

---

# Purchase Tickets

## `:CreatePurchaseTicket(userId, info)`

Creates a temporary purchase tracking ticket.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |
| `info` | `any` |

### Returns

`Ticket`

### Ticket Format

```lua
{
	Info = any,
	Owners = {
		userId
	},
	Job = string
}
```

### Automatically

- Generates unique ticket ID.
- Tracks ownership.
- Cleans after 1 hour.

### Best Used For

- Complex purchase flows.
- Temporary transaction tracking.

---

## `:GetTicket(jobId)`

Retrieves a purchase ticket.

### Parameters

| Name | Type |
|------|------|
| `jobId` | `string` |

### Returns

`Ticket?`

### Notes

Returns `nil` if the ticket does not exist.

---

# Events

## `:OnPurchaseComplete()`

Returns the global purchase completion Signal.

### Returns

`Signal`

### Fires When

Any completed purchase event is processed.

### Signal Data

```lua
{
	UserId = number,
	AssetId = number,
	AssetType = string,
	Timestamp = number
}
```

---

## `:OnGamepassAcquired()`

Returns the Gamepass acquisition Signal.

### Returns

`Signal`

### Fires When

A player successfully purchases a Gamepass.

### Signal Data

```lua
{
	UserId = number,
	GamepassId = number,
	Timestamp = number
}
```

---

## `:OnProductPurchased()`

Returns the Developer Product purchase Signal.

### Returns

`Signal`

### Fires When

A Developer Product receipt is successfully processed.

### Signal Data

```lua
{
	UserId = number,
	ProductId = number,
	ReceiptId = string,
	Timestamp = number
}
```

---

## `:OnGiftReceived()`

Returns the gift received Signal.

### Returns

`Signal`

### Fires When

A player loads pending gifts.

### Signal Data

```lua
{
	ToUserId = number,
	FromUserId = number,
	GamepassId = number,
	Timestamp = number
}
```

---

# Listeners

## `:ListenToPurchases(callback)`

Registers a listener for all completed purchases.

### Parameters

| Name | Type |
|------|------|
| `callback` | `function` |

### Returns

`Connection`

### Callback Data

```lua
callback(data)
```

### Best Used For

- Global purchase tracking
- Analytics
- Rewards

---

## `:ListenToGamepasses(callback)`

Registers a listener for Gamepass purchases.

### Parameters

| Name | Type |
|------|------|
| `callback` | `function` |

### Returns

`Connection`

### Callback Data

```lua
callback(data)
```

---

## `:ListenToProductPurchases(callback)`

Registers a listener for Developer Product purchases.

### Parameters

| Name | Type |
|------|------|
| `callback` | `function` |

### Returns

`Connection`

### Callback Data

```lua
callback(data)
```

---

## `:ListenToGifts(userId, callback)`

Registers a listener for gifts received by a player.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |
| `callback` | `function` |

### Returns

`Connection`

### Example

```lua
Commerce:ListenToGifts(
	player.UserId,
	function(gift)

		print(
			"Received gift:",
			gift.GamepassId
		)

	end
)
```

---

# Callback Helpers

## `:PromptGamepassPurchaseWithCallback(userId, gamepassId, callback)`

Callback-based alternative to `PromptGamepassPurchaseAsync()`.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |
| `gamepassId` | `number` |
| `callback` | `function` |

### Returns

`Maid`

### Callback Format

```lua
callback(
	success,
	userId,
	gamepassId,
	error?
)
```

### Best Used For

Projects that prefer callbacks over Promise chaining.

---

## `:PromptProductPurchaseWithCallback(userId, productId, callback)`

Callback-based alternative to `PromptProductPurchaseAsync()`.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |
| `productId` | `number` |
| `callback` | `function` |

### Returns

`Promise`

### Callback Format

```lua
callback(
	success,
	userId,
	productId,
	error?
)
```

---

# Listener Removal

## `:UnlistenToPurchases(callback)`

Removes a purchase listener.

### Parameters

| Name | Type |
|------|------|
| `callback` | `function` |

### Returns

`nil`

---

## `:UnlistenToGamepasses(callback)`

Removes a Gamepass listener.

### Parameters

| Name | Type |
|------|------|
| `callback` | `function` |

### Returns

`nil`

---

## `:UnlistenToProductPurchases(callback)`

Removes a Developer Product listener.

### Parameters

| Name | Type |
|------|------|
| `callback` | `function` |

### Returns

`nil`

---

# Cleanup

## `:Cleanup(userId)`

Removes temporary player-specific CommerceHub data.

### Parameters

| Name | Type |
|------|------|
| `userId` | `number` |

### Automatically Removes

- Purchase tickets
- Rate limit data
- Temporary player state

### Best Used For

Player removal cleanup.

---

## `:ClearCache()`

Clears all Marketplace caches.

### Automatically Removes

- Product information cache
- Price cache

### Best Used For

- Debugging
- Development
- Forced refreshes

---

## `:ClearAllListeners()`

Removes all registered callback listeners.

### Automatically Removes

- Purchase listeners
- Gamepass listeners
- Product listeners

### Notes

Use carefully. Existing Signal connections are not affected.

---

## `:Destroy()`

Destroys the CommerceHub instance.

### Automatically

- Clears internal caches.
- Removes tracked tickets.
- Clears listeners.
- Clears pending data.
- Destroys Signals.

### Notes

Only call when CommerceHub is no longer required.

---
