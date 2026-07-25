# Central Syntax

Below are the most common usage patterns you'll use when working with CommerceHub.

---

## Creating a CommerceHub Instance

```lua
local CommerceHub = require(path.To.CommerceHub)

local Commerce = CommerceHub.new()
Commerce:Init()
```

---

## Awaiting a Promise

```lua
Commerce:GetProductPriceAsync(player.UserId, PRODUCT_ID)
    :andThen(function(price)
        print(price)
    end)
    :catch(function(err)
        warn(err)
    end)
```

---

## Prompting a Purchase

```lua
Commerce:PromptGamepassPurchaseAsync(
    player.UserId,
    GAMEPASS_ID
)
```

---

## Using Signals

```lua
Commerce:OnPurchaseComplete():Connect(function(player, productId)
    print(player.Name, productId)
end)
```

---

## Registering Listeners

```lua
local connection = Commerce:ListenToPurchases(function(player, productId)
    print(player.Name, "purchased", productId)
end)

connection:Disconnect()
```

---

## Callback-Based API

```lua
local maid = Commerce:PromptProductPurchaseWithCallback(
    player.UserId,
    PRODUCT_ID,
    function(success)
        if success then
            print("Purchase completed!")
        end
    end
)
```

---

## Gifting

```lua
Commerce:GiftGamepassAsync(
    sender.UserId,
    receiver.UserId,
    GAMEPASS_ID
)
```

---

## Loading Pending Gifts

```lua
Commerce:GetPendingGiftsAsync(player.UserId)
    :andThen(function(gifts)
        for _, gift in ipairs(gifts) do
            print(gift)
        end
    end)
```

---

## Cleaning Up

```lua
Commerce:Cleanup(player.UserId)
```

---

## Destroying CommerceHub

```lua
Commerce:Destroy()
```

---

# Common Return Types

| Return Type | Description |
|-------------|-------------|
| `Promise<T>` | An asynchronous operation that resolves with a value of type `T`. |
| `Signal` | A custom event that can be connected to using `:Connect()`. |
| `Connection` | Returned by listener methods and can be disconnected using `:Disconnect()`. |
| `Ticket` | A CommerceHub purchase tracking object. |
| `Gift` | A stored gift object containing metadata about a gifted purchase. |

---

# Parameter Conventions

CommerceHub uses the following conventions throughout its API.

| Type | Description |
|------|-------------|
| `userId` | Roblox UserId (`number`) of a player. |
| `gamepassId` | Roblox Gamepass asset ID. |
| `productId` | Roblox Developer Product ID. |
| `assetId` | Marketplace asset identifier. |
| `callback` | Function executed when an operation completes. |
| `expectedPrice` | Optional price validation used before prompting a purchase. |

---

# Promise Pattern

Nearly every asynchronous method follows the same pattern.

```lua
Commerce:SomeAsyncMethod(...)
    :andThen(function(result)
        -- Success
    end)
    :catch(function(err)
        -- Failure
    end)
```

---

# Naming Convention

CommerceHub follows a consistent naming scheme.

| Suffix | Meaning |
|---------|---------|
| `Async` | Returns a Promise. |
| `WithCallback` | Uses callback functions instead of Promises. |
| `On...` | Returns a Signal. |
| `ListenTo...` | Registers a listener and returns a Connection. |
| `Get...` | Retrieves cached or live Marketplace information. |
| `Prompt...` | Opens a Roblox purchase prompt. |
| `Create...` | Creates a new CommerceHub object or record. |
| `Validate...` | Verifies data before processing. |
| `Cleanup` | Removes temporary player-specific data. |
| `Destroy` | Completely shuts down the CommerceHub instance. |
