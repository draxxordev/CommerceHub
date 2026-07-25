# CommerceHub

A wrapper around Roblox's Default `MarketplaceService`, designed to simplify gameplay purchase systems while providing a clean, extensible API.

CommerceHub goes beyond simple purchase prompts by including built-in validation, caching, asynchronous workflows, signals, rate limiting, and lifecycle management, making it suitable for both small projects and larger experiences.

## Features

*  Gamepass & Developer Product purchase wrappers
*  Built-in gamepass gifting system
*  Promise-based asynchronous API
*  Signal & callback support
*  Product information and price caching
*  User, asset, and receipt validation
*  Configurable rate limiting
*  Purchase ticket tracking
*  Automatic cleanup and lifecycle management
*  Regional price support
*  Clean, developer-friendly API

## Example

```lua
local CommerceHub = require(path.To.CommerceHub)

local Commerce = CommerceHub.new()
Commerce:Init()

Commerce:PromptGamepassPurchaseAsync(player.UserId, GAMEPASS_ID)
    :andThen(function()
        print("Purchase prompt opened!")
    end)
    :catch(function(err)
        warn(err)
    end)
```

## Why CommerceHub?

Instead of interacting directly with `MarketplaceService` every time, CommerceHub provides a centralized API that handles common concerns automatically, allowing you to focus on building your game rather than rewriting commerce logic.

## Goals

* Reduce repetitive MarketplaceService code.
* Provide a safer and more consistent purchasing API.
* Encourage scalable, maintainable commerce systems.
* Make advanced commerce features accessible through a simple interface.

---

Built with love an passion for other Roblox developers.

By @Draxxor
