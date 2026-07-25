# CommerceHub

A wrapper around Roblox's Default `MarketplaceService`, designed to simplify gameplay purchase systems while providing a clean, extensible API.

CommerceHub goes past simple purchase prompts by including built-in validation, caching, asynchronous workflows, signals, rate limiting, and lifecycle management, making it suitable for both small projects and larger games.

## Features

- Gamepass purchasing
- Developer Product purchasing
- Receipt processing
- Product handlers
- Gamepass gifting
- Promise workflows
- Signal workflows
- Callback workflows
- Marketplace caching
- Rate limiting
- Purchase tracking
- ProfileStore persistence
- Automatic cleanup

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

Instead of interacting directly with `MarketplaceService` every time, CommerceHub provides a centralized API that handles common concerns automatically, allowing you to focus on building your game rather than rewriting gamepass logic.

## Goals

* Reduce repetitive MarketplaceService code.
* Provide a safer and more consistent purchasing API.
* Encourage scalable, maintainable commerce systems.
* Make advanced commerce features accessible through a simple interface.

---

# License

MIT

---

# Built with love and passion for other Roblox developers.
❤️

By @Draxxor
