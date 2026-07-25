# API Reference

## Constructor

### `CommerceHub.new()`

Creates a new CommerceHub instance.

```lua
local CommerceHub = require(path.To.CommerceHub)

local Commerce = CommerceHub.new()
Commerce:Init()
```

---

# Initialization

### `:Init()`

Initializes CommerceHub and connects internal services.

Automatically:

* Loads pending gifts for joining players.
* Cleans player data on leave.
* Sets up internal listeners.

---

# Product Information

### `:GetProductInfoAsync(assetId)`

Returns cached Marketplace product information.

**Returns:** `Promise<ProductInfo>`

---

### `:GetProductPriceAsync(userId, productId)`

Returns the regional price of a developer product.

**Returns:** `Promise<number>`

---

# Ownership

### `:IsGamepassOwnedAsync(userId, gamepassId)`

Checks whether a player owns a gamepass.

**Returns:** `Promise<boolean>`

---

### `:IsProductOwnedAsync(userId, productId)`

Checks whether a player owns a developer product.

**Returns:** `Promise<boolean>`

---

### `:GetUserGamepassesAsync(userId)`

Returns every gamepass owned by the user.

**Returns:** `Promise<{number}>`

---

# Purchases

### `:PromptGamepassPurchaseAsync(userId, gamepassId)`

Prompts a gamepass purchase.

**Returns:** `Promise`

---

### `:PromptProductPurchaseAsync(userId, productId, expectedPrice?)`

Prompts a developer product purchase.

**Returns:** `Promise`

---

### `:PurchaseWithSignal(userId, productId)`

Prompts a purchase and returns a signal that fires when completed.

**Returns:** `Signal`

---

# Gifting

### `:GiftGamepassAsync(fromUserId, toUserId, gamepassId)`

Queues and stores a gamepass gift.

**Returns:** `Promise<Gift>`

---

### `:GetPendingGiftsAsync(userId)`

Loads all pending gifts for a player.

**Returns:** `Promise<{Gift}>`

---

### `:ClaimGiftAsync(userId, giftIndex)`

Claims and removes a pending gift.

**Returns:** `Promise<Gift>`

---

### `:LoadPlayerGifts(userId)`

Loads pending gifts and fires gift signals.

**Returns:** `Promise`

---

# Receipts

### `:ValidatePurchaseReceipt(receiptId, userId, productId)`

Validates a pending purchase receipt.

**Returns:** `Promise`

---

# Tickets

### `:CreatePurchaseTicket(userId, info)`

Creates a purchase tracking ticket.

**Returns:** `Ticket`

---

### `:GetTicket(jobId)`

Retrieves a ticket by its Job ID.

**Returns:** `Ticket?`

---

# Events

### `:OnPurchaseComplete()`

Returns the global purchase signal.

---

### `:OnGamepassAcquired()`

Returns the gamepass acquisition signal.

---

### `:OnProductPurchased()`

Returns the developer product purchase signal.

---

### `:OnGiftReceived()`

Returns the gift received signal.

---

# Listeners

### `:ListenToPurchases(callback)`

Registers a purchase callback.

Returns a connection object.

---

### `:ListenToGamepasses(callback)`

Registers a gamepass callback.

Returns a connection object.

---

### `:ListenToProductPurchases(callback)`

Registers a developer product callback.

Returns a connection object.

---

### `:ListenToGifts(userId, callback)`

Registers a callback for gifts received by a player.

Returns a connection object.

---

# Callback Helpers

### `:PromptGamepassPurchaseWithCallback(...)`

Prompts a purchase using callbacks instead of promises.

Returns a `Maid`.

---

### `:PromptProductPurchaseWithCallback(...)`

Prompts a product purchase using callbacks instead of promises.

Returns a `Maid`.

---

# Cleanup

### `:Cleanup(userId)`

Removes all temporary player data.

---

### `:ClearCache()`

Clears all internal Marketplace caches.

---

### `:ClearAllListeners()`

Removes every registered listener.

---

### `:Destroy()`

Destroys CommerceHub, disconnects signals, clears caches, listeners, and internal state.
